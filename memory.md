# DolphinDB 内存管理

DolphinDB是一款提供多用户并发读写的分布式数据库软件，其中高效的内存管理是其性能优异的基础，DolphinDB内存管理包括以下功能：

* __session变量内存管理__ ，为用户提供和回收编程环境所需内存，隔离session间的内存空间； 
* __分布式表的内存管理__ ，session间共享分区表数据，以提高内存使用率；
* __为流数据提供缓存__ ，流数据发送节点提供持久化和发送队列缓存,订阅节点提供接受队列缓存； 
* __为写入DFS数据库提供缓存__ ，通过批量写入DFS大幅提升写入吞吐量；

## 1. 概述
### 1.1 Session和user
DolphinDB通过session来隔离不同用户间的内存空间，通过GUI，web或者其他API链接到server，即创建了一个session。用户登录一个session可以使用该session中年的内存变量。私有变量的内存都是存在于某一个session中。如下图：



usr1可以登录session1，创建变量v和t。如果此时，usr2登录到该session中，则usr2可以看到并且使用session1中的变量。
因此，session类似容器，里面真正持有变量空间。用户类似观察者，可以登录不同的session查看和使用该session中年的内存和变量。

### 1.2 share变量和session变量

session变量只在定义的session中可见，定义变量的默认方式为session变量。比如 ```v = 1..100000```，v为私有变量仅在定义的session中可见。
share变量不属于某一个session，在所有session中可见，目前仅 __table__ 可以设置为share属性。可通过如下方式创建share变量：
```
t = table(1 2 3 as id, 4 5 6 as num)
share t as st
```
则st为share变量，在所有的session中共享，不属于某一个session。如下图所示：



### 1.3 内存查看方式
函数getSessionMemoryStat() 查看每个session占用的内存空间。


函数mem()来查看DolphinDB server 总的内存占用。


后续通过这两个函数查看示例的内存。



## 2. session变量内存管理

### 2.1 创建变量

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

DolphinDB提供不同session间的内存隔离，不同的session中创建同名变量，内存空间占用也是完全独立的。例如再新建一个session（打开另一个GUI），创建同名的vector以及table。如下：
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

由上可知，sevver内存共占用1.2G，在两个session中。当前第一个session中登陆了usr1用户，第二个session中登陆了admin用户。内存空间完全隔离。
  
### 2.2 释放内存

可通过undef函数，释放变量的内存，如下
```
undef(`v)
```
或者
```
v = NULL
```
除了undef函数外，当session关闭时，比如关闭GUI，web notebook 连接断开，或者API 连接端口，都会触发serverd端对该session的所有内存进行回收。


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
流数据系统中，流数据表本身是内存表，会占用内存。同时发送端有发送队列，持久化队列，订阅端有接收队列来缓存数据。
![image](https://github.com/myspf/tutorial/blob/master/streaming.png?raw=true)

如上图当数据进入流数据系统时，首先写入流表内存表，然后进入持久化队列和发送队列（假设用户设置为异步持久化），持久化队列异步写入磁盘，发送队列发送到订阅端。流表本身的内存占用取决于函数enableTablePersistence()的第四个参数capacity，该参数指定流表中保存消息的最大行数。 同时由于网络或者磁盘瓶颈，可能会导致持久化队列和发送队列的数据堆积，而占用内存。  
当订阅端收到数据后，先放入接受队列，然后用户定义的handler从接收队列中取数据并处理。如果handler处理缓慢，会导致接收队列有数据堆积，占用内存。  

上面的三个队列的容量可以分别通过参数 maxPersistenceQueueDepth，maxPubQueueDepthPerSite，maxSubQueueDepth 来设置大小，单位为行。运行过程，可以通过函数 getStreamingStat() 的输出来流表内存中保留的数据量以及各个队列深度。  

## 5 为写入DFS数据库提供缓存





### 6 内存使用建议
在企业的生产环境中，DolphinDB往往作为流数据中心以及历史数据仓库，给业务人员提供数据查询和计算。当用户较多时，不当的使用容易造成Server端内存耗尽，抛出“out of memory” 异常。可遵循以下建议，尽量避免内存的不合理使用。

#### 6.1 内存相关选项推荐配置
*  __maxMemSize__ : DolphinDB 实例使用的最大内存量，应该根据系统实际的物理内存，以及节点的个数来合理的设置该值。设置的越大，性能以及系统的处理能力越大，但如果设置值超粗了系统提供的内存大小，则有可能会被操作系统杀掉。比如操作系统实际的物理内存是16G，该选项设置为32G，那么运行过程中，有可能被操作系统默默杀掉。
*  __maxPersistenceQueueDepth__ : 流表持久化队列的最大消息数。对于异步持久化的发布流表，先异步的将数据放到持久化队列中，然后负责持久化的线程（persistenceWorkerNum设置持久化线程数）不断的讲队列中的消息持久化到磁盘上。该选项指明了，该流表持久化队列中的最大消息条数。默认设置为1000000。

*  __maxPubQueueDepthPerSite__ : 最大消息发布队列深度。针对某个订阅节点，发布节点建立一个消息发布队列，该队列中的消息发送到订阅端。该选项指明，该发布节点针对某个订阅节点的消息发布队列的最大深度。默认值为10000000。

*  __maxSubQueueDepth__ : 订阅节点上最大的每个订阅线程最大的可接收消息的队列深度。订阅的消息，会先放入订阅消息队列，该值指明该队列的大小。默认设置为10000000。

*  __chunkCacheEngineMemSize__ : 指定cache engine的容量（单位为GB）。cache engine开启后，写入数据时，系统会先把数据写入缓存，当缓存中的数据量达到chunkCacheEngineMemSize的30%时，才会写入磁盘。默认值是0，即不开启cache engine。开启cache engine的同时，必须设置dataSync=1。

#### 6.2 高效实用内存编程原则
*  __合理均匀分区__ , DolphinDB是以分区为单位加载数据，因此，分区大小对内存影响巨大。合理均匀的分区，不管对内存使用还是对性能而言，都是有积极的作用。因此，在创建数据库的时候，根据数据规模，合理规划分区大小。每个分区的常用字段大小约100左右为宜。
   
*  __用户尽量避免持有大量数据的变量__, 当用户数据量较大的变量，比如 ```v = 1..10000000```,或者把查询结果赋值给一个变量 ```t = select * from t where date = 2010.01.01```，那么v和t将会在用户的session占用大量的内存，如果不及时释放，当其他用户申请内存时，就有可能因为内存不足而抛出异常。

*  __只查询需要的列，避免使用select \*__,如果用select \*则会把该分区所有列加载到内存，实际中，往往只需要几列，因此为避免内存浪费，尽量明确写出所有查询的列，而不是用\*代替。

*  __数据查询尽可能的加分区过滤条件__,DolphinDB按照分区进行数据检索，如果不加分区过滤条件，则会全部扫描所有数据，内存很快被耗尽。有多个过滤条件的话，select语句要优先写分区的过滤条件。

*  __用完的变量或者session，尽快释放__,根据上面的分析可以知道，用户的私有变量在创建的session里面保存，并且session关闭的时候，会回收这些内存。因此，及早释放这些私有变量（undef）或者关闭session来释放内存。

