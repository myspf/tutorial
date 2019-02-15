# DolphinDB 高性能集群配置

DolphinDB架构设计上保证了随着计算机资源的增加，性能几乎线性增加。DolphinDB提供了一系列参数，保证最优化集群配置，充分利用硬件资源（包括cpu、内存、磁盘、网络）。合理的参数配置，使系统均衡合理的使用这些资源，最大化发挥机器的性能。

各种相关参数如下：
* cpu：workerNum、localExecutors、maxBatchJobWorker、maxDynamicWorker、webWorkerNum
* 内存：maxMemSize
* 磁盘：volumes、diskIOParallelLevel
* 网络：maxConnections、maxConnectionPerSite、tcpNoDelay

DolphinDB作为系统级的数据库，包括数据写入以及分析，流计算引擎两大应用场景。

DolphinDB集群包括controller、agent、datanode，agent只负责关闭、启动datanode。因此集群的参数配置只考虑controller和datanode即可，而上面的参数对controller和datanode都适用。

本文主要介绍如何合理的配置dolphindb的各种选项，使得充分的发挥机器的性能。集群的搭建请参考教程：[单物理机集群搭建]  

### 1. 系统性能综述
dolphindb是一个功能强大，支持多用户的分布式数据库系统。并且还集成了流数据，作业处理，分布式文件系统等功能，支持海量数据的高并发读写，流式计算。整个系统采用多线程架构，数据库文件的存储支持多种类型的分区。dolphindb的架构设计，决定了其在廉价多台pc机，以及高性服务器上都能最大化发挥硬件性能。

计算机的资源无非包括cpu、内存、网络、磁盘。

cpu 决定系统的运算速度，dolphindb每个数据节点都可以做了client，来接受用户请求，进行计算，如果需要远端数据，同时会把任务发给远端的数据节点。在大数据系统里，往往用户的一个任务会被系统分解成非常多的小任务，这些小任务通常都可以并发执行。workernum  和 local excutor来完成任务的分解和执行。因此这两个参数会直接决定系统的并发度。
同时，dolphindb系统还引入了其他参数，来应对其他的场景，比如http请求并发度，作业任务并发度。

memory 是现代计算机提升性能的关键，为了充分的利用内存，dolphindb自己管理内存和cache。参数 maxMemSize 决定了节点可用的最大，该参数越大系统的性能越高。当然，如果该参数设置的内存超出操作系统可提供的最大内存，那么会导致dolphidnb被系统kill。

disk 存储数据文件，数据库系统频繁的从磁盘加载数据，并不定期的写入数据，磁盘I/O往往成为性能的瓶颈，如果多个线程同时访问同一个磁盘volume，那么显而易见，性能会非常糟糕，如果系统有多个volumes，dolphindb也提供了配置选项来同时利用多个volume的读写能力。volomns 指明节点存储数据可以使用的volume。


dolphindb的采用的是分布式存储技术，也就是说数据会按照分区存在不同的节点上，计算的时候会先在有数据的节点进行计算，然后汇总，




### 1. 选项介绍
按照硬件资源CPU、内存、磁盘、网络相关的选项进行介绍

#### 1.1 CPU相关选项
dolphindb架构采用多线程技术，合理的并发度能极大提升系统性能。并发度太低，不利用系统使用硬件的多线程能力，并发度太高，容易导致过多的线程切换，造成总体性能降低。影响并发度的主要参数如下:

__workerNum__  
woker负责接收客户端请求，分解任务，根据任务粒度自己执行或者交给excutor执行。该参数直接决定了系统的并发数，推荐设置为：xxx


__localExecutors__  
localExcutor负责执行woker分解的任务。和workNum类似，直接决定了系统的并发度，推荐设置为：


相对于上面的两个参数，下面的几个参数不会直接影响性能，只是特定场景下影响系统性能。
 
__maxBatchJobWorker__  
batchJob Worker 执行批处理任务，这些任务是指通过submitJob函数提交的任务，通常耗时较长。该参数决定了执行批处理任务的并发度。一般情况下，推荐设置位： xxxx。 

注意：如果没有批处理任务，创建的线程会回收，所以并不会占用系统资源。


__maxDynamicWorker__  
dynamic worker作为上面介绍的worker的补充，当所有的worker线程占满后，如果新的任务进来，则会创建dynamic worker来执行。推荐设置位：xxx。并且用完后会释放资源。





#### 1.2 系统管理员

DolphinDB集群第一次启动时，会自动创建用户名为"admin"，密码为"123456"的系统管理员。此管理员拥有所有的权限，例如创建或删除数据库，可读写所有表，在数据库中创建或删除数据表，可使用所有的函数视图，可执行或测试脚本。此管理员可以创建或删除其他管理员。其他管理员刚被创建后，可创建或删除其他管理员，用户，或组，但除此之外没有任何权限。管理员可自我授权，亦可互相授权。请注意管理员"admin"无法被删除。

管理员可使用函数或命令`createUser`, `deleteUser`, `createGroup`, `deleteGroup`, `addGroupMember`, `deleteGroupMember`, `getUserAccess`, `getUserList`, `getGroupList`, `resetPwd`对用户和组进行方便的操作。

#### 1.3 权限类别

DolphinDB提供以下8种权限类别:

1. TABLE_READ: 从指定数据表中读取数据 
2. TABLE_WRITE: 将数据写入指定数据表  
3. DBOBJ_CREATE: 创建指定数据库中的对象(数据表) 
4. DBOBJ_DELETE: 删除指定数据库中的对象(数据表) 
5. VIEW_EXEC: 运行函数视图 
6. DB_MANAGE: 创建和删除数据库   
7. SCRIPT_EXEC: 运行脚本文件   
8. TEST_EXEC: 执行单元测试  

其中前面5种需要提供操作对象，后面3种不需要提供操作对象。

请注意，权限配置中涉及到的数据库及数据表均为在分布式文件系统（DFS）中建立的。

#### 1.4 权限设置

只有管理员才可设置权限，且只能在控制节点上执行权限类操作。刚创建的用户或组没有被赋予或被禁止任何权限。管理员可使用`grant`/`deny`/`revoke`命令来设置用户或者组的权限。在1.3种的8种权限，可作为这三个命令的accessType参数值。

以下通过两个例子，说明权限设置的操作。

#### 例子1
管理员登录：
```
login(`admin, `123456)
```
创建用户 NickFoles：
```
createUser("NickFoles","AB123!@")  
```
赋予用户 NickFoles 可读任何DFS数据表的权限：
```
grant("NickFoles",TABLE_READ,"*") 
```
禁止用户 NickFoles 创建或删除数据库：
```
deny("NickFoles",DB_MANAGE)   
```
创建 SBMVP 组，并且把用户 NickFoles 加入到该组： 
```
createGroup("SBMVP", "NickFoles")  
```
赋予 SBMVP 组可在数据库"dfs://db1"和"dfs://db2"中创建数据表的权限：
```
grant("SBMVP",DBOBJ_CREATE,["dfs://db1","dfs://db2"])    
```

最后用户 NickFoles 的权限为：可以访问所有的数据表，不能创建或删除数据库，可以对"dfs://db1"和"dfs://db2"进行创建数据表的操作。  

#### 例子2

通过组可以方便的设置用户权限:
```
createUser("EliManning", "AB123!@")  
createUser("JoeFlacco","CD234@#")  
createUser("DeionSanders","EF345#$")  
createGroup("football", ["EliManning","JoeFlacco","DeionSanders"])  
grant("football", TABLE_READ, "dfs://TAQ/quotes")  
grant("DeionSanders", DB_MANAGE)  
```

该例子创建了3个用户和1个组，并且这三个用户属于该组。赋予此组可读数据表"dfs://TAQ/quotes"的权限，同时只赋予用户DeionSanders创建和删除数据库的权限。

#### 例子3
可以使用`grant`或`deny`对所有对象(以\*代表)赋予或禁止权限。例如，赋予用户 JoeFlacco 可读任何DFS数据表的权限：
```
grant("JoeFlacco",TABLE_READ,"*")  
```
当`grant`或`deny`所有对象后，只可以对所有对象使用`revoke`，若对某个单一对象使用`revoke`无效:  
```
revoke("JoeFlacco",TABLE_READ,"dfs://db1/t1")
```
以上命令无效。

```
revoke("JoeFlacco",TABLE_READ,"*")
```
以上命令取消了用户 JoeFlacco 可读任何DFS数据表的权限。

与之类似，使用`grant`或`deny`对组赋予或禁止权限后，只能对改组使用`revoke`来取消该权限设置。若对某个组员使用`revoke`来取消该权限设置则无效。

#### 1.5 权限确定规则

用户的最终权限由用户本身的权限与其所属的所有组的权限共同决定。不同的组有可能对某用户某权限的规定有冲突。以下为权限确定规则：
* 若用户在至少一组中被赋予某权限，而且没有在任一组中被禁止该权限，则此用户拥有此权限。
* 若用户在至少一组中被禁止某权限，即使该用户在其它组中被赋予此权限，也被被禁止了此权限。此情况下该用户要获得此权限，管理员必须在所有禁止该权限的组中使用`revoke`或`grant`以取消这些权限禁止，并且该用户至少在一组中被赋予此权限。
请注意，在上述规则中，为了叙述方便，用户本身也可视为一个特殊的组。

```  
createUser("user1","123456")  
createUser("user2","123456")  
createGroup("group1")  
createGroup("group2")  
addGroupMember(["user1","user2"],"group1")
addGroupMember(["user1","user2"],"group2")
grant("user1",TABLE_READ,"*")  
deny("group1",TABLE_READ,"dfs://db1/t1")  
deny("group2",TABLE_READ,"dfs://db1/t2")   
```  
以上三行的结果为，用户user1可以读取除"dfs://db1/t1"和"dfs://db1/t2"以外的所有数据表。  

``` 
grant("user2",TABLE_WRITE,"*")  
deny("group1",TABLE_WRITE,"*")  
grant("group2",TABLE_WRITE,"dfs://db1/t2")  
```
以上三行的结果为，用户user1和user2不能写数据到任何数据表。


#### 1.6 基于函数视图（function view）的权限控制

函数视图提供了一种灵活的方式来控制用户访问数据表，在不给予用户可以阅读数据表所有原始数据的权限的情况下，让用户可以获取由函数视图产生的信息。只有系统管理员有创建和删除函数视图的权限。

管理员定义一个函数视图：  
```
def countTradeAll(){  
	return exec count(*) from loadTable("dfs://TAQ","Trades")  
}
addFunctionView(countTradeAll)  
grant("NickFoles",VIEW_EXEC,"countTradeAll")  
```
以用户名NickFoles登录，执行视图countTradeAll
```
countTradeAll()
```
虽然用户NickFoles没有访问表"dfs://TAQ/Trades"的权限，但是可以运行函数视图在此例中来获取表的行数。

函数视图也可以带参数。用户在使用的时候可以输入参数获取相应的结果。下面的例子，我们创建一个函数视图，获取某一个股票在某一天的所有交易记录。
```
def getTrades(s, d){
	return select * from loadTable("dfs://TAQ","Trades") where sym=s, date=d
}
addFunctionView(getTrades)
grant("NickFoles",VIEW_EXEC,"getTrades")  
```
以用户名NickFoles登录，并在执行视图getTrades指定股票代码和日期：
```
getTrades("IBM", 2018.07.09)
```

### 2. 程序调度和流计算中的权限控制

程序调度和流计算在后台运行，很多情况下，没有显式登录，因此权限验证跟用户显式登录的情况有些不同。这两类后台任务都是以创建该任务的用户身份来运行。

#### 2.1 schedule jobs 权限设置

程序调度是指用户指定在特定的时间，以特定的频率执行一系列任务，多用于适合批处理类业务场景。 
```
login("NickFoles","AB123!@")  
def readTable(){  
	read_t1=loadTable("dfs://db1","t1")  
	return exec count(*) from read_t1  
}  
scheduleJob("readTableJob","read DFS table",readTable,minute(now()),date(now()),date(now())+1,'D');  
```

不管NickFoles有没有读"dfs://db1/t1"的权限，`readTable`任务都能设置成功。  

在readTable任务实际运行时，如果用户NickFoles有读"dfs://db1/t1"的权限，则成功执行，否则鉴权失败。  

另外，使用`deleteScheduledJob`命令的的时候，系统管理员可以删除其他用户制定的任务，非管理员用户只能删除自己创建的任务。

#### 2.2 streaming 权限设置

当用户使用`subscribeTable`函数从一个流数据表中订阅实时数据存入数据表时，应当确认其有写入此数据表的权限。

```
login("NickFoles","AB123!@")
def saveTradesToDFS(mutable dfsTrades, msg): dfsTrades.append!(select today() as date, * from msg)  
subscribeTable("NODE1", "trades_stream", "trades", 0, saveTradesToDFS{trades}, true, 1000, 1)  
```

在上例中，流数据处理任务把接收到的数据保存到数据表dfsTrades中。在此流数据处理任务执行的时候，系统会动态地鉴定NickFoles是否有写入数据表dfsTrades的权限，如果没有则鉴权失败。

### 3. 使用HTTPS实现安全通信

DolphinDB支持使用HTTPS安全协议与web进行通信。

#### 3.1 使能HTTPS配置

两种使能配置HTTPS的方法：

+ 控制节点配置文件中使能。在控制节点的配置文件中添加"enableHTTPS=true"。

+ 在启动控制节点的命令行中使能。启动命令行添加 "-enableHTTPS true"。

#### 3.2 HTTPS证书设置

DolphinDB使用服务端证书验证的安全策略。默认情况下，会生成自制证书，客户需要安装服务端的证书，否则浏览器提示不安全连接。每台物理server需要一份证书，因此控制节点和代理节点需要生成证书，数据节点使用同一个物理服务器上代理节点生成的证书。用户也可购买经过第三方认证的证书。

##### 3.2.1 第三方认证证书

将第三方证书重命名为 server.crt，并且拷贝到控制节点和代理节点的home目录下的keys文件夹中，若keys文件夹不存在，则需手动创建。由于第三方证书经过公认的权威授权机构颁布，所以浏览器默认信任该证书，不需要再手动安装。绝大部分应用场景适合使用此方式。

##### 3.2.2 安装自制证书

在小型封闭集群内部通信时，用户也可以使用自制证书进行OPENSSL安全通信，不过安装自制证书稍微繁琐，具体过程如下:

> 1. 设置 publicName
由于证书的生成需要知道计算机的域名，对需要生成证书的物理server设置该选项为计算机的域名，可以在命令行或配置文件中设置。下面是在Linux中启动控制节点的命令行例子。这里www.ABCD.com 是控制节点所在计算机的域名。  
```
./dolphindb -enableHTTPS true -home master -publicName www.ABCD.com -mode controller -localSite 192.168.1.30:8500:rh8500 -logFile  ./log/master.log
```
> 2. 查看证书是否正确生成  
启动控制节点， 在home目录下的keys文件夹中，查看是否有自制证书文件*server.crt*和私钥文件*serverPrivate.key*。

> 3. 安装自制证书到浏览器的授信证书中心
不同的浏览器安装选项稍有不同，以Google Chrome为例，选择Settings->Advanced->Manage certificates->AUTHORITIES->Import,导入上面生成的server.crt文件。

![](images/Selection_047.png)  

在浏览器中输入 https://www.ABCD.com:8500/ 以访问集群管理器。若浏览器地址栏显示绿色的小锁，说明证书安装成功，可以进行HTTPS访问。

![](images/Selection_046.png)

### 4. 支持SSO （Single Sign On)

在集群管理界面中，可以点击任意数据节点，链接到该节点的notebook上。从控制节点跳转到数据节点，有可能是访问了不同的物理服务器（跨域访问）。DolphinDB提供了SSO使得用户无须在集群操作中访问不同的物理服务器时重新登陆系统。

DolphinDB提供两个用于SSO的API函数：
+ `getAuthenticatedUserTicket()` 获取当前登录用户的加密ticket
+ `authenticateByTicket(ticket)` 使用上面获取的ticket登录系统

DolphinDB的开发者可以方便安全的使用这些接口来对系统进行扩展。



