# DolphinDB 数据同步
DolphinDB提供离线方式和在线方式实现不同集群间数据库的同步。 
* __离线方式__ ，通过数据库备份和恢复功能实现数据同步。
* __在线方式__ ，通过建立在线连接，把数据从一个库读出来再写入到另一个库中。
数据库指DFS分布式的数据库，而非内存表或流数据表等

## 1. 离线方式
离线方式是先把数据中的各个表，通过DolphinDB提供的backup函数导入到磁盘上，然后将数据同步的到数据库所在的物理机器上，再通过restore函数把将数据从磁盘上恢复到到数据库中。如下所示：
![image](https://github.com/myspf/tutorial/blob/master/Selection_387.png) 

### 1.1 同步流程
* __数据备份__ ，可通过backup函数将需要同步的数据表备份到磁盘上，同时可以制定同步的时间范围，比如当天或者某段时间，
```
t = loadTable(dbName,tableName)		
backup(backupDir,sql(sqlCol("*"), t , expr(sqlCol("day"),eq,2019.01.01)))
```
上面，t为所要复制的分布式表。backup函数把分布式表中符合条件的数据备份到backupDir目录中。复制数据的条件通过sql元代码指定，上面例子中复制数据库t中列day为2019.01.01的一天的数据。

* __节点间数据文件同步__ ，DolphinDB支持shell命令，可利用操作系统提供的文件同步手段来同步目录，比如rsync或者scp命令。其中rsync是linux上的常用命令，只同步发生变化的文件，非常高效。
```
cmd = "rsync -av  " + backupDir + "/*  " + userName + "@" + restoreIP + ":" + restoreDir 
shell(cmd)
```
如上，把一台机器上backupDir目录下的所有发生变化的文件同步到另一条机器的restoreDir目录下。注意:自动同步需要配置ssh免密登录。

* __数据恢复__ ，数据同步过来以后，可以从restoreDir中恢复出所需要的数据，通过DolphinDB提供的restore函数
```
restore(restoreDir,dbName,tableName,"%",true,loadTable(dbName,tableName))
```
如上，把restore下面的所有数据导入到数据库中，完成备份。

## 2 在线方式
在线方式，要求两个集群同时在线，通过建立socket连接，直接从一个集群中读数据，并写入到另一个集群上。如下
![image](https://github.com/myspf/tutorial/blob/master/Selection_388.png) 

代码如下：
```
def writeData(dbName,tableName,t){
	login(`admin,`123456)
	loadTable(dbName,tableName).append!(t)
}
def writeRemoteDB(t,remoteIP,remotePort,dbName,tableName){
	conn = xdb(remoteIP,remotePort)
	remoteRun(conn,writeData,dbName,tableName,t)
}
t = loadTable(dbName,tableName)
sd = sql(sqlCol("*"),<t>, expr(sqlCol("day"),eq,2019.01.01))
datasrc = repartitionDS(sd, "sym", RANGE, 10)
mr(ds = datasrc, mapFunc = writeRemoteDB{,remoteIP,remotePort,dbName,tableName}, parallel = false)

```
在线同步的一个关键问题是，如何避免一次拷贝的数据量太大，导致内存不够。这里我们采用的方法是，通过repartitionDS函数，将一天数据再进行切分，切分成多个数据块后，再进行数据同步，避免内存不足。

### 3 两种方式的比较
离线方式，要求要有额外的磁盘空间存储同步的数据，并且数据转移3次（先备份到磁盘，在转移到另一台机器，再导入），性能相较同步的方式会差。优势是，如果数据库多种多样，分区错综复杂，那么这种方式使用方便，几乎不用考虑分区和内存，因此，backup函数自动按照分区进行进行操作。

在线方式，要求两个集群必须同时能够提供服务，不占用磁盘空间，但要对数据库进行合理的划分，避免内存不足。