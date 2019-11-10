# DolphinDB 内存管理

DolphinDB提供多用户并发操作，为了最大化性能，自己管理内存，DolphinDB的内存管理包括以下特点：
* 用户无须关注内存，为用户提供局部变量提供内存管理
* 提供分布式数据库的缓存，最大化性能
* 为发布节点提供流数据队列，以缓存发布消息
* 为订阅节点提供流数据队列，以缓存订阅消息

## 1. 概述

DolphinDB server提供多用户同时并发访问，DolphinDB提供了session机制，来为每个用户创建一个编程环境，比如user1 创建表t，user2也创建同名表t，显然两个用户的表t虽然同名，确是完全独立的，底层在内存分上也是独立的。
DolphinDB提供了函数 getSessionMemoryStat()，来查看每个用户所占用的内存大小。也提供了全局函数mem(true) 来查看server上的整体内存使用情况。
```
getSessionMemoryStat()
mem(true)
```
后续使用这两个函数查看内存变化。

## 2. 用户私有变量的内存管理
### 2.1 内存申请
如下用户创建vector变量，1亿个int类型，约400兆。
```
v = 1..100000000
```
通过函数mem查看server端的内存占用情况
```
sum(mem().blockSize - mem().freeSize) 
```
结果为: 402,865,056，符合预期

再新建一个session（打开另一个GUI），创建同样有1亿长度的vector v，查看内存结果为：805,525,536。
说明虽然系统有两个同名变量，但是属于不同的session，因此内存是完全独立的。

### 2.1 内存释放
用户可通过undef函数，释放变量的内存，如下
```
undef(`v)
```
或者注销掉session(关闭GUI、web或则API客户端），当session所在的连接中断时，所在的内存资源也会被回收。

例如，通过web的notebook连接到server，并且创建了变量，当web关闭时，这些变量占用的内存会被DolphinDB server自动回收掉。


## 3 分布式表的内存管理

第二节提到的变量都是session的私有变量，其他session是看不到的，而且也只能在一台机器上，不能是分布式的。

### 3.1  特点
DolphinDB支持多用户并发查询海量数据，要查询数据必须先将数据加载到内存，而往往内存远小于数据量。如果多个用户同时请求相同的数据，如果内存里面有该数据对的多个副本，显然对内存是极大的浪费。为了应对这种场景，DolphinDB对分布式表的内存管理有以下特点：
* 对于分布式表，数据在哪个节点上，则占用哪个节点的内存，与执行查询任务的节点无关
* 对同一数据，内存只保留一个副本，提供给多用户访问
* 内存允许的情况下，对之前查询过的数据尽量多缓存
下面对这两个特点进行详细的解释：

DolphinDB是列式存储并且分布式表都是分区的，比如按天分区2019.01.01..2019.01.10，每天是1个分区。分布式表内存管理的单位是一个分区的一个列，比如用户查询如下：
```
select id from t where date = 2019.01.01
```
则会加载2019.01.01分区的id列中所有数据，到内存表中。

再比如
```
select * from t where date = 2019.01.01
```
则会加载2019.01.01分区的所有列中所有数据，到内存表中。
上面第一点说明，如果user1 请求了2019.01.01分区的数据，如果另一个用户请求同一个分区的数据的话，则内存里面只有一份，而不是两份。  
第二点说明，用户请求分区数据任务完成后，该内存中的数据并不会被DolphinDB立即释放，而是缓存在内存中，直到系统内存触发上限（由maxMemSize设置）。  

注意，如果上面语句写成”t=select * from t where date = 2019.01.01“，则server端会创建t变量指向该内存，并且t绝不会自动回收，直到用户手动释放t，或者session关闭时被动回收。




### 3.2 示例

为了方便的查看各个节点的内存使用，我们搭建的集群采用2个节点，单副本模式，一层分区，按天。30天，每个分区1000万行，10列（另外9列为LONG类型），所以每个分区共1000 万行 * 80字节/行 = 800M。整个表共3亿行，大小为24GB。




在配置集群时请核对授权文件规定的节点个数以及每个节点最多内核。如果配置超出授权文件规定，集群将无法正常启动。这些异常信息都会记录在log文件中。

进入server子目录。以Linux系统为例，在server目录下创建config, data及log子目录。 这些子目录是为了方便用户理解本教程，但是不是必须的。

```sh
cd /DolphinDB/server/

mkdir /DolphinDB/server/config
mkdir /DolphinDB/server/data
mkdir /DolphinDB/server/log
```

#### 3.1.1 配置控制节点的参数文件

在config目录下，创建controller.cfg文件，可填写以下集群管理的常用参数。用户可根据实际需要调整参数。controller.cfg文件中只有localSite参数是必需的，其它参数都是可选参数。

```txt
localSite=192.168.1.103:8920:ctl8920
localExecutors=3
maxConnections=128
maxMemSize=16
webWorkerNum=4
workerNum=4
dfsReplicationFactor=1
dfsReplicaReliabilityLevel=0
```

以下是对这些参数的解释：

| 参数配置        | 解释          |
|:------------- |:-------------|
|localSite=192.168.1.103:8920:ctl8920 |节点局域网信息，格式为 IP地址:端口号:节点别名，**IP地址为内网IP**，所有字段均为必选项。|
|localExecutors=3  |                 本地执行者的数量。默认值是CPU的内核数量 - 1。|
|maxConnections=128         |        最大向内连接数|
|maxMemSize=16    |                 最大内存（GB）|
|webWorkerNum=4  |                   处理http请求的工作池的大小。默认值是1。|
|workerNum=4          |              常规交互式作业的工作池大小。默认值是CPU的内核数量。|
|dfsReplicationFactor=1     |        每个表分区或文件块的副本数量。默认值是2。|
|dfsReplicaReliabilityLevel=0 |      多个副本是否可以保存在同一台物理服务器上。 0：可以; 1：不可以。默认值是0。|

#### 3.1.2 配置集群成员参数文件

在config目录下，创建cluster.nodes文件，可填写如下内容。用户可根据实际需要调整参数。cluster.nodes用于存放集群代理节点和数据节点信息。本教程使用4个数据节点，用户可更改节点个数。该配置文件分为两列，第一例存放节点IP地址，端口号，和节点别名。这三个信息由冒号：分隔。第二列是说明节点类型。比如代理节点类型为agent, 而数据节点类型为datanode。节点别名是大小写敏感的，而且在集群内必须是唯一的。

```txt
localSite,mode
192.168.1.103:8910:agent,agent
192.168.1.103:8921:DFS_NODE1,datanode
192.168.1.103:8922:DFS_NODE2,datanode
192.168.1.103:8923:DFS_NODE3,datanode
192.168.1.103:8924:DFS_NODE4,datanode
```

#### 3.1.3 配置数据节点参数文件

在config目录下，创建cluster.cfg文件，可填写如下内容。用户可根据实际需要调整参数。cluster.cfg的配置适用于集群中所有数据节点。

```txt
maxConnections=128
maxMemSize=32
workerNum=8
localExecutors=7
webWorkerNum=2
```

### 3.2 配置代理节点参数文件

在config目录下，创建agent.cfg文件，可填写如下常用参数。用户可根据实际需要调整参数。只有LocalSite和controllerSite是必需参数。其它参数均为可选参数。

```txt
workerNum=3
localExecutors=2
maxMemSize=4
localSite=192.168.1.103:8910:agent
controllerSite=192.168.1.103:8920:ctl8920
```

在controller.cfg中的参数localSite应当与所有代理节点的配置文件agent.cfg中的参数controllerSite一致, 因为代理节点使用agent.cfg中的参数controllerSite来寻找controller。若controller.cfg中的参数localSite有变化，即使只是node alias有改变，所有代理节点的配置文件agent.cfg中的参数controllerSite都应当做相应的改变。

### 3.3. DolphinDB集群启动

#### 3.3.1 启动代理节点

在可执行文件所在目录(server目录)运行以下命令行。请注意agent.log存放在log子目录下。如果出现agent无法正常启动的情况，可以查看此log file来诊断错误原因。

##### Linux后台模式启动

```sh
nohup ./dolphindb -console 0 -mode agent -home data -config config/agent.cfg -logFile log/agent.log &
```

建议通过Linux命令`nohup`（头） 和 `&`（尾）启动为后台运行模式，这样即使终端失去连接，DolphinDB也会持续运行。 

“-console”默认是为 1，如果要设置为后台运行，必须要设置为0（"-console 0")，否则系统运行一段时间后会自动退出。

“-mode”表示节点性质，“-home”指定数据以及元数据存储路径，“-config”指定配置文件路径，“-logFile”指定log文件路径。

##### Linux启动前端交互模式

```sh
./dolphindb -mode agent -home data -config config/agent.cfg -logFile log/agent.log
```

##### Windows

```sh
dolphindb.exe -mode agent -home data -config config/agent.cfg -logFile log/agent.log
```

#### 3.3.2 启动控制节点

在可执行文件所在目录(server目录)运行以下命令行。请注意controller.log存放在log子目录下，如果出现agent无法正常启动的情况，可以查看此log文件来诊断错误原因。

##### Linux启动为后台模式

```sh
nohup ./dolphindb -console 0 -mode controller -home data -config config/controller.cfg -clusterConfig config/cluster.cfg -logFile log/controller.log -nodesFile config/cluster.nodes &
```

##### Linux启动前端交互模式

```sh
./dolphindb -mode controller -home data -config config/controller.cfg -clusterConfig config/cluster.cfg -logFile log/controller.log -nodesFile config/cluster.nodes
```

##### Windows

```sh
dolphindb.exe -mode controller -home data -config config/controller.cfg -clusterConfig config/cluster.cfg -logFile log/controller.log -nodesFile config/cluster.nodes
```

#### 3.3.3 如何关闭代理节点和控制节点

如果是启动为前端交互模式，可以在控制台中输入"quit"退出。

```sh
quit
```

如果是启动为后台交互模式，需要用Linux系统`kill`命令。假设运行命令的Linux系统用户名为 "ec2-user"。

```sh
ps aux | grep dolphindb  | grep -v grep | grep ec2-user|  awk '{print $2}' | xargs kill -TERM
```

#### 3.3.4 启动网络上的集群管理器

启动控制节点和代理节点之后，可以通过DolphinDB提供的集群管理界面来开启或关闭数据节点。在浏览器的地址栏中输入(目前支持浏览器为Chrome与Firefox)：

```txt
 192.168.1.103:8920
```

(8920为控制节点的端口号)

![集群](images/cluster_web.JPG)

#### 3.3.5 DolphinDB权限控制

DolphinDB提供了良好的安全机制。只有系统管理员才有权限做集群部署。在初次使用DolphinDB网络集群管理器时，需要用以下默认的系统管理员账号登录。

```txt
系统管理员帐号名: admin
默认密码       : 123456
```

点击登录链接：

![登陆](images/login_logo.JPG)

输入管理员用户名和密码：

![输入账号](images/login.JPG)

使用上述账号登录以后，可修改"admin"的密码，亦可添加用户或其他管理员账户。

#### 3.3.6 启动数据节点

选择所有数据节点，点击执行图标并确定。节点启动可能要耗时30秒到一分钟。点击刷新图标来查看状态。若看到State栏全部为绿色对勾，则整个集群已经成功启动。

![启动节点](images/cluster_web_start_node.JPG)

![集群启动](images/cluster_web_started.JPG)

如果出现长时间无法正常启动，请查看log目录下该节点的logFile. 如果节点名字是DFS_NODE1，那对应的logFile应该在 log/DFS_NODE1.log。

log文件中有可能出现错误信息"Failed to bind the socket on XXXX"，这里的XXXX是待启动的节点端口号。这可能是因为此端口号被其它程序占用，这种情况下将其他程序关闭再重新启动节点即可。也可能是因为刚刚关闭了使用此端口的数据节点，Linux kernel还没有释放此端口号。这种情况下稍等30秒，再启动节点即可。

也可在控制节点执行以下代码来启动数据节点：

```txt
startDataNode(["DFS_NODE1", "DFS_NODE2","DFS_NODE3","DFS_NODE4"])
```

#### 3.3.7 节点启动失败可能原因分析

如果节点长时间无法启动，可能有以下原因：

1. **端口号被占用**。查看log文件，如果log文件中出现错误信息"Failed to bind the socket on XXXX"，这里的XXXX是待启动的节点端口号。这可能是该端口号已经被其他程序占用，这种情况下将其他程序关闭或者重新给DolphinDB节点分配端口号在重新启动节点即可，也有可能是刚刚关闭了该节点，Linux kernel还没有释放此端口号。这种情况下稍等30秒，再启动节点即可。

2. **防火墙未开放端口**。防火墙会对一些端口进行限制，如果使用到这些端口，需要在防火墙中开放这些端口。

3. **配置文件中的IP地址、端口号或节点别名没有书写正确。**

4. 如果集群是部署在**云端**或**k8s**环境，需要在`agent.cfg`和`cluster.cfg`文件中加上配置项`lanCluster=0`。

## 4. 基于Web的集群管理

经过上述步骤，我们已经成功部署DolphinDB集群。在实际使用中我们经常会需要改变集群配置。DolphinDB的网络界面提供更改集群配置的所有功能。

### 4.1. 控制节点参数配置

点击"Controller Config"按钮会弹出一个控制界面，这里的localExectors, maxConnections, maxMemSize, webWorkerNum以及workerNum等参数是我们在3.1.1中创建controller.cfg时填写的。这些配置信息都可以在这个界面上更改，新的配置会在重启控制节点之后生效。注意如果改变控制节点的localSite参数值，一定要在所有agent.cfg中对controllerSite参数值应做相应修改，否则会造成集群无法正常运行。

![controller_config](images/cluster_web_controller_config.JPG)

### 4.2. 增删数据节点

点击"Nodes Setup"按钮，会进入集群节点配置界面。下图显示的配置信息是我们在3.1.2中创建的cluster.nodes中的信息。在此界面中可以添加或删除数据节点。新的配置会在整个集群重启之后生效。集群重启的具体步骤为：（1）关闭所有数据节点，（2）关闭控制节点，（3）启动控制节点，（4）启动数据节点。另外需要注意，如果节点上已经存放数据，删除节点有可能会造成数据丢失。

![nodes_setup](images/cluster_web_nodes_setup.JPG)

若新的数据节点位于一个新的物理机器上，我们必须在此物理机器上根据3.2中的步骤配置并启动一个新的代理节点，在cluster.nodes中增添有关新的代理节点和数据节点的信息，并重新启动控制节点。

### 4.3. 修改数据节点参数

点击"Nodes Config"按钮, 可进行数据节点配置。以下参数是我们在3.1.3中创建cluster.cfg中提供的。除了这些参数之外，用户还可以根据实际应用在这里添加配置其它参数。重启所有数据节点后即可生效。

![nodes_config](images/cluster_web_nodes_config.JPG)

## 5. DolphinDB 集群详细配置以及参数意义

* [中文](https://www.dolphindb.cn/cn/help/ClusterSetup.html)
* [英文](https://www.dolphindb.com/help/ClusterSetup.html)
