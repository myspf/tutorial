# DolphinDB 数据同步
DolphinDB提供离线方式和在线方式实现不同集群间数据库的同步。 
* __离线方式__ ，通过数据库备份和恢复功能实现数据同步。
* __在线方式__ ，通过建立在线连接，把数据从一个库读出来再写入到另一个库中。
数据库指DFS分布式的数据库，而非内存表或流数据表等

## 1. 离线方式
离线方式是先把数据库中的各个表，通过DolphinDB提供的backup函数以二进制形式导入到磁盘上，然后将数据同步的到数据库所在的物理机器上，再通过restore函数把将数据从磁盘上恢复到到数据库中。如下所示：
![image](https://github.com/myspf/tutorial/blob/master/Selection_387.png) 

### 1.1 同步流程
* __数据备份__ ，可通过backup函数将需要同步的数据表备份到磁盘上，备份是以分区为单位。需要同步的数据可以用sql语句指定，如下：
示例1，备份数据库(db1)中表(mt)的所有数据：
```
backupDir = /hdd/hdd1/backDir		
backup(backupDir,<select * from loadTable("dfs://db1","mt")>)
```

示例2，备份数据库(db1)中表(mt)的最近7天的数据，假设时间分区字段是TradingDay(DATE):
```
backupDir = /hdd/hdd1/backDir		
backup(backupDir,<select * from loadTable("dfs://db1","mt") where TradingDay > date(now()) - 7 and  TradingDay <= date(now())>)
```

示例3，备份数据库(db1)中表(mt)的某些列(col1,col2,col3)的数据：
```
backupDir = /hdd/hdd1/backDir		
backup(backupDir,<select col1,col2,col3 from loadTable("dfs://db1","mt")>)
```
更灵活的sql元语句表示参考DolphinDB元编程。

* __节点间数据文件同步__ ，如果需要同步的两个数据库不在同一台物理机器上，则需要同步二进制文件。DolphinDB支持shell命令，可利用操作系统提供的文件同步手段来同步目录，比如rsync或者scp命令。其中rsync是linux上的常用命令，只同步发生变化的文件，非常高效。
```
cmd = "rsync -av  " + backupDir + "/*  " + userName + "@" + restoreIP + ":" + restoreDir 
shell(cmd)
```
如上，把一台机器上backupDir目录下的所有发生变化的文件同步到另一条机器的restoreDir目录下。其中，userName和restoreIP是通过ssh登录的用户名以及远程机器的ip地址。
注意:以上命令需要配置ssh免密登录。当然，也可以通过其他服务器同步工具实现。

* __数据恢复__ ，数据同步过来以后，可以从restoreDir中恢复出所需要的数据，通过DolphinDB提供的restore函数
示例1，恢复所有备份数据库(db1)表(mt)的所有数据到数据库(db2)的表(mt)中：
```
restore(restoreDir,"dfs://db1","mt","%",true,loadTable("dfs://db2","mt"))
```
除了恢复所有数据，还可以根据条件恢复指定分区。详细参考教程考[数据备份与恢复](https://github.com/dolphindb/Tutorials_CN/blob/master/restore-backup.md)。

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
