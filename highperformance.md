# DolphinDB 高性能集群配置

DolphinDB是一个分布式数据库系统，提供了一系列配置选项，方便用户进行配置，以充分利用机器硬件资源（包括CPU、内存、磁盘、网络）。合理的参数配置，使系统均衡合理的使用这些资源，最大化发挥机器的性能。

DolphinDB提供了很多选项来配置集群使用的硬件能力，比如计算能力的最大并发度、可使用内存的最大量、最大网络链接数、流数据发送队列的最大深度等，从硬件能力的角度分类，所有选项如下：

- CPU：workerNum、localExecutors、maxBatchJobWorker、maxDynamicWorker、webWorkerNum、persistenceWorkerNum、subExecutors
- 内存：maxMemSize、maxPersistenceQueueDepth、maxPubQueueDepthPerSite、maxSubQueueDepth
- 磁盘：volumes、diskIOParallelLevel、dfsReplicationFactor、persistenceDir
- 网络：maxConnections、maxPubConnections、_maxSubConnections、maxConnectionPerSite、tcpNoDelay

除了硬件资源，操作系统层面也对进程的资源使用进行了限制，比如允许进程打开的最大文件数，这些操作系统层面配置，本文暂不讨论。DolphinDB集群的搭建请参考教程 [https://github.com/dolphindb/Tutorials_EN/blob/master/single_machine_cluster_deploy.md]()

### 1. 系统性能综述

DolphinDB 是一个功能强大，支持多用户的分布式数据库系统，同时集成了流数据，作业处理，分布式文件系统等功能，支持海量数据的高并发读写，流式计算。整个系统采用多线程架构，数据库文件的存储支持多种类型的分区。DolphinDB提供了很多选项来合理化配置集群，以最大化发挥机器的性能。DolphinDB选项和和硬件资源关系概述如下：

**CPU**  为系统提供计算能力，DolphinDB 每个数据节点都可以做为客户端，来接受用户请求，进行计算，如果需要远端数据，同时会把任务发给远端的数据节点。在大数据系统里，往往用户的一个任务会被系统分解成非常多的小任务，这些小任务通常都可以并发执行。 **workerNum** 和 **localExecutors** 来完成任务的分解和执行。因此这两个参数会直接决定系统的并发度。
同时，还引入了其他参数配置其他的场景，比如http请求并发度 **webWorkerNum** ，作业任务并发度 **maxBatchJobWorker** 。针对流数据场景还配置了 **persistenceWorkerNum**, **subExecutors** 来配置发布和订阅的并发能力。

__内存__  是现代计算机提升性能的关键，为了充分的利用内存，DolphinDB自己管理内存和cache。参数 __maxMemSize__ 指明了节点可用的最大内存，该参数越大系统的性能越高。当然，如果该参数设置的内存超出操作系统可提供的最大内存，那么会导致DolphinDB被操作系统杀死。DolphinDB提供了很多队列深度相关选项，这些选项结合节点最大可用内存量(**maxMemSize**)来配置，配置的过大，可能超出系统提供的内存范围，配置过小，会限制系统的整体性能。

__磁盘__  存储数据文件，数据库系统频繁的从磁盘加载数据，并不定期的写入数据，磁盘I/O往往成为性能的瓶颈，如果多个线程同时访问同一个磁盘卷，那么显而易见，性能会非常糟糕，如果系统有多个磁盘卷，DolphinDB也提供了配置选项来同时利用多个磁盘的读写能力。选项 __volumes__ 指明节点存储数据表可以使用的磁盘卷， __diskIOParallelLevel__ 指定了系统可以并行读写磁盘的能力。
对流数据的持久化，选项 __persistenceDir__ 指定流数据的持久化目录，如果系统中有多个节点作为流数据发布中心，该选项可以设置为不同的磁盘卷。以增加系统并发写入能力。

__网络__  提供集群之间通信，对于分布式系统，网络本身的时延和吞吐量对系统性能影响极大，网络主要是由基础硬件决定的。DolphinDB提供了一些对连接数控制的选项，来合理的控制系统资源消耗，比如 **maxConnections**、 **maxConnectionPerSite**、 **maxPubConnections**、 **maxSubConnections**。同时为了提高系统响应速度，提供 __tcpNoDelay__ 选项。

典型的DolphinDB的集群场景包括：作为分布式时序数据库，提供入库和查询功能；作为稳定的流数据发布中心，提供来自各种应用的订阅服务；

### 2. 分布式时序数据库高性能配置

作为时序数据库系统，是DolphinDB最典型的应用场景，下面按照CPU、内存、磁盘、网络等硬件资源介绍DolphinDB相关的选项。

#### 2.1 CPU配置选项

DolphinDB架构采用多线程技术，合理的并发度能极大提升系统性能。并发度太低，不利用系统使用硬件的多线程能力，并发度太高，容易导致过多的线程切换，造成总体性能降低。影响并发度的主要参数如下:

__workerNum__  : woker负责接收客户端请求，分解任务，根据任务粒度自己执行或者交给excutor执行。该参数直接决定了系统的并发数，根据物理机器上逻辑线程数，以及该物理机器bak的DolphinDB数据节点个数设置。

__localExecutors__ : localExcutor负责执行woker分解的任务。和workNum类似，直接决定了系统的并发度，推荐设置为 workerNum - 1。  

__maxBatchJobWorker__  : batchJob Worker 执行批处理任务，这些任务是指通过submitJob函数提交的任务，通常耗时较长。该参数决定了执行批处理任务的并发度。根据系统执行批处理任务的多少群确定。 
注意：如果没有批处理任务，创建的线程会回收，所以并不会占用系统资源。

__maxDynamicWorker__ : dynamic worker作为上面介绍的worker的补充，当所有的worker线程占满后，如果新的任务进来，则会创建dynamic worker来执行，并且用完后会释放资源。

__webWorkerNum__ : web worker处理http请求，表示处理http请求的线程数目。web的连接可以通过集群管理器界面，连接到某个节点的notebook进行交互式操作。

上面的选项针对系统的整体并发计算能力。同时，DolphinDB也提供了通过函数来设置某个用户执行任务的最大优先级和并行度，具有较高由优先级的任务会有更多的机会和资源执行，从应用的层面来提高某些任务的计算并发度和优先级。

__setMaxJobPriority__ : 优先级范围是0-8，高优先级可以获取更多的执行时间和机会。任务的默认优先级为 4。
__setMaxJobParallelism__ : 并行度范围是0-64，并行度代表可以并发度，高并行度可以好的利用机器的多核资源，并发执行任务。任务的并行度默认位2。

#### 2.2 内存配置选项

__maxMemSize__ : DolphinDB 实例使用的最大内存量，应该根据系统实际的物理内存，以及节点的个数来合理的设置该值。设置的越大，性能以及系统的处理能力越大，但如果设置值超粗了系统提供的内存大小，则有可能会被操作系统杀掉。比如操作系统实际的物理内存是16G，该选项设置为32G，那么运行过程中，有可能被操作系统默默杀掉。

#### 2.3 磁盘配置选项

__volumes__ : 分布式数据库存储分区数据的位置，如果系统有多个volume，建议每个节点配置成不同的volume，这样DolphinDB从系统读写数据，可以并行的利用多个磁盘的I/O接口，大大提升读写数据的速度。

__diskIOParallelLevel__ : 磁盘I/O并行参数，该值可以设置为每个节点可以使用的磁盘volumes的数量。

__dfsReplicationFactor__ : 分布式数据库的副本数，该值对系统性能和存储空间都有较大影响。推荐设置为2，一方面保证数据的高可用，另一方面两个副本，可以起到负载均衡的作用，读数据时从负载较低的节点读取，提高系统整体性能。如果磁盘存储空间非常有限，不能提供2副本的空间，可以设置为1。

#### 2.4 网络配置选项

__maxConnections__ : 可接受的最大连接数。DolphinDB每个实例，收到用户请求，建立一个连接，来完成用户任务。该选项越大，可以同时接受处理外部请求越多。该选项默认为 32 + maxConnectionPerSite * 节点数

__maxConnectionPerSite__ : 对外某一个节点的最大连接数。DolphinDB数据是分布式存储的，每个节点需要和其他节点进行连接通信，该选项指明该节点能连接和任一节点能建立的最大连接数。默认是 localExecutors + workerNum + webWorkerNum ，推荐使用默认设置。

__tcpNoDelay__ : 使能TCP的 TCP_NODELAY 选项，可以有效的降低请求的时延。推荐设置为true。

> __注意__ : 数据库的分区设计对查询的性能影响很大。分区过大，会造成并行加载容易出现内存不足，从而造成操作系统频繁对内存和磁盘进行数据交换，大大降低降低性能。分区过小，造成系统中存在大量的子任务，导致节点间产生大量的通信和调度，并且还会频繁的访问磁盘的小文件，也会明显降低性能。关于分区的大小以及详细的设计，请参考教程  [https://github.com/dolphindb/Tutorials_CN/blob/master/database.md]()

### 3. 流计算模块高性能配置  

流数据作为一个较为独立的功能模块，有些配置选项专门为流计算设计，大部分场景默认配置能满足要求。如果用户对流数据的性能要求高，需要一些定制化配置，可以参考如下配置选项。

#### 3.1 发布节点配置选项  

__persistenceWorkerNum__ : 异步持久化模式下，负责将持久化数据写到磁盘上的线程数。
数据发布表持久化的线程数。如果启用了持久化功能，则会将发布的数据同步或者异步的方式持久化到磁盘上。函数enableTablePersistence的第二参数指定了是同步还是异步的方式持久化。同步保证消息不会丢失，而异步方式则会大大提升系统的吞吐量。因此，如果应用场景能容忍发布节点异常宕机时丢失最后几条消息，建议设置为异步持久化。

__persistenceDir__ : 流数据表持久化数据保存的目录。一个物理server中的多个节点发布流表，每个节点的该选项应该设为不同的磁盘卷。这样可以并行的利用多个磁盘的IO写入能力。

__maxPersistenceQueueDepth__ : 流表持久化队列的最大消息数。对于异步持久化的发布流表，先异步的将数据放到持久化队列中，然后负责持久化的线程（persistenceWorkerNum设置持久化线程数）不断的讲队列中的消息持久化到磁盘上。该选项指明了，该流表持久化队列中的最大消息条数。默认设置为1000000。

__maxPubQueueDepthPerSite__ : 最大消息发布队列深度。针对某个订阅节点，发布节点建立一个消息发布队列，该队列中的消息发送到订阅端。该选项指明，该发布节点针对某个订阅节点的消息发布队列的最大深度。默认值为10000000。

__maxPubConnections__ : 最大的发布-订阅连接数。一个订阅节点和一个发布节点之间（可能订阅该节点的多张流表）建立1个发布-订阅连接。该选项指定能够订阅该发布节点上表的最大订阅节点的个数。例如设置为８，则最多有８个订阅节点来订阅该发布节点上的流表。

> __注意__ : 发布表的持久化函数 enableTablePersistence(table, [asynWrite=true], [compress=true], [cacheSize=-1]) 中几个参数的合理设置，对性能影响很大。
> asynWrite : 流数据是否异步持久化，显然异步持久化会大大提升系统性能，代价是宕机的时候，可能会造成最后几条消息丢失。可容忍的场景下，建议设为true。  
> compress : 持久化到磁盘的数据是否压缩。如果压缩，数据量很降低很多，同时减少磁盘写入量，提升性能，但压缩解压有一定代价，建议设为true。  
> cacheSize : 流表中保存在内存中数据的最大条数，越大的话，对实时查询，订阅都有性能提升。根据物理机内存总量，合理分配。建议大于1000000。

#### 3.2 订阅节点配置选项

__subExecutors__ : 订阅节点处理流数据的线程数。每个流表只能在1个executor上处理，所以根据该节点上订阅流表的多少，来设置该值。另外，一个订阅只能在一个线程上执行，一个线程也可以执行多个订阅。默认值为 1。

__maxSubQueueDepth__ : 订阅节点上最大的每个订阅线程最大的可接收消息的队列深度。订阅的消息，会先放入订阅消息队列，该值指明该队列的大小。默认设置为10000000。

__maxSubConnections__ : 订阅节点的最大订阅连接数。如果一个节点订阅同一个发布节点上的多张流表，那么这些流表共享一个连接。所以，一个订阅DolphinDB节点、API订阅应用等都是一个订阅连接。默认为 64.

> __注意__ : 订阅函数 subscribeTable([server], tableName, [actionName], [offset=-1], handler, [msgAsTable=false], [batchSize=0], [throttle=1], [hash=-1])，有几个参数对性能影响很大。
> batchSize : 触发handler处理消息的累计消息量（行数）阈值。根据流数据的频率，建议设置该值，批处理会很大的提升性能。  
> throttle : 触发handler处理消息的时间（秒）阈值。如果batchSize也设置，那么哪个先满足条件，都会触发handler计算。  
> handler : 处理流数据的函数。该函数里面应该高度优化，尽量采用向量化编程。因为会多次调用，整体性能影响很大。

### 4. 典型服务器配置实例

#### 4.1 分布式时序数据库配置

物理服务配置：24core，48线程；内存256G；磁盘1.8T\*12。作为历史数据仓库，提供数据入库以及查询功能。

- 集群设计  
  集群包括1个controller，每台物理机1个agent，以及多个datanode。
  datanode个数太少的话，不能充分利用系统并发能力，个数太多的话，管理不方便，而且每个datanode分配的内存就会很少，限制了单datanode的计算能力；也容易造成线程竞争，反而降低了系统的性能。一般来说对于一台物理机器，建议datanode个数在4-8之间为宜。本例采用6节点。
  这样集群包括1个controller，1个agent，6个datanode。
- 内存设置  
  maxMemSize : 每个数据节点分30G，controller分30G，操作系统以及其他进程保留50G左右；
- 线程设置  
  workerNum: 共48线程，每个datanode和controller可以设计 8 worker，这样共需要56线程，考虑到实际中，很少有任务能把系统所有节点都跑满，所以这样设计是合理的。
  localExecutors: 通常来说，localExecutors 设置为 workerNum - 1 位最优，是因为worker接受到任务，分配给exector，worker本身也可以执行该任务。
- 磁盘设置  
  volumes: 12个Volumne可以分给不同的数据节点，这样大大提升系统的并行IO读写能力。每个数据节点分两个volumn来存储数据，如果系统volume跟节点个数不是倍数关系，可以某个节点少一个volume。
  dfsReplicationFactor: 设置为2，既能保证数据的高可用，又能提供查询的负载均衡。
  diskIOParallelLevel: 设置为2，每个节点2个volumes。
- 网络设置  
  tcpNoDelay : 设置为true，提高系统响应速度。
  maxConnections : 该节点支持的最大接入进来的连接数，一般根据接入客户端的多少来设置，比如GUI、web、api等都是独立的连接，本例设置为64。

#### 4.2 流计算配置

基于稳定性考虑，生产环境的流数据中心一般与历史数据库分开部署，部署到独立的物理机器上（集群或者单节点方式部署）。下面以金融领域为例，介绍一种典型的部署场景。

物理服务配置：8core，16线程；内存64G；磁盘1.0T\*4；流数据表包括： 期货、期权、股票指数、逐笔委托、股票、逐笔成交，前面三张数据量比较小，每秒几百条，后面三张每秒1万条左右。

- 集群设计  
  整体的数据量不大，实际上一个节点就完全可以处理，考虑到以后的扩展性，以及最大化DolphinDB的性能，我们搭建一个发布集群来处理。
  包括4个节点，每个节点内存分12G。其中三个节点上每个节点处理两张表，按照数据量均匀的原则，节点1处理期货和逐笔委托，以此类推。
- 每个节点的参数配置
  persistenceWorkerNum : 集群共有4个节点，物理机器包括16逻辑线程，因此参数设为4。
  maxPubConnections : 根据订阅客户端的多少设置，假设设为16。
  maxPersistenceQueueDepth : 按照每行200字节估算，500万行大概1G，所以该参数可以设置为 10000000，共2G。
  maxPubQueueDepthPerSite : 设置为3000000。最极端的情况，16个订阅端，共0.6G\*16 = 9.6G。系统内存也满足要求。
  persistenceDir : 每个节点设置为一个单独的磁盘卷。这样持久化的时候，可以并发写入。

订阅端可以是各种API客户端，或者DolphinDB server。
如果是DolphinDB server作为订阅客户端， 性能相关的主要配置下这三个参数 subExecutors、maxSubQueueDepth、maxSubConnections，根据实际情况进行合理配置。subExecutors 根据订阅表的多少配置，其他连个参数可以采用默认值。

### 5. 性能监控  

DolphinDB提供了各种工具来监控集群的性能。包括controller和各个数据节点的硬件性能、查询性能、流数据性能等指标。方便用户实时监控系统的资源和性能。

#### 5.1 集群管理器

集群管理器可以监控到30多个性能指标。比较常用的包括各个节点的 cpu利用率、平均负载、内存使用量、连接数、query查询统计、任务队列深度、磁盘写入速度、磁盘读取速度、网络接收速率、网络发送速率等。相关指标如下：

__CPU使用率__  : CpuUsage、AvgLoad

__内存监控__   : MemUsed、MemAlloc、MemLimit

__磁盘监控__   : DiskCapacity、DiskFreeSpaceRatio、DiskWriteRate、DiskReadRate、LastMinuteWriteVolume、LastMinuteReadVolume

__网络监控__  : networkSendRate、networkRecvRate	lastMinuteNetworkSend	lastMinuteNetworkRecv

__实时查询性能指标__  : medLast10QueryTime、maxLast10QueryTime、medLast100QueryTime、maxLast100QueryTime20、maxRunningQueryTime、runningJobs、queuedJobs、runningTasks、queuedTasks、jobLoad

__实时流数据性能指标__  : lastMsgLatency、cumMsgLatency

这些指标也可以通过函数 getClusterPerf() 以table的形式获取到。通过这些监控指标可以反映出整个集群的性能情况。
比如cpu过高，平均负载过大，说明cpu可能成为集群性能瓶颈；如果磁盘读取基本达到极限，说明IO限制了整体的性能；然后可以再根据上面的指标，对瓶颈点进行配置调优。

#### 5.2 流数据性能监控

使用getStreamingStat()函数可以获取到发布端和订阅端的流数据性能指标。该函数返回一个dictionary，包括了对发布和订阅端的性能指标。

##### 5.2.1 发布端性能指标

pubTables : 该节点所有的发布连接，以及每个连接当前发布的 msgOffset，每行代表一个topic(订阅节点，发布节点，订阅表名，订阅action名）。该msgOffset是指该发布表在这个topic的当前发布位置。

pubConns : 与pubTable相似，区别是每行代表一个client端的所有订阅，即这个client在该发布节点订阅的所有表。通过queueDepth可以观察消息的积累程度，如果该值很大，说明发布的队列消息积累严重。可能网络传输慢，发送数据量太大，或者消息被订阅端消费的太慢。

##### 5.2.1 订阅端性能指标

subConns : 统计发布端到订阅端消息的时延等信息。每行代表一个发布端，统计指标包括某个发布端向该订阅端，发布的消息总数，累计的时延，最后一条消息的时延。如果时延太长，有可能是不在同一台物理机器上，两个机器的时间有差异；还有可能是网络本身时延较高；

subWorkers : 每个subWorker处理消息的统计信息。subWorker的个数由上面subExecutors指定，统计指标包括累积消息的队列深度，已经处理的消息个数等。如果queueDepth 太大，或者达到 queueDepthLimit 值，说明该订阅消费消息太慢，导致消息积累。需要优化消息处理的handler，以更快的处理消息。

#### 5.3 用户内存使用性能监控

函数getSessionMemoryStat()返回该节点的不同用户的内存使用总量。如果碰到内存超限，可以使用该函数，排查哪个用户占用过多内存。
