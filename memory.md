# DolphinDB 内存管理

DolphinDB是分布式数据库，支持多用户同时在线计算分析，以及对历史数据库的并发查询。通过内存管理，来为不同的用户合理的分配和回收内存，以及提供分布式表在多用户间的共享，当多用户对同一个分布式表查询时，内存中的数据只有一份数据，而不是每个用户有一份拷贝，这样可以极大的提升内存的利用率。 DolphinDB的内存管理主要提供以下功能：

* 为用户提供私有变量的内存空间，隔离会话间的内存空间；
* 多用户间共享分区表，提高内存使用率；
* 为流数据提供消息缓存队列，作为发布订阅消息队列的缓存；

## 1. 概述

DolphinDB server提供多用户同时并发访问，为每个用户创建一个session，达到多用户内存和变量的隔离。比如user1 创建表t，user2也创建同名表t，显然两个用户的表t虽然同名，确是完全独立的，底层在内存分上也是独立的。
我们通过两个视角来观察内存使用，一个是每个用户占用内存大小；另一个是从全局的角度看，server上内存的分配和使用情况。通过两个函数来对用户和全局内存进行观察。
通过getSessionMemoryStat()，来查看每个用户所占用的内存大小。
```
getSessionMemoryStat()
```
输出图片
getSessionMemoryStat()输出的详细解释。

通过函数mem()来查看某个DolphinDB server 所用的内存大小。
```
mem()
```
输出图片
详细解释各个输出列。
后续使用这两个函数查看内存变化。

## 2. 用户私有变量的内存管理
创建的非share类型的变量都属于用户的私有变量，典型的分布式表和共享流表不属于某个用户的私有变量。再次强调，DolphinDB的内存隔离是以session为区别的，与用户无关。也就是说，内存变量只存在于session中，不同的用户在该session中登录，就会看到这些变量。不同的会话间，是完全隔离的，因此可以有同名的变量。
会话类似一个容器，里面包括各种变量，哪个用户登录到该容器中，就会看到容器中的所有变量。
当打开GUI，通过web notebook，各种API接口，甚至console连接server的时候，都创建了新的会话。

### 2.1 创建私有变量

我们在single mode单节点上，先创建一个用户user1，然后登陆。创建vector变量，1亿个int类型，约400兆。
```
login("admin","123456")  //创建用户需要登陆admin
createUser("user1","123456")
login("user1",123456)
v = 1..100000000
sum(mem().blockSize - mem().freeSize) //输出内存占用结果
```
结果为: 402,865,056，内存占用400兆左右，符合预期。

再创建一个table，1000万行，5列，每列4字节，约200兆。
```
n = 10000000
t = table(n:n,["tag1","tag2","tag3","tag4","tag5"],[INT,INT,INT,INT,INT])
(mem().blockSize - mem().freeSize).sum()
```
结果为：612,530,448，约600兆，符合预期。

### 2.2 session间私有变量的内存隔离
DolphinDB提供不同session间的内存隔离。即使同一用户，在不同的session中创建同名变量，内存空间占用也是完全独立的。  
再新建一个session（打开另一个GUI），同样用user1登陆，创建同名的vector以及table。代码如下：  
```
login("user1","123456")
v = 1..100000000
n = 10000000
t = table(n:n,["tag1","tag2","tag3","tag4","tag5"],[INT,INT,INT,INT,INT])
(mem().blockSize - mem().freeSize).sum()
```
结果为 ：1224926000，占用内存约1.2G。

用函数getSessionMemoryStat()查看sesion的内存，如下：
```
login(`admin,`123456)
getSessionMemoryStat()
```
输出如下：
  userId|sessionId|memSize
  ---|:--:|---:
  admin| 4,203,157,148 | 612,369,840
  user1| 1,769,725,800 | 612,369,840

由上可知，sever内存共占用1.2G，在两个session中。大家可能会有疑问，这个session也是用user1登录，创建的变量，为什么admin用户占用600兆。原因是，查看内存，执行getSessionMemoryStat()函数前，我们用admin登录了（只有admin用户有执行该函数的权限），因此该session当前的user为admin而非user1。
  
### 2.3 私有变量的内存释放
对于会话中私有变量的释放，通常有两种方式，手动undef或者关闭session。
#### 2.3.1 undef释放私有变量
可通过undef函数，释放变量的内存，如下
```
undef(`v)
```

#### 2.3.2 关闭session，释放变量
当session关闭，比如关闭GUI，web notebook 连接断开，或者API 连接端口，都会触发serverd端，对该会话的所有私有变量的回收。
例如我们创建vector ：
```
login("admin",123456)
v = 1..100000000
sum(mem().blockSize - mem().freeSize) 
```
 输出结果为 402,865,056，内存占用400兆400兆，此时关闭GUI，再打开，观察内存占用，
```
sum(mem().blockSize - mem().freeSize) 
```
输出结果为 ：174,992，内存已经全部被回收。

## 3 分布式表的内存管理
分布式表的内存是全局共享的，对分布式表某个分区的数据，在内存中只保留一份数据，不管登录到哪个session中，都可以看到相同的一份数据，这样极大的节省了内存的使用。
历史数据库都是以分布式表的形式存在数据库中，用户平时查询操作也往往直接与分布式表交互。分布式表的内存管理有如下特点：

* 内存是以分区为单位进行管理；
* 同一分区，内存只保留一份数据；
* 内存允许情况下，尽量多缓存数据；
* 缓存达到用户设置上限时，会自动回收；

下面我们通过一个具体的实例介绍以上特点。搭建的集群采用2个节点，单副本模式，按天分30个区，每个分区1000万行，10列（1列为timestamp类型，9列为LONG类型），所以每个分区的每列 1000万行 * 8字节/列 = 80M，每个分区共1000 万行 * 80字节/行 = 800M，整个表共3亿行，大小为24GB。
>函数clearAllCache() 可以情况已经缓存的数据，下面的每次测试前，先用该函数清空节点上的所有缓存。

### 3.1 内存以分区为单位进行管理
DolphinDB是列式存，当用户对分布式表的数据进行查询时，加载数据的原则是，只把用户所要求的分区和列加载到内存中。而且，除非用户取数据（比如用t = select ..），那么数据只会加载到所在的节点上，不会传输到其他节点。
示例1 在node1 上加载 1个分区（2019.01.01，该分区在）的一列数据tag1，该分区储存在node1上，可以在controller上通过函数getClusterChunksStatus()查看分区分布情况。
```
select max(tag1) from loadTable(dbName,tableName) where date = 2019.01.01
sum(mem().blockSize - mem().freeSize) 
```
输出结果为：168,000,992。上面的select需要两列数据，分区deta和tag1，按照上面的计算每列80M，两列160M左右符合预期。

示例2 在node1 上查询 2019.01.01的所有数据，并观察内存占用
```
select * from loadTable(dbName,tableName) where date = 2019.01.01
sum(mem().blockSize - mem().freeSize)
```
输出结果为： 839,101,024，按照上面计算，每个分区800M，结果符合预期。

示例3 在node2 上查询 存储在node1上的分区数据 2019.01.01。GUI上连接node2，然后执行查询
```
select max(tag1),min(tag2) from loadTable(dbName,tableName) where date = 2019.01.01
sum(mem().blockSize - mem().freeSize) 
```
输出结果为： 193504。由于分区存储在node1上，虽然通过node2查询，但是数据仍然只加载到node1的内存。所以node2上内存显示很小。这时到node1上查看内存占用结果为 251,905,744，共3列数据，结果符合预期。  
所以在数据分区设计的时候，合理考虑分区大小以及分区间数据量尽量相近尤为关键。

### 3.2 内存中只保留数据的一份副本
DolphinDB支持海量数据的并发查询，为了高效利用内存，对相同分区的数据，内存中只保留一份数据。
示例4 打开两个GUI，分别登陆连接node1和node2，查询分区2019.01.01的数据。
```
select * from loadTable(dbName,tableName) where date = 2019.01.01
sum(mem().blockSize - mem().freeSize)
```
上面的代码不管执行几次，node1上内存显示一直是 839,101,024，而node2上无内存占用。因为分区数据只存储在node1上，所以node1会加载所有数据，而node2不占用任何内存。

### 3.3 内存允许情况下，尽量多缓存数据
通常情况下，最近访问的数据往往更容易再次被访问，因此DolphinDB在内存允许的情况下（内存占用不超过用户设置的maxMemSize），尽量多缓存数据，来提升后续数据的访问效率。
示例5：在node1 上先加载2019.01.01的分区数据，查看内存
```
select * from loadTable(dbName,tableName) where date = 2019.01.01
sum(mem().blockSize - mem().freeSize)
```
输出结果为：839118272。再查询 2019.01.03的数据，该分区也存储在node1上，
```
select * from loadTable(dbName,tableName) where date = 2019.01.03
sum(mem().blockSize - mem().freeSize)
```
输出结果为：1,678,002,080。共1.6G，包含两个分区的数据，之前分区的数据仍然在内存中保留。

### 3.4  缓存达到用户设置上限时，会自动回收
如果DolphinDB server使用的内存，没有超过用户设置的maxMemSize，则不会回收内存。当总的内存使用达到maxMemSize时，DolphinDB 会采用LRU的内存回收策略， 来腾出足够的内存给用户。
示例：不断的查询导致，缓存接近maxMemSize，然后用户申请大量内存，如果不清理缓存则不足以提供足够的内存给用户。
```
select * from loadTable(dbName,tableName) where date = 2019.01.03
sum(mem().blockSize - mem().freeSize)
```
内存占用达到xxxx，然后继续申请内存
```
v = 1..1000000000
sum(mem().blockSize - mem().freeSize)
```
说明DolphinB自动回收了足够的内存供用户使用。

### 3.5 内存使用量和分区大小的关系
由于DolphinDB是以分区为单位管理内存，因此内存的使用量跟分区关系密切。假如用户分区不均匀，导致某个分区数据量超大，甚至整个都不足以容纳整个分区，那么当涉及到该分区的查询计算时，系统会抛出“out of memory”的异常。
一般原则，如果用户设置的maxMemSize大小为8GB，则每个分区常用的查询列之和为100-200兆为宜。
如果表有10列常用查询字段，每列8字段，则每个分区约100-200万行。


## 4 作为流数据消息缓存队列

!(https://github.com/myspf/tutorial/blob/master/streaming.png?raw=true)
![image](https://github.com/myspf/tutorial/blob/master/streaming.png?raw=true)


## 5 内存常用函数（包括监控，清理内存）





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
