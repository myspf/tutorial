# DolphinDB 内存管理
DolphinDB是一款提供多用户并发读写的分布式数据库软件，其中高效的内存管理是其性能优异的基础，DolphinDB内存管理包括以下方面：

* __Session变量内存管理__ ，为用户提供和回收编程环境所需内存，隔离Session间的内存空间； 
* __分布式表的内存管理__ ，Session间共享分区表数据，以提高内存使用率；
* __为流数据提供缓存__ ，流数据发送节点提供持久化和发送队列缓存,订阅节点提供接收数据队列缓存； 
* __为写入DFS数据库提供缓存__ ，写入DFS的数据先写到缓存里里面， 通过批量写入大幅提升吞吐量；

## 1. 概述
DolphinDB的内存管理涉及到几个概念，首先是Session和user，Session是提供内存变量的载体，用户通过登录可以访问Session里的变量。  
根据变量的可见性，分为Session变量和share变量，Session变量只在定义的Session中可见，在其他Session中不可见。而Share变量对所有的Session可见。
### 1.1 Session和用户
DolphinDB通过session来隔离不同用户间的内存空间，通过GUI，Web或者其他API链接到server，即创建了一个Session。用户登录一个Session可以使用该Session中年的内存变量。私有变量的内存都是存在于某一个Session中。如下图：

![image](https://github.com/myspf/tutorial/blob/master/user.png) 


usr1可以登录Session1，创建变量v和t。如果此时，usr2登录到该Session中，则usr2可以看到并且使用Session1中的变量。
因此，Session类似容器，里面真正持有变量空间。用户类似观察者，可以登录不同的session查看和使用该Session中年的内存和变量。

### 1.2 Session变量和Share变量

Session变量只在定义的Session中可见，定义变量的默认方式为Session变量。比如 ```v = 1..100000```，v为私有变量仅在定义的Session中可见。
Share变量不属于某一个session，在所有session中可见，目前仅 __table__ 可以设置为Share变量。可通过如下方式创建Share变量。  
示例1 创建share变量
```
t = table(1 2 3 as id, 4 5 6 as num)
share t as st
```
则st为Share变量，在所有的Session中共享，不属于某一个Session。如下图所示：

![image](https://github.com/myspf/tutorial/blob/master/session.png)

### 1.3 查看内存占用量

函数getSessionMemoryStat() 查看每个session占用的内存空间。输出结果为table，包括3列。  

|  userId   | sessionId  | memSize |
|  ----  | ----  | ---- |

 userId表示该Session中登录的用户，sessionId表示session，memSize表示占用内存大小，单位为字节。

函数mem()来查看DolphinDB server 总的内存占用。输出结果为table，包括4列。 

|  blockSize   | freeSize  | freeBlock | leafSize |
| ---- | ----  | ---- | ---- |

blockSize表示已经分配的内存，freeSize 表示未使用内存，blockSize - freeSize 表示实际使用的内存。
后续的示例通过这两个函数查看DolphinDB节点上的内存占用。

## 2. Session变量内存管理

### 2.1 创建变量

我们在DolphinDB 节点上，先创建一个用户user1，然后登陆。创建vector变量，1亿个int类型，约400兆。  
示例2 创建session变量vector
```
login("admin","123456")  //创建用户需要登陆admin
createUser("user1","123456")
login("user1",123456)
v = 1..100000000
sum(mem().blockSize - mem().freeSize) //输出内存占用结果
```
结果为: 402,865,056，内存占用400兆左右，符合预期。

再创建一个table，1000万行，5列，每列4字节，约200兆。  
示例3 创建session变量table
```
n = 10000000
t = table(n:n,["tag1","tag2","tag3","tag4","tag5"],[INT,INT,INT,INT,INT])
(mem().blockSize - mem().freeSize).sum()
```
结果为：612,530,448，约600兆，符合预期。

### 2.2 Session间内存隔离
DolphinDB提供不同session间的内存隔离，不同的session中创建同名变量，内存空间占用也是完全独立的。再新建一个session（打开另一个GUI），创建同名的vector以及table。  
示例4 不同的session中创建session变量
```
login("user1","123456")
v = 1..100000000
n = 10000000
t = table(n:n,["tag1","tag2","tag3","tag4","tag5"],[INT,INT,INT,INT,INT])
(mem().blockSize - mem().freeSize).sum()
```
结果为 ：1224926000，占用内存约1.2G。

用函数getSessionMemoryStat()查看sesion的内存。  
示例5 查看内存占用
```
login(`admin,`123456)
getSessionMemoryStat()
```
输出如下：  

|  userId   | sessionId  | memSize |
|  ----  | ----  | ---- |
| admin  | 4,203,157,148 | 612,369,840 |
| user1  | 1,769,725,800 | 612,369,840 |


由上可知，sevver内存共占用1.2G，在两个Session中。当前第一个Session中登陆了usr1用户，第二个Session中登陆了admin用户。内存空间完全隔离。
  
### 2.3 释放内存

可通过undef函数，释放变量的内存。  
示例6 undef或者赋值为NULL释放session变量
```
undef(`v)
```
或者
```
v = NULL
```
除了手动释放变量，当Sessionn关闭时，比如关闭GUI，web notebook 连接断开，或者API连接端口，都会触发对该Session的所有内存进行回收。

## 3 分布式表的内存管理
DolphinDB中对分布式表是以分区为单位管理的，数据在不同的session间共享，分布式表的内存是全局共享的，不管登录到哪个Session中，都可以看到相同的一份数据，这样极大的节省了内存的使用。
历史数据库都是以分布式表的形式存在数据库中，用户平时查询操作也往往直接与分布式表交互。分布式表的内存管理有如下特点：

* 内存以分区列为单位进行管理；
* 数据只加载到所在的节点，不会在节点间转移；
* 同一分区，内存只保留一份数据；
* 内存使用不超过maxMemSize情况下，尽量多缓存数据；
* 缓存数据达到maxMemSize时，系统自动回收；

下面我们通过一个具体的实例介绍以上特点。搭建的集群采用2个节点，单副本模式，按天分30个区，每个分区1000万行，11列（1列为DATE类型，1列为INT类型，9列为LONG类型），所以每个分区的每列(long类型） 1000万行 * 8字节/列 = 80M，每个分区共1000 万行 * 80字节/行 = 800M，整个表共3亿行，大小为24GB。
>函数clearAllCache() 可以清空已经缓存的数据，下面的每次测试前，先用该函数清空节点上的所有缓存。

### 3.1 内存以分区列为单位进行管理
DolphinDB是列式存储，当用户对分布式表的数据进行查询时，加载数据的原则是，只把用户所要求的分区和列加载到内存中。  
示例7 计算分区2019.01.01最大的tag1的值。该分区储存在node1上，可以在controller上通过函数getClusterChunksStatus()查看分区分布情况，而且由上面可知，每列约80兆。在node1上执行如下代码，并查看内存占用。
```
select max(tag1) from loadTable(dbName,tableName) where day = 2019.01.01
sum(mem().blockSize - mem().freeSize) 
```
输出结果为：84,267,136。我们只查询1个分区的一列数据，所以把该列数据全部加载到内存，其他的列不加载。

示例8 在node1 上查询 2019.01.01的前100条数据，并观察内存占用。
```
select top 100 * from loadTable(dbName,tableName) where day = 2019.01.01
sum(mem().blockSize - mem().freeSize)
```
输出结果为： 839,255,392。虽然我们只取100条数据，但是DolphinDB加载数据的最小单位是分区列，所以需要加载每个列的全部数据，也就是整个分区的全部数据，约800兆。

> __注意：__ 合理分区避免“OOM”，DolphinDB是以分区为单位管理内存，因此内存的使用量跟分区关系密切。假如用户分区不均匀，导致某个分区数据量超大，甚至整个都不足以容纳整个分区，那么当涉及到该分区的查询计算时，系统会抛出“out of memory”的异常。  
一般原则，如果用户设置的maxMemSize大小为8GB，则每个分区常用的查询列之和为100-200兆为宜。  
如果表有10列常用查询字段，每列8字段，则每个分区约100-200万行。  

### 3.2 数据只加载到所在的节点，不会在节点间转移
在数据量大的情况下，节点间转移数据是非常耗时的操作。DolphinDB的数据是分布式存储的，当执行计算任务时，把计算任务发送到数据所在的节点，而不是把数据转移到计算所在的节点，这样大大降低数据在节点间的转移，提升计算效率。

示例9 在node1上计算两个分区tag1的最大值。其中分区2019.01.02数组存储在node1上，分区2019.01.03数据存储在node2上。
```
select max(tag1) from loadTable(dbName,tableName) where day in [2019.01.02,2019.01.03]
sum(mem().blockSize - mem().freeSize) 
```
输出结果为：84,284,096。在node2上用查看内存占用 结果为 84,250,624。每个节点存储的数据都为80M左右，也就是node1上存储了分区 2019.01.02的数据，node2上存储了 2019.01.03的数据。

示例10 在node1上查询分区2019.01.02和2019.01.03的所有数据，我们预期node1加载2019.01.02数据，node2加载2019.01.03的数据，都是800M左右，执行如下代码并观察内存。
```
select top 100 * from loadTable(dbName,tableName) where day in [2019.01.02,2019.01.03]
sum(mem().blockSize - mem().freeSize)
```
node1上输出结果为，839,279,968。node2上输出结果为，839,246,496。结果符合预期

> __注意：__ 谨慎使用“select * ", 如果分区列数很多，比如有500列，100万行，那么如果使用 "select top 100 * "加载数据，则所需内存 1000000 * 500列 * 10字节/列 = 5GB，这样会占用约5GB的内存，因此用户需谨慎使用这种类型的查询操作。

### 3.3 内存中只保留数据的一份副本
DolphinDB支持海量数据的并发查询，为了高效利用内存，对相同分区的数据，内存中只保留一份数据。
示例11 打开两个GUI，分别连接node1和node2，查询分区2019.01.01的数据，该分区的数据存储在node1上。
```
select * from loadTable(dbName,tableName) where date = 2019.01.01
sum(mem().blockSize - mem().freeSize)
```
上面的代码不管执行几次，node1上内存显示一直是 839,101,024，而node2上无内存占用。因为分区数据只存储在node1上，所以node1会加载所有数据，而node2不占用任何内存。

### 3.4 内存使用不超过maxMemSize情况下，尽量多缓存数据
通常情况下，最近访问的数据往往更容易再次被访问，因此DolphinDB在内存允许的情况下（内存占用不超过用户设置的maxMemSize），尽量多缓存数据，来提升后续数据的访问效率。
 
示例12：数据节点设置的maxMemSize = 8GB，我们连续加载9个分区，每个分区约800M，总内存占用约7.2GB，观察内存的变化趋势。
```
days = chunksOfEightDays();
for(d in days){
 select * from loadTable(dbName,tableName) where  = day
 sum(mem().blockSize - mem().freeSize)
}
```
内存随着加载分区数的增加变化规律如下图所示：
![image](https://github.com/myspf/tutorial/blob/master/partition9.png) 

当遍历每个分区数据时，在内存使用量不超过maxMemSize的情况下，分区数据会全部缓存到内存中，以在用户下次访问时，直接从内存中提供数据，而不需要再次从磁盘加载。

### 3.5  缓存数据达到maxMemSize时，系统自动回收
如果DolphinDB server使用的内存，没有超过用户设置的maxMemSize，则不会回收内存。当总的内存使用达到maxMemSize时，DolphinDB 会采用LRU的内存回收策略， 来腾出足够的内存给用户。
示例7，上面用例中我们只加载了8天的数据，此时我们继续共遍历15天数据，查看缓存达到maxMemSize时，内存的占用情况。如下图所示：
![image](https://github.com/myspf/tutorial/blob/master/partiton15.png)   
如上图所示，当缓存的数据超过maxMemSize时，系统自动回收内存，总的内存使用量仍然小于用户设置的最大内存量8GB。

示例13: 当缓存数据接近用户设置的maxMemSize时，继续申请Session变量的内存空间，查看系统内存占用。
此时先查看下系统的内存使用
```
sum(mem().blockSize - mem().freeSize)
```
输出结果为 7,550,138,448。内存占用超过7GB，而用户设置的最大内存使用量为 8GB，此时我们继续申请4GB空间。
```v = 1..1000000000
sum(mem().blockSize - mem().freeSize)
```
输出结果为 8,196,073,856。约为8GB，也就是如果用户定义变量，也会触发缓存数据的内存回收，以保证有足够的内存提供给用户使用。

## 4 作为流数据消息缓存队列
当数据进入流数据系统时，首先写入流表，然后写入持久化队列和发送队列（假设用户设置为异步持久化），持久化队列异步写入磁盘，发送队列发送到订阅端。  
当订阅端收到数据后，先放入接受队列，然后用户定义的handler从接收队列中取数据并处理。如果handler处理缓慢，会导致接收队列有数据堆积，占用内存。如下图所示：
![image](https://github.com/myspf/tutorial/blob/master/streaming.png)


队列的深度可以通参数来设置（见6.1节），运行过程，可以通过函数 getStreamingStat() 来查看流表的大小以及各个队列的深度。  

## 5 为写入DFS数据库提供缓存

DolphinDB为了提高写入的吞吐量，采用先缓存写入的数据，等到一定数量时，批量写入。这样减少和磁盘文件的交互次数，提升写入性能，可提升30%+的写入速度。因此，也需要一定的内存空间来临时缓存这些数据，如下图所示：

![image](https://github.com/myspf/tutorial/blob/master/cacheEngine.png)

当事务t1，t2，t3都完成时，将三个事务的数据一次性写入到DFS的数据库磁盘上。Cache Engine空间一般推荐为maxMemSize的1/8~1/4,可根据最大内存和写入数据量适当调整。

## 6 内存相关配置选项
DolphinDB提供了一些内存相关的配置选项，如下：

*  __maxMemSize__ : DolphinDB 实例使用的最大内存量，应该根据系统实际的物理内存，以及节点的个数来合理的设置该值。设置的越大，性能以及系统的处理能力越大，但如果设置值超粗了系统提供的内存大小，则有可能会被操作系统杀掉。比如操作系统实际的物理内存是16G，该选项设置为32G，那么运行过程中，有可能被操作系统默默杀掉。

*  __maxPersistenceQueueDepth__ : 流表持久化队列的最大消息数。对于异步持久化的发布流表，先将数据放到持久化队列中，再异步持久化到磁盘上。该选项默认设置为1000万。在磁盘写入成为瓶颈时，队列会堆积数据。

*  __maxPubQueueDepthPerSite__ : 最大消息发布队列深度。针对某个订阅节点，发布节点建立一个消息发布队列，该队列中的消息发送到订阅端。默认值为1000万，当网络出现拥塞时，该发送队列会堆积数据。

*  __maxSubQueueDepth__ : 订阅节点上最大的每个订阅线程最大的可接收消息的队列深度。订阅的消息，会先放入订阅消息队列。默认设置为1000万，当handler处理速度较慢，不能及时处理订阅到的消息时，该队列会有数据堆积。

*  __chunkCacheEngineMemSize__ : 指定cache engine的容量。cache engine开启后，写入数据时，系统会先把数据写入缓存，当缓存中的数据量达到chunkCacheEngineMemSize的30%时，才会写入磁盘。

*  __流表的capacity__ ：在函数enableTablePersistence()中第四个参数指定，该值表明流表中保存在内存中的最大行数，达到该值是，从内存中删除一半数据。当流数据节点中，流表比较多时，要整体合理设置该值，防止内存不足。

## 7 高效使用内存编程原则
在企业的生产环境中，DolphinDB往往作为流数据中心以及历史数据仓库，给业务人员提供数据查询和计算。当用户较多时，不当的使用容易造成Server端内存耗尽，抛出“out of memory” 异常。可遵循以下建议，尽量避免内存的不合理使用。

*  __合理均匀分区__ , DolphinDB是以分区为单位加载数据，因此，分区大小对内存影响巨大。合理均匀的分区，不管对内存使用还是对性能而言，都是有积极的作用。因此，在创建数据库的时候，根据数据规模，合理规划分区大小。每个分区的常用字段大小约100左右为宜。
   
*  __用户尽量避免持有大量数据的变量__, 当用户数据量较大的变量，比如 ```v = 1..10000000```,或者把查询结果赋值给一个变量 ```t = select * from t where date = 2010.01.01```，那么v和t将会在用户的session占用大量的内存，如果不及时释放，当其他用户申请内存时，就有可能因为内存不足而抛出异常。

*  __只查询需要的列，避免使用select \*__,如果用select \*则会把该分区所有列加载到内存，实际中，往往只需要几列，因此为避免内存浪费，尽量明确写出所有查询的列，而不是用\*代替。

*  __数据查询尽可能的加分区过滤条件__,DolphinDB按照分区进行数据检索，如果不加分区过滤条件，则会全部扫描所有数据，内存很快被耗尽。有多个过滤条件的话，select语句要优先写分区的过滤条件。

*  __用完的变量或者session，尽快释放__,根据上面的分析可以知道，用户的私有变量在创建的session里面保存，并且session关闭的时候，会回收这些内存。因此，及早释放这些私有变量（undef）或者关闭session来释放内存。

*  __合理配置流数据的缓存区__,一般情况下流数据的容量capacity会直接影响发布节点的内存占用。比如，capacity设置1000万条，那么流数据表在超过1000万条时，会回收约一半的内存占用，也就是内存中会保留500万条左右。因此，根据发布节点的最大内存，合理的流表的capacity，尤其是在多张发布表的情况，更需要谨慎设计。

## 8 内存管理相关函数

__clearAllCache()__ ，清除分区表的缓存。  
__mem([freeUnusedBlocks=false])__ ，查看内存使用情况，如果freeUnusedBlocks=true，系统将会释放未使用的内存块。  
__undef(obj, [objType=VAR])__ ，从内存中释放变量和函数定义。objType: 需要取消定义的对象的类型，VAR（本地变量）,SHARED（共享变量）。  
__getSessionMemoryStat()__ ，返回当前节点的所有会话的内存使用信息。由于共享表和分布式表不属于某个用户或会话，因此返回结果中不包含它们占用的内存信息。  


