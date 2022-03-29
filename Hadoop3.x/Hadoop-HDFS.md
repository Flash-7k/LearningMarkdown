# Hadoop-HDFS

## 第1章 HDFS概述

### 1.1 HDFS产出背景及定义

+ **HDFS产生背景**

  一个操作系统中存不下所有数据，只能分配到更多操作系统管理的磁盘中，但是不方便管理和维护。因此需要**一种系统来管理多台机器上的文件**，也就是**分布式文件管理系统**，HDFS只是其中一种

+ **HDFS定义**

  + **Hadoop Distributed File System**
  + **文件系统**：存储文件，通过目录树来定位文件
  + **分布式**：许多服务器联合起来

+ **HDFS使用场景**

  + **一次写入，多次读出**

### 1.2 HDFS优缺点

1. **优点**
   + **高容错性**
     + 数据自动保存多个副本。
     + 某一个副本丢失后，可以自动恢复
   + **适合处理大数据**
     + **数据规模**：GB、TB、PB
     + **文件规模**：百万规模以上
   + **可构建在廉价机器上**
     + 多副本机制，提高可靠性
2. **缺点**
   + **不适合低延时数据访问**
     + 如做不到毫秒级的存储数据
   + **无法高效的对大量小文件进行存储**
     + 存储大量小文件，占用NameNode大量内存来存储文件目录和块信息
     + 存储大量小文件，寻址时间会超过读取时间，违法HDFS的设计目标
   + **不支持并发写入、文件随机修改**
     + 一个文件只能有一个写入，不允许多线程同时写
     + 仅支持数据append，不支持文件的随机修改

### 1.3 HDFS组成架构

1. **NameNode（nn）**
   + 管理HDFS的名称空间
   + 配置副本策略
   + 管理数据块（Block）映射信息
   + 处理客户端读写请求
2. **DataNode（dn）**
   + 存储实际的数据块
   + 执行数据块的读写操作
3. **Client**
   + 文件切分：文件上传HDFS时，将文件切分成一个个Block，然后上传
   + 与NameNode交互，获取文件的位置信息
   + 与DataNode交互，读取或写入数据
   + 提供一些命令管理HDFS：如NameNode格式化
   + 通过一些命令访问HDFS：如CRUD
4. **SecondaryNameNode：并非NameNode的热备。当NameNode挂掉时，并不能马上替换NameNode并提供服务**
   + 辅助NameNode，分担其工作量：如定期合并Fsimage和Edits，并推送给NameNode
   + 紧急情况下，可辅助恢复NameNode

### 1.4 HDFS文件块大小（面试重点）

**默认大小：Hadoop1.x  64M，Hadoop2.x/3.x  128M**

+ 若寻址时间约为10ms，由”寻址时间为传输时间的1%时，为最佳状态“
+ 得，传输时间=10ms/0.01=1000ms=1s
+ 目前磁盘的传输速率普遍为100MB/s
+ 因此Block默认大小128M

**思考：为什么块大小不能太大也不能太小？**

+ 太小：增加寻址时间
+ 太大：存取时间明显大于寻址时间

**总结：HDFS块的大小设置主要取决于磁盘传输速率**

## 第2章 HDFS的Shell操作（开发重点）

### 2.1 命令大全

```linux
[flash7k@hadoop102 hadoop-3.1.3]$ bin/hadoop fs

[-appendToFile <localsrc> ... <dst>]
        [-cat [-ignoreCrc] <src> ...]
        [-chgrp [-R] GROUP PATH...]
        [-chmod [-R] <MODE[,MODE]... | OCTALMODE> PATH...]
        [-chown [-R] [OWNER][:[GROUP]] PATH...]
        [-copyFromLocal [-f] [-p] <localsrc> ... <dst>]
        [-copyToLocal [-p] [-ignoreCrc] [-crc] <src> ... <localdst>]
        [-count [-q] <path> ...]
        [-cp [-f] [-p] <src> ... <dst>]
        [-df [-h] [<path> ...]]
        [-du [-s] [-h] <path> ...]
        [-get [-p] [-ignoreCrc] [-crc] <src> ... <localdst>]
        [-getmerge [-nl] <src> <localdst>]
        [-help [cmd ...]]
        [-ls [-d] [-h] [-R] [<path> ...]]
        [-mkdir [-p] <path> ...]
        [-moveFromLocal <localsrc> ... <dst>]
        [-moveToLocal <src> <localdst>]
        [-mv <src> ... <dst>]
        [-put [-f] [-p] <localsrc> ... <dst>]
        [-rm [-f] [-r|-R] [-skipTrash] <src> ...]
        [-rmdir [--ignore-fail-on-non-empty] <dir> ...]
<acl_spec> <path>]]
        [-setrep [-R] [-w] <rep> <path> ...]
        [-stat [format] <path> ...]
        [-tail [-f] <file>]
        [-test -[defsz] <path>]
        [-text [-ignoreCrc] <src> ...]
```

### 2.2 常用命令

+ **-help : 输出这个命令参数**

```linux
[flash7k@hadoop102 hadoop-3.1.3]$ hadoop fs -help rm
```

+ **上传**

  + **语法：hadoop fs 命令 本地文件路径 HDFS文件路径**
  + **-moveFromLocal：从本地 剪切 粘贴到HDFS**

  + **-copyFromLocal：从本地文件系统中 拷贝 文件到HDFS路径去**
  + **-put：等同于copyFromLocal，生产环境更习惯用put**
  + **-appendToFile：追加一个文件到已经存在的文件末尾**

+ **下载**

  + **语法：hadoop fs 命令 HDFS文件路径 本地文件路径**
  + **-copyToLocal：从HDFS拷贝到本地**
  + **-get：等同于copyToLocal，生产环境更习惯用get**

+ **HDFS直接操作**

  + **-ls: 显示目录信息**

  ```linux
  [flash7k@hadoop102 hadoop-3.1.3]$ hadoop fs -ls /sanguo
  ```

  + **-cat：显示文件内容**

  ```linux
  [flash7k@hadoop102 hadoop-3.1.3]$ hadoop fs -cat /sanguo/shuguo.txt
  ```

  + **-chgrp、-chmod、-chown：Linux文件系统中的用法一样，修改文件所属权限**

  ```linux
  [flash7k@hadoop102 hadoop-3.1.3]$ hadoop fs  -chmod 666  /sanguo/shuguo.txt
  [flash7k@hadoop102 hadoop-3.1.3]$ hadoop fs  -chown  atguigu:atguigu   /sanguo/shuguo.txt
  ```

  + **-mkdir：创建路径**

  ```linux
  [flash7k@hadoop102 hadoop-3.1.3]$ hadoop fs -mkdir /jinguo
  ```

  + **-cp：从HDFS的一个路径拷贝到HDFS的另一个路径**

  ```linux
  [flash7k@hadoop102 hadoop-3.1.3]$ hadoop fs -cp /sanguo/shuguo.txt /jinguo
  ```

  + **-mv：在HDFS目录中移动文件**

  ```linux
  [flash7k@hadoop102 hadoop-3.1.3]$ hadoop fs -mv /sanguo/wuguo.txt /jinguo
  ```

  + **-tail：显示一个文件的末尾1kb的数据**

  ```linux
  [flash7k@hadoop102 hadoop-3.1.3]$ hadoop fs -tail /jinguo/shuguo.txt
  ```

  + **-rm：删除文件或文件夹**

  ```linux
  [flash7k@hadoop102 hadoop-3.1.3]$ hadoop fs -rm /sanguo/shuguo.txt
  ```

  + **-rm -r：递归删除目录及目录里面内容**

  ```linux
  [flash7k@hadoop102 hadoop-3.1.3]$ hadoop fs -rm -r /sanguo
  ```

  + **-du统计文件夹的大小信息**

  ```linux
  [flash7k@hadoop102 hadoop-3.1.3]$ hadoop fs -du -s -h /jinguo
  27  81  /jinguo
  
  [flash7k@hadoop102 hadoop-3.1.3]$ hadoop fs -du  -h /jinguo
  14  42  /jinguo/shuguo.txt
  7   21   /jinguo/weiguo.txt
  6   18   /jinguo/wuguo.tx
  ```

  + **-setrep：设置HDFS中文件的副本数量**

  ```linux
  [flash7k@hadoop102 hadoop-3.1.3]$ hadoop fs -setrep 10 /jinguo/shuguo.txt
  ```

  > 注意：设置的副本数只记录在NameNode的元数据中。
  >
  > 是否真的会有这么多副本，取决于DataNode的数量。
  >
  > 因为目前只有3台设备，最多也就3个副本，只有服务器增加到10台时，副本数才能达到10

## 第3章 HDFS的读写流程（面试重点）

### 3.1 HDFS写数据流程

1. Client 通过 Distributed FileSystem 模块向 NN 请求上传文件
2. NN 检查 Client 权限、检查目标文件是否存在、父目录是否存在
3. NN 响应 Client 是否可以上传文件
4. Client 请求上传第一个Block，询问上传到哪几个DN
5. NN 进行副本存储结果选择，返回DN1、DN2、DN3，表示采用这三个DN存储数据
   + DN1：本地节点
   + DN2：其他机架随机一个节点
   + DN3：DN2所在机架随机另一个节点
6. Client 通过 FSDataOutputStream 模块向DN1请求建立Block传输通道，DN1向DN2请求，DN2向DN3请求，江通信管道建立完成
7. DN1、DN2、DN3逐级应答 Client
8. **Client 开始传输数据**
   + 先从磁盘读取数据放到本地内存缓存，准备传输第一个Block
   + 起始有一个Chunk大小的buf，当数据写满这个buf，计算Chunksum值进行校验，然后填塞入Packet
   + 当Packet被Chunk填满后，将这个Packet放入应答队列中等待应答
   + 进入应答队列的Packet回被另一线程按序取出，发送到下一个DN
   + DN1收到一个Paceket就传给DN2，DN2传给DN3
9. 当第一个Block传输完成后，Client 再次请求NN上传第二个Block（重复3-7）

> **Block、Packet、Chunk**
>
> Block：最大的单位，是最终存储在DN上的数据粒度，由dfs.block.size参数决定，默认128M，取决于客户端配置
>
> Packet：中等的单位，是数据流向DN过程中的数据粒度，由dfs.write.packet.size参数决定，默认64k，以这个值为参考动态调整
>
> Chunk：最小的单位，是数据流向DN过程中进行数据校验的数据粒度，由io.byte.per.checksum参数决定，默认512Byte
>
> 注意：事实上Chunk还包含一个4Byte的校验值，因此Chunk写入Packet时是516Byte
>
> 数据与校验值的比例为128:1，所以一个128M的Block会有一个1M的校验文件

### 3.2 HDFS读数据流程

1. 客户端通过 DistributedFileSystem 向 NN 请求下载文件
2. NN 通过查询元数据，找到存储有目标文件块的DN地址
3. Client 挑选一台DN（就近原则、负载均衡）请求读取数据
4. DN开始传输数据给Client，从磁盘里读取数据输入流，以Packet为单位来做校验
5. Client 以Packet为单位接收，先在本地内存缓存，然后写入目标文件磁盘

## 第4章 NameNode和SecondaryNameNode

### 4.1 NN和2NN工作机制

> **思考：NN中元数据存储位置**
>
> + 若存储在磁盘：经常随机访问、响应客户请求，效率过低
>
> + 若存储在内存：一断电，元数据就丢失
>
> **因此产生Fsimage：在磁盘中备份元数据**
>
> + 若内存中元数据更新，同步更新Fsimage：效率过低
> + 若内存中元数据更新，不更新Fsimage：数据一致性问题
>
> **因此产生Edits：追加记录对内存中元数据的操作**
>
> **引入新的节点2NN：定期专门合并Fsimage和Edits，合成元数据，提高效率**

1. **第一阶段：NN启动**
   + 第一次启动，NN格式化后，创建Fsimage和Edits文件
   + 如果不是第一次启动，直接加载Edits编辑日志和Fsimage镜像文件到内存
   + 客户端对元数据进行增删改的请求
   + NN记录操作日志，更新滚动日志Edits
   + NN在内存中对元数据进行增删改
2. **第二阶段：2NN工作**
   + 2NN判断NN是否需要CheckPoint：定时时间到、Edits中数据满
   + 2NN请求执行CheckPoint
   + NN滚动正在写的Edits日志
   + 将滚动前的Edits编辑日志和Fsimage镜像文件拷贝到2NN
   + 2NN加载Edits和Fsimage，并合并生成元数据
   + 2NN生成新的镜像文件fsimage.chkpoint
   + 将fsimage.chkpoint拷贝到NN
   + NN将fsimage.chkpoint重新命名成fsimage

### 4.2 Fsimage和Edits

1. **Fsimage**

   HDFS元数据的一个永久性检查点，包含HDFS的所有文件目录和文件inode的序列化信息

2. **Edits**

   存放HDFS的所有更新操作的路径，Client执行的所有写操作首先会被记录到Edits中

3. **seen_txid**

   保持Edits最后一个数字

### 4.3 CheckPoint时间设置

1. **通常情况下，SecondaryNameNode每隔一小时执行一次**

```hdfs-default.xml
<property>
  <name>dfs.namenode.checkpoint.period</name>
  <value>3600s</value>
</property>
```

2. **一分钟检查一次操作次数，当操作次数达到1百万时，SecondaryNameNode执行一次**

```hdfs-default.xml
<property>
  <name>dfs.namenode.checkpoint.txns</name>
  <value>1000000</value>
<description>操作动作次数</description>
</property>

<property>
  <name>dfs.namenode.checkpoint.check.period</name>
  <value>60s</value>
<description> 1分钟检查一次操作次数</description>
</property>
```

## 第5章 DataNode

### 5.1 DataNode工作机制

+ Block在DN上以文件形式存储在磁盘中，包括两个文件：数据本身、数据信息（数据块的长度、块数据的校验和、时间戳）

+ DN启动后向NN注册。之后周期性（默认6小时）向NN上报所有块信息

  + DN向NN汇报信息的周期，默认6小时

  ```xml
  <property>
  	<name>dfs.blockreport.intervalMsec</name>
  	<value>21600000</value>
  	<description>Determines block reporting interval in milliseconds.</description>
  </property>
  ```

  + DN扫描本节点块信息列表的周期，默认6小时

  ```xml
  <property>
  	<name>dfs.datanode.directoryscan.interval</name>
  	<value>21600s</value>
  	<description>Interval in seconds for Datanode to scan data directories and reconcile the difference between blocks in memory and on the disk.
  	Support multiple time unit suffix(case insensitive), as described
  	in dfs.heartbeat.interval.
  	</description>
  </property>
  ```

+ 心跳默认3秒一次，返回结果待遇NN给该DN的命令。若超过10分钟没有收到DN的心跳，则认为该DN不可用

+ 集群运行中可以安全加入和退出一些机器

### 5.2 数据完整性

+ 当DN读取Block时，计算CheckSum
+ 若计算后的CheckSumh，与Block创建时的值不一样，说明Block已损坏
+ Client读取其他DN上的Block
+ 常见校验算法：crc（32），md5（128），sha1（160）
+ DN在其文件创建后周期性验证CheckSum

### 5.3 掉线时限参数设置

+ DN进程死亡或网络故障，无法与NN通信
+ NN不会立即将该节点判定为死亡，等待一段超时时长（默认10mins+30s）
+ 超时时长TimeOut = 2×dfs.namenode.heartbeat.recheck-interval + 10×dfs.heartbeat.interval
+ hdfs-site.xml默认dfs.namenode.heartbeat.recheck-interval为5分钟，dfs.heartbeat.interval为3秒

```xml
<property>
    <name>dfs.namenode.heartbeat.recheck-interval</name>
    <value>300000</value>
</property>

<property>
    <name>dfs.heartbeat.interval</name>
    <value>3</value>
</property>
```



​	
