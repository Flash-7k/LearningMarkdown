# 1. 集群搭建

## 配置文件

位置：`$HADOOP_HOME/etc/hadoop`

+ `core-site.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>

<configuration>
    <!-- 指定NameNode的地址 -->
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://hadoop102:8020</value>
    </property>

    <!-- 指定hadoop数据的存储目录 -->
    <property>
        <name>hadoop.tmp.dir</name>
        <value>/opt/module/hadoop-3.1.3/data</value>
    </property>

    <!-- 配置HDFS网页登录使用的静态用户为flash7k -->
    <property>
        <name>hadoop.http.staticuser.user</name>
        <value>flash7k</value>
    </property>
</configuration>
```

+ `hdfs-site.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>

<configuration>
    <!-- nn web端访问地址-->
    <property>
        <name>dfs.namenode.http-address</name>
        <value>hadoop102:9870</value>
    </property>
    <!-- 2nn web端访问地址-->
    <property>
        <name>dfs.namenode.secondary.http-address</name>
        <value>hadoop104:9868</value>
    </property>
</configuration>
```

+ `yarn-site.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>

<configuration>
    <!-- 指定MR走shuffle -->
    <property>
        <name>yarn.nodemanager.aux-services</name>
        <value>mapreduce_shuffle</value>
    </property>

    <!-- 指定ResourceManager的地址-->
    <property>
        <name>yarn.resourcemanager.hostname</name>
        <value>hadoop103</value>
    </property>

    <!-- 环境变量的继承 -->
    <property>
        <name>yarn.nodemanager.env-whitelist</name>
<value>JAVA_HOME,HADOOP_COMMON_HOME,HADOOP_HDFS_HOME,HADOOP_CONF_DIR,CLASSPATH_PREPEND_DISTCACHE,HADOOP_YARN_HOME,HADOOP_MAPRED_HOME</value>
    </property>


    <!-- 开启日志聚集功能 -->
    <property>
        <name>yarn.log-aggregation-enable</name>
        <value>true</value>
    </property>

    <!-- 设置日志聚集服务器地址 -->
    <property>  
        <name>yarn.log.server.url</name>  
        <value>http://hadoop102:19888/jobhistory/logs</value>
    </property>

    <!-- 设置日志保留时间为7天 -->
    <property>
        <name>yarn.log-aggregation.retain-seconds</name>
        <value>604800</value>
    </property>

</configuration>
```

+ `mapred-site.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>

<configuration>
    <!-- 指定MapReduce程序运行在Yarn上 -->
    <property>
        <name>mapreduce.framework.name</name>
        <value>yarn</value>
    </property>

    <!-- 历史服务器端地址 -->
    <property>
        <name>mapreduce.jobhistory.address</name>
        <value>hadoop102:10020</value>
    </property>

    <!-- 历史服务器web端地址 -->
    <property>
        <name>mapreduce.jobhistory.webapp.address</name>
        <value>hadoop102:19888</value>

</configuration>
```

+ `workers`

```xml
hadoop102
hadoop103
hadoop104
```

> 注意：该文件中添加的内容结尾不允许有空格，文件中不允许有空行

## 启动流程

1. **如果是第一次启动**，需要在hadoop102节点格式化NameNode
   
   > **注意：格式化NameNode，会产生新的集群id，导致NameNode和DataNode的集群id不一致，集群找不到已往数据。如果集群在运行过程中报错，需要重新格式化NameNode的话，一定要先停止namenode和datanode进程，并且要删除所有机器的data和logs目录，然后再进行格式化**

```linux
hdfs namenode -format
```

2. 启动HDFS

```shell
# 整体启动/停止
sbin/start-dfs.sh
sbin/stop-dfs.sh

# 逐一启动/停止
hdfs --daemon start/stop namenode/datanode/secondarynamenode
```

3. 启动YARN：在配置了ResourceManager的节点

```shell
#整体启动/停止
sbin/start-yarn.sh
sbin/stop-yarn.sh

# 逐一启动/停止
yarn --daemon start/stop  resourcemanager/nodemanager
```

4. 启动Historyserver

```shell
bin/mapred --daemon start historyserver
```

5. **Web端查看HDFS的NameNode**
+ 浏览器中输入：http://hadoop102:9870
+ 查看HDFS上存储的数据信息
6. **Web端查看YARN的ResourceManager**
+ 浏览器中输入：http://hadoop103:8088
+ 查看YARN上运行的Job信息
7. **Web端查看历史服务器**
+ 浏览器中输入：http://hadoop102:19888/jobhistory
+ 查看到MR程序运行详情

## 常用端口号

| 端口名称              | Hadoop2.x | Hadoop3.x      |
|:----------------- |:--------- |:-------------- |
| NameNode内部通信端口    | 8020/9000 | 8020/9000/9820 |
| NameNode Web端口    | 50070     | 9870           |
| MapReduce查看执行任务端口 | 8088      | 8088           |
| 历史服务器通信端口         | 10020     | 10020          |
| 历史服务器Web端口        | 19888     | 19888          |
| 2NameNode Web端口   |           | 9868           |

# 2. HDFS

## HDFS组成架构

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

## 文件块大小决定机制

**默认大小：Hadoop1.x  64M，Hadoop2.x/3.x  128M**

+ 若寻址时间约为10ms，由”寻址时间为传输时间的1%时，为最佳状态“
+ 得，传输时间=10ms/0.01=1000ms=1s
+ 目前磁盘的传输速率普遍为100MB/s
+ 因此Block默认大小128M

**思考：为什么块大小不能太大也不能太小？**

+ 太小：增加寻址时间
+ 太大：存取时间明显大于寻址时间

**总结：HDFS块的大小设置主要取决于磁盘传输速率**

## 写数据流程

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
   + 先从磁盘读取数据放到本地内存缓存，准备传输第一个`Block`
   + 起始有一个`Chunk`大小的`buf`，当数据写满这个`buf`，计算`Chunksum`值进行校验，然后填塞入Packet
   + 当`Packet`被`Chunk`填满后，将这个`Packet`放入应答队列中等待应答
   + 进入应答队列的`Packet`回被另一线程按序取出，发送到下一个DN
   + DN1收到一个`Paceket`就传给DN2，DN2传给DN3
9. 当第一个Block传输完成后，Client 再次请求NN上传第二个`Block`（重复3-7）

> **Block、Packet、Chunk**
> 
> **`Block`**：最大的单位，是最终存储在DN上的数据粒度，由`dfs.block.size`参数决定，默认128M，取决于客户端配置
> 
> **`Packet`**：中等的单位，是数据流向DN过程中的数据粒度，由`dfs.write.packet.size`参数决定，默认64k，以这个值为参考动态调整
> 
> **`Chunk`**：最小的单位，是数据流向DN过程中进行数据校验的数据粒度，由`io.byte.per.checksum`参数决定，默认512Byte
> 
> **注意：**事实上Chunk还包含一个4Byte的校验值，因此`Chunk`写入`Packet`时是516Byte
> 
> 数据与校验值的比例为128:1，所以一个128M的Block会有一个1M的校验文件

## 读数据流程

1. 客户端通过 DistributedFileSystem 向 NN 请求下载文件
2. NN 通过查询元数据，找到存储有目标文件块的DN地址
3. Client 挑选一台DN（就近原则、负载均衡）请求读取数据
4. DN开始传输数据给Client，从磁盘里读取数据输入流，以`Packet`为单位来做校验
5. Client 以`Packet`为单位接收，先在本地内存缓存，然后写入目标文件磁盘

## NameNode与2NameNode

**工作机制**

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
   + NN滚动正在写的`Edits`日志
   + 将滚动前的`Edits`编辑日志和`Fsimage`镜像文件拷贝到2NN
   + 2NN加载`Edits`和`Fsimage`，并合并生成元数据
   + 2NN生成新的镜像文件`fsimage.chkpoint`
   + 将`fsimage.chkpoint`拷贝到NN
   + NN将`fsimage.chkpoint`重新命名成`fsimage`

**Fsimage和Edits**

+ **Fsimage**
  
  HDFS元数据的一个永久性检查点，包含HDFS的所有文件目录和文件inode的序列化信息

+ **Edits**
  
  存放HDFS的所有更新操作的路径，Client执行的所有写操作首先会被记录到Edits中

+ **seen_txid**
  
  保持Edits最后一个数字

**CheckPoint时间**：`hdfs-default.xml`

1. **通常情况下，SecondaryNameNode每隔一小时执行一次**

```xml
<property>
    <name>dfs.namenode.checkpoint.period</name>
    <value>3600s</value>
</property>
```

2. **一分钟检查一次操作次数，当操作次数达到1百万时，SecondaryNameNode执行一次**

```xml
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

## DataNode工作机制

**工作机制**

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

**数据完整性**

+ 当DN读取Block时，计算CheckSum
+ 若计算后的CheckSum，与Block创建时的值不一样，说明Block已损坏
+ Client读取其他DN上的Block
+ 常见校验算法：crc（32），md5（128），sha1（160）
+ DN在其文件创建后周期性验证CheckSum

**掉线时限参数设置**

+ DN进程死亡或网络故障，无法与NN通信
+ NN不会立即将该节点判定为死亡，等待一段超时时长（默认10mins+30s）
+ 超时时长`TimeOut = 2×dfs.namenode.heartbeat.recheck-interval + 10×dfs.heartbeat.interval`
+ `hdfs-site.xml`默认`dfs.namenode.heartbeat.recheck-interval`为5分钟，`dfs.heartbeat.interval`为3秒

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

# 3. MapReduce

## Job提交流程

+ **Job提交**

```java
waitForCompletion()

submit();

// 1建立连接
    connect();    
        // 1）创建提交Job的代理
        new Cluster(getConfiguration());
            // （1）判断是本地运行环境还是yarn集群运行环境
            initialize(jobTrackAddr, conf); 

// 2 提交job
submitter.submitJobInternal(Job.this, cluster)

    // 1）创建给集群提交数据的Stag路径（临时文件）
    Path jobStagingArea = JobSubmissionFiles.getStagingDir(cluster, conf);

    // 2）获取jobid ，在Stag下创建Job路径
    JobID jobId = submitClient.getNewJobID();

    // 3）拷贝jar包到集群（本地模式则不拷贝jar包）
    copyAndConfigureFiles(job, submitJobDir);    
    rUploader.uploadFiles(job, jobSubmitDir);

    // 4）计算切片，向Stag路径生成切片规划文件
    writeSplits(job, submitJobDir);
        maps = writeNewSplits(job, jobSubmitDir);
        input.getSplits(job);

    // 5）向Stag路径写XML配置文件
    writeConf(conf, submitJobFile);
    conf.writeXml(out);

    // 6）提交Job,返回提交状态
    // 提交完后立刻删除Stag路径下文件
    status = submitClient.submitJob(jobId , submitJobDir.toString() , job.getCredentials());
```

* **流程解析**
  + `Driver：Job.waitForCompletion(true)`
  + `Job.submit()`
  + `JobSubmiter：Cluster`成员，proxy -> 判断是MR是在本地还是YARN
    + 创建stagingDir
      + 本地：`File://.../staging`
      + YARN：`hdfs://.../staging`
    + 获取jobid
      + 本地：`File://.../staging/jobid`
      + YARN：`hdfs://.../staging/jobid`
    + 调用`FileInputFormat.getSplits()`获取切片规划，并序列化成文件：**`Job.split`**
    + 将Job相关参数写到文件：**`Job.xml`**
    + 如果是YARN，还需要获取Job的jar包：**`xxx.jar`**

## MR流程

+ **MapReduce**
1. 待处理文件

2. 客户端`submit()`前，获取待处理数据的信息，配置参数，形成任务分配规划

3. 提交信息：`Job.split`（切片信息）、`wc.jar`（程序jar包）、`Job.xml`（配置信息）
   
   > 提交完后判断是提交到YARN还是本地运行

4. 计算出MapTask数量
   
   > Mrappmaster与Node Manager协作

5. MapTask通过`InputFormat`实现类，`RecorderReader`，以键值对形式读取待处理文件

6. Mapper执行逻辑运算
   
   > `map（k，v）`
   > 
   > `Context.write（k，v）`

7. outputCollector，向环形缓冲区中写入<k,v>数据
   
   > 默认100M，达到80%后反向
   > 
   > 一半存储元数据：索引、kvmeta、kvindex
   > 
   > 一半存储数据：<k,v>，bufindex

8. 分区、快排

9. 溢出到文件（分区、区内有序），由内存到磁盘

10. Merge 归并排序

11. Combiner 合并

12. 启动相应数量ReduceTask，并告知ReduceTask处理数据范围（数据分区）

13. ReduceTask拉取数据到本地磁盘，合并文件，归并排序

14. Reducer一次读取一组数据
    
    > `Reduce（k，v）`
    > 
    > `Context.write（k，v）`

15. `GroupingComparaton（k，knext）`分组

16. `OutputFormat`，`RecordWriter`，`Write（k，v）`
+ **Shuffle**（MR的7-16）
1. MapTask收集自定义的map（）方法输出的kv对，放到内存缓冲区中
2. 从内存缓冲区不断溢写本地磁盘文件（可能溢出多个文件）
3. 多个溢出文件会被合并成大的溢出文件
4. 在溢写和合并的过程中，都需要调用Partitioner分区和对key排序
5. ReduceTask根据分区号，去各个MapTask上抓取相应的结果分区数据
6. ReduceTask抓取同一个分区的，来自不同MapTask的结果文件，再将这些文件进行合并（归并排序）
7. 合并成大文件后，进入ReduceTask的逻辑运算过程（从文件中取出一个个kv对，调用自定义的reduce（））

> 注意：
> 
> 1. Shuffle环形缓冲区的大小会影响MapReduce的执行效率。缓冲区越大，磁盘io次数越少，执行速度越快
> 2. 缓冲区大小设置：`mapreduce.task.io.sort.mb`默认100M

## MapTask并行度决定机制

**MapTask的并行度决定Map阶段的任务处理并发度，进而影响到整个Job的处理速度**

+ **MapTask并行度决定机制**
  
  + **数据块**：HDFS在**物理上**把数据分块。数据块是HDFS存储数据单位
  + **数据切片**：MapRduce在**逻辑上**把数据分块。数据切片是MapReduce程序计算输入数据的单位，一个切片对应启动一个MapTask
  + **决定机制**
    + 一个Job的MapTask由Client在提交Job时的切片数决定
    + 一个切片对应一个MapTask
    + 默认情况下，切片大小=数据块大小= BlockSize
    + 切片时不考虑数据集整体，逐个针对每个文件单独切片

+ 源码中计算切片大小的公式
  
  + `Math.max ( minSize , Math.min ( maxSize , blockSize ))`
  + `mapreduce.input.fileinputformat.split.minsize = 1`
  + `mapreduce.input.fileinputformat.split.maxsize = Long.MAXValue`

+ 切片大小参数设置
  
  + `maxsize`：如果比`blocksize`小，则让切片变小，值为`maxsize`
  + `minsize`：如果比`blocksize`大，则让切片变大，值为`minsize`

## Map切片机制

### FileInputFormat

+ **切片机制**
  
  + 简单地按照文件的内容长度进行切片
  + 切片大小默认等于Block大小
  + 切片时**不考虑数据集整体**，而是逐个**针对每个文件单独切片**

+ **源码解析（`input.getSplits(job`))**
  
  + 程序找到数据存储的目录
  
  + 遍历处理目录下的每一个文件（规划切片）
  
  + 处理第一个文件`xx.txt`
    
    + 获取文件大小 `fs.sizeOf (xx.txt)`
    
    + 计算切片大小 `computeSplitSize (Math.max (minSize, Math.min (maxSize,blockSize))) = blockSize`
    
    + 默认情况下，切片大小即为数据块大小
      
      > 可通过修改默认为`Long.MAXValue `的`maxSize`调小切片，修改默认为1的`minSize`调大切片
    
    + 开始切片，形成第1个切片：0-128M，第2个切片：128-200M
      
      > 每次切片时，都要判断切完剩下部分是否大于块的1.1倍，不大于1.1倍就划分一块切片
    
    + 将切片信息写到一个切片规划文件`Job.split`中
    
    + 整个切片的核心过程在getSplits()方法中完成
    
    + `InputSplit`只记录了切片的元数据信息（起始位置、长度、所在节点列表）
  
  + 提交切片规划文件到YARN上，MrAppMaster根据切片规划文件计算开启MapTask数量

### CombineTextInputFormat

> 默认切片机制：**按文件规划切片**。不管文件多小，都会是一个单独的切片，产生一个MapTask
> 
> 缺点：**如果有大量小文件，就会产生大量MapTask，处理效率低下**

+ **CombineTextInputFormat**
  
  + 应用场景：大量小文件
  
  + 作用：将多个小文件从逻辑上规划到一个切片中。使多个小文件交给一个MapTask处理
  
  + 方式：设置虚拟存储切片最大值
    
    ```java
    CombineTextInputFormat.setMaxInputSplitSize(job, 4194304); // 4m
    ```
  
  + 生成切片过程：虚拟存储、切片
  
  + 样例：
    
    > 四个小文件：a--1.7M、b--5.1M、c--3.4M、d--6.8M
    > 
    > 虚拟存储：a--1.7M、b--2.55M+2.55M、c--3.4M、d--3.4M+3.4M
    > 
    > 切片过程：最终形成三个切片，（1.7+2.55）M、（2.55+3.4）M、（3.4+3.4）M
  
  + 解释
    
    + 虚拟存储过程
      
      > 比较 输入目录下所有文件大小 和 设置的` setMaxInputSplitSize`值
      > 
      > 文件<=设置最大值，逻辑上划分一个块；
      > 
      > 设置最大值<文件<=最大值*2，将文件均分成2个虚拟存储块（防止出现太小切片）
      > 
      > 文件>设置最大值*2，以最大值切割一块
    
    + 切片过程
      
      > 虚拟存储文件大小>=设置的 `setMaxInputSplitSize`值，单独形成一个切片
      > 
      > 否则跟下一个虚拟存储文件进行合并，共同形成一个切片

## MapTask工作机制

1. **Read阶段**
   
   MapTask 通过 **`InputFormat`** 获得的 **`RecordReader`**，从输入` InputSplit `中解析出一个个`key / value`

2. **Map阶段**
   
   将解析出的` key / value `交给用户编写的 **`map()`** 函数处理，并产生一系列新的` key / value`

3. **Collect阶段**
   
   用户编写的 `map()` 函数数据处理完成后，一般会调用**`OutputCollector.collect()`** 输出结果，将产生新的 `key / value`，**分区（调用Partitioner）**，并**写入一个环形内存缓冲区**中

4. **Spill阶段（溢写）**
   
   环形缓冲区达到80%后，MapReduce 会将数据写到本地磁盘上，生成一个临时文件。注意：数据写入本地磁盘前，先对数据进行一次本地排序，并在必要时进行合并、压缩
   
   + 使用快速排序算法**对缓冲区内的数据进行排序**。先按照分区编号，再按照 key。排序后，数据以分区为单位聚集在一起，**同一分区内所有数据按照 key 有序**
   + 按照分区编号由小到大，依次将每个分区中的**数据写入任务工作目录下的临时文件output / spillN.out** （N表示当前溢写次数）。<u>如果用户设置了 Combiner，则写入文件之前，对每个分区中的数据进行一次聚集操作</u>
   + 将分区数据的元**信息写到内存索引数据结构 SpillRecord** 中，元信息包括在临时文件中的偏移量、压缩前数据大小、压缩后数据大小。<u>如果当前内存索引大小超过1MB，则将内存索引写到文件 `output` /` spillN.out.index`中</u>

5. **Merge阶段**
   
   当所有数据处理完成后，MapTask **对所有临时文件进行一次合并**，以确保最终只会生产一个数据文件，并将其保存到文件 `output / file.out` 中，同时生成相应的索引文件 `output / file.out.index`
   
   > **合并方式**：MapTask 以分区为单位进行合并。对于分区采用**多轮递归合并**，每轮合并 `mapreduce.task.io.sort.factor`（默认10）个文件，并将产生的文件重新加入待合并列表中。排序后，重复以上过程，直到最终获得一个大文件
   > 
   > **好处**：避免同时打开大量文件和同时随机读取大量小文件带来的开销

## ReduceTask并行度决定机制

> **MapTask**并行度由切片个数决定，切片个数由**输入文件**和**切片规则**决定
> 
> **ReduceTask**数量的决定直接由**手动设置**

注意：

1. ReduceTask = 0，表示没有Reduce阶段，输出文件个数和Map个数一致
2. ReduceTask 默认值为1，输出文件个数为1
3. 如果数据分布不均匀，可能在 Reduce 阶段产生数据倾斜
4. ReduceTask 数量并不是任意设置，需要**考虑业务逻辑需求**。有时需要计算全局汇总结果，只能有一个ReduceTask
5. 具体多少个ReduceTask，需要**根据集群性能而定**
6. 若分区数不是1，而ReduceTask为1：不执行分区过程。因为**执行分区的前提是ReduceNum大于1**

## ReduceTask工作机制

1. **Copy阶段**
   
   ReduceTask 从各个 MapTask 上远程拷贝一片数据，如果数据大小不超过一定阈值，则直接放到内存中，否则写到磁盘上

2. **Sort阶段**
   
   在远程拷贝数据过程中，ReduceTask 启动两个后台线程对内存和磁盘上的文件进行合并，以防止内存使用过多或磁盘上文件过多
   
   > 用户编写`reduce()`，输入数据根据key进行聚集，为了将 key 相同的数据聚在一起，Hadoop 采用基于排序的策略。由于各个 MapTask 已经对处理结果进行了局部排序，ReduceTask 只需对所有数据进行一次归并排序即可

3. **Reduce阶段**
   
   `reduce() `将计算结果写到 HDFS 上

## Join

+ Map端主要工作：对来自不同表或文件的 key / value，打标签以区别不同来源的记录，然后连接字段作为key，其余部分和新加的标志作为value，输出
+ Reduce端主要工作：在每一个分组当中将不同来源文件的记录（在Map阶段已标记）分开，最后进行合并

### Reduce Join

+ 需求：订单数据表order有订单号、产品号、数量，商品信息表有产品号、产品名，需要根据产品号将产品名合并到订单数据表中

+ 需求分析：将关联条件作为Map输出的key，满足Join条件的数据并携带数据来源的文件信息作为value，发往同一个ReduceTask，在Reduce中进行数据的串联

+ 代码
  
  + 创建商品和订单合并后的TableBean类
    
    ```java
    package com.flash7k.mapreduce.reducejoin;
    
    import org.apache.hadoop.io.Writable;
    
    import java.io.DataInput;
    import java.io.DataOutput;
    import java.io.IOException;
    
    public class TableBean implements Writable {
    
        private String id; //订单id
        private String pid; //产品id
        private int amount; //产品数量
        private String pname; //产品名称
        private String flag; //判断是order表还是pd表的标志字段
    
        public TableBean() {
        }
    
        public String getId() {
            return id;
        }
    
        public void setId(String id) {
            this.id = id;
        }
    
        public String getPid() {
            return pid;
        }
    
        public void setPid(String pid) {
            this.pid = pid;
        }
    
        public int getAmount() {
            return amount;
        }
    
        public void setAmount(int amount) {
            this.amount = amount;
        }
    
        public String getPname() {
            return pname;
        }
    
        public void setPname(String pname) {
            this.pname = pname;
        }
    
        public String getFlag() {
            return flag;
        }
    
        public void setFlag(String flag) {
            this.flag = flag;
        }
    
        @Override
        public String toString() {
            return id + "\t" + pname + "\t" + amount;
        }
    
        @Override
        public void write(DataOutput out) throws IOException {
            out.writeUTF(id);
            out.writeUTF(pid);
            out.writeInt(amount);
            out.writeUTF(pname);
            out.writeUTF(flag);
        }
    
        @Override
        public void readFields(DataInput in) throws IOException {
            this.id = in.readUTF();
            this.pid = in.readUTF();
            this.amount = in.readInt();
            this.pname = in.readUTF();
            this.flag = in.readUTF();
        }
    }
    ```
  
  + 编写TableMapper类
    
    ```java
    package com.flash7k.mapreduce.reducejoin;
    
    import org.apache.hadoop.io.LongWritable;
    import org.apache.hadoop.io.Text;
    import org.apache.hadoop.mapreduce.InputSplit;
    import org.apache.hadoop.mapreduce.Mapper;
    import org.apache.hadoop.mapreduce.lib.input.FileSplit;
    
    import java.io.IOException;
    
    public class TableMapper extends Mapper<LongWritable,Text,Text,TableBean> {
    
        private String filename;
        private Text outK = new Text();
        private TableBean outV = new TableBean();
    
        @Override
        protected void setup(Context context) throws IOException, InterruptedException {
            //获取对应文件名称
            InputSplit split = context.getInputSplit();
            FileSplit fileSplit = (FileSplit) split;
            filename = fileSplit.getPath().getName();
        }
    
        @Override
        protected void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException {
    
            //获取一行
            String line = value.toString();
    
            //判断是哪个文件,然后针对文件进行不同的操作
            if(filename.contains("order")){  //订单表的处理
                String[] split = line.split("\t");
                //封装outK
                outK.set(split[1]);
                //封装outV
                outV.setId(split[0]);
                outV.setPid(split[1]);
                outV.setAmount(Integer.parseInt(split[2]));
                outV.setPname("");
                outV.setFlag("order");
            }else {                             //商品表的处理
                String[] split = line.split("\t");
                //封装outK
                outK.set(split[0]);
                //封装outV
                outV.setId("");
                outV.setPid(split[0]);
                outV.setAmount(0);
                outV.setPname(split[1]);
                outV.setFlag("pd");
            }
    
            //写出KV
            context.write(outK,outV);
        }
    }
    ```
  
  + 编写TableReducer类
    
    ```java
    package com.flash7k.mapreduce.reducejoin;
    
    import org.apache.commons.beanutils.BeanUtils;
    import org.apache.hadoop.io.NullWritable;
    import org.apache.hadoop.io.Text;
    import org.apache.hadoop.mapreduce.Reducer;
    
    import java.io.IOException;
    import java.lang.reflect.InvocationTargetException;
    import java.util.ArrayList;
    
    public class TableReducer extends Reducer<Text,TableBean,TableBean, NullWritable> {
    
        @Override
        protected void reduce(Text key, Iterable<TableBean> values, Context context) throws IOException, InterruptedException {
    
            ArrayList<TableBean> orderBeans = new ArrayList<>();
            TableBean pdBean = new TableBean();
    
            for (TableBean value : values) {
    
                //判断数据来自哪个表
                if("order".equals(value.getFlag())){   //订单表
    
                  //创建一个临时TableBean对象接收value
                    TableBean tmpOrderBean = new TableBean();
    
                    try {
                        BeanUtils.copyProperties(tmpOrderBean,value);
                    } catch (IllegalAccessException e) {
                        e.printStackTrace();
                    } catch (InvocationTargetException e) {
                        e.printStackTrace();
                    }
    
                  //将临时TableBean对象添加到集合orderBeans
                    orderBeans.add(tmpOrderBean);
                }else {                                    //商品表
                    try {
                        BeanUtils.copyProperties(pdBean,value);
                    } catch (IllegalAccessException e) {
                        e.printStackTrace();
                    } catch (InvocationTargetException e) {
                        e.printStackTrace();
                    }
                }
            }
    
            //遍历集合orderBeans,替换掉每个orderBean的pid为pname,然后写出
            for (TableBean orderBean : orderBeans) {
    
                orderBean.setPname(pdBean.getPname());
    
               //写出修改后的orderBean对象
                context.write(orderBean,NullWritable.get());
            }
        }
    }
    ```
  
  + 编写TableDriver类
    
    ```java
    package com.flash7k.mapreduce.reducejoin;
    
    import org.apache.hadoop.conf.Configuration;
    import org.apache.hadoop.fs.Path;
    import org.apache.hadoop.io.NullWritable;
    import org.apache.hadoop.io.Text;
    import org.apache.hadoop.mapreduce.Job;
    import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
    import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;
    
    import java.io.IOException;
    
    public class TableDriver {
        public static void main(String[] args) throws IOException, ClassNotFoundException, InterruptedException {
            Job job = Job.getInstance(new Configuration());
    
            job.setJarByClass(TableDriver.class);
            job.setMapperClass(TableMapper.class);
            job.setReducerClass(TableReducer.class);
    
            job.setMapOutputKeyClass(Text.class);
            job.setMapOutputValueClass(TableBean.class);
    
            job.setOutputKeyClass(TableBean.class);
            job.setOutputValueClass(NullWritable.class);
    
            FileInputFormat.setInputPaths(job, new Path("D:\\input"));
            FileOutputFormat.setOutputPath(job, new Path("D:\\output"));
    
            boolean b = job.waitForCompletion(true);
            System.exit(b ? 0 : 1);
        }
    }
    ```
  
  + 总结
    
    缺点：Reduce Join，合并操作在Reduce阶段完成，处理压力太大，Map节点的运算负载却很低，资源利用率不高。在Reduce阶段极易产生数据倾斜
    
    解决方案：Map Join

### Map Join

+ 使用场景：一张表很小，另一张表很大

+ 优点：减少数据倾斜

+ 方法：Map端缓存多张表，采用DistributedCache
  
  + 在Mapper的setup阶段，将文件读取到缓存集合中
  
  + 在Driver中加载缓存
    
    ```java
    //缓存普通文件到Task运行节点。
    job.addCacheFile(new URI("file:///e:/cache/pd.txt"));
    //如果是集群运行,需要设置HDFS路径
    job.addCacheFile(new URI("hdfs://hadoop102:8020/cache/pd.txt"));
    ```

+ 需求分析
  
  + DistributedCacheDriver缓存文件
    1. 加载缓存数据 `job.addCacheDriver(new URI(文件路径))`；
    2. Map Join不需要Reduce阶段，设置 `job.setNumReduceTask(0)`;
  + 读取缓存的文件数据
    + `setup()`
      1. 获取缓存的文件
      2. 循环读取缓存文件中的一行
      3. 切割
      4. 缓存数据到集合
      5. 关流
    + `map()`
      1. 获取一行
      2. 截取
      3. 获取所需关联数据
      4. 获取其他数据
      5. 拼接
      6. 输出

+ 代码
  
  + 在MapJoinDriver驱动类中添加缓存文件
    
    ```java
    package com.flash7k.mapreduce.mapjoin;
    
    import org.apache.hadoop.conf.Configuration;
    import org.apache.hadoop.fs.Path;
    import org.apache.hadoop.io.NullWritable;
    import org.apache.hadoop.io.Text;
    import org.apache.hadoop.mapreduce.Job;
    import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
    import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;
    
    import java.io.IOException;
    import java.net.URI;
    import java.net.URISyntaxException;
    
    public class MapJoinDriver {
    
        public static void main(String[] args) throws IOException, URISyntaxException, ClassNotFoundException, InterruptedException {
    
            // 1 获取job信息
            Configuration conf = new Configuration();
            Job job = Job.getInstance(conf);
            // 2 设置加载jar包路径
            job.setJarByClass(MapJoinDriver.class);
            // 3 关联mapper
            job.setMapperClass(MapJoinMapper.class);
            // 4 设置Map输出KV类型
            job.setMapOutputKeyClass(Text.class);
            job.setMapOutputValueClass(NullWritable.class);
            // 5 设置最终输出KV类型
            job.setOutputKeyClass(Text.class);
            job.setOutputValueClass(NullWritable.class);
    
            // 加载缓存数据
            job.addCacheFile(new URI("file:///D:/input/tablecache/pd.txt"));
            // Map端Join的逻辑不需要Reduce阶段，设置reduceTask数量为0
            job.setNumReduceTasks(0);
    
            // 6 设置输入输出路径
            FileInputFormat.setInputPaths(job, new Path("D:\\input"));
            FileOutputFormat.setOutputPath(job, new Path("D:\\output"));
            // 7 提交
            boolean b = job.waitForCompletion(true);
            System.exit(b ? 0 : 1);
        }
    }
    ```
  
  + 在MapJoinMapper类中的setup方法中读取缓存文件
    
    ```java
    package com.flash7k.mapreduce.mapjoin;
    
    import org.apache.commons.lang.StringUtils;
    import org.apache.hadoop.fs.FSDataInputStream;
    import org.apache.hadoop.fs.FileSystem;
    import org.apache.hadoop.fs.Path;
    import org.apache.hadoop.io.IOUtils;
    import org.apache.hadoop.io.LongWritable;
    import org.apache.hadoop.io.NullWritable;
    import org.apache.hadoop.io.Text;
    import org.apache.hadoop.mapreduce.Mapper;
    
    import java.io.BufferedReader;
    import java.io.IOException;
    import java.io.InputStreamReader;
    import java.net.URI;
    import java.util.HashMap;
    import java.util.Map;
    
    public class MapJoinMapper extends Mapper<LongWritable, Text, Text, NullWritable> {
    
        private Map<String, String> pdMap = new HashMap<>();
        private Text text = new Text();
    
        //任务开始前将pd数据缓存进pdMap
        @Override
        protected void setup(Context context) throws IOException, InterruptedException {
    
            //通过缓存文件得到小表数据pd.txt
            URI[] cacheFiles = context.getCacheFiles();
            Path path = new Path(cacheFiles[0]);
    
            //获取文件系统对象,并开流
            FileSystem fs = FileSystem.get(context.getConfiguration());
            FSDataInputStream fis = fs.open(path);
    
            //通过包装流转换为reader,方便按行读取
            BufferedReader reader = new BufferedReader(new InputStreamReader(fis, "UTF-8"));
    
            //逐行读取，按行处理
            String line;
            while (StringUtils.isNotEmpty(line = reader.readLine())) {
                //切割一行    
                //01    小米
                String[] split = line.split("\t");
                pdMap.put(split[0], split[1]);
            }
    
            //关流
            IOUtils.closeStream(reader);
        }
    
        @Override
        protected void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException {
    
            //读取大表数据    
            //1001    01    1
            String[] fields = value.toString().split("\t");
    
            //通过大表每行数据的pid,去pdMap里面取出pname
            String pname = pdMap.get(fields[1]);
    
            //将大表每行数据的pid替换为pname
            text.set(fields[0] + "\t" + pname + "\t" + fields[2]);
    
            //写出
            context.write(text,NullWritable.get());
        }
    }
    ```

# 4. Yarn

整章都很重要，详情见`Hadoop-Yarn`笔记

# 5. 生产调优

## NameNode故障处理

+ 需求：NameNode进程挂了并且存储的数据也丢失了，如何恢复NameNode

+ 模拟故障：
  
  + kill -9 NameNode进程
  + 删除NameNode存储的数据（`rm -rf /opt/module/hadoop-3.1.3/data/tmp/dfs/name/*`）

+ 问题解决
  
  + 拷贝SecondaryNameNode中数据到原NameNode存储数据目录
  
  ```shell
  scp -r flash7k@hadoop104:/opt/module/hadoop3.1.3/data/dfs/namesecondary/* /opt/module/hadoop3.1.3/data/dfs/name/
  ```
  
  + 重新启动NameNode
  
  ```shell
  hdfs --daemon start namenode
  ```

## 小文件归档

每个文件均按块存储，每个块的元数据存储在NameNode的内存中。大量的小文件会耗尽NameNode中的大部分内存，因此HDFS存储小文件会非常低效。注意，存储小文件所需要的磁盘容量和数据块的大小无关。例如，一个1MB的文件设置为128MB的块存储，实际使用的是1MB的磁盘空间，而不是128MB。

+ 解决办法
  
  + HDFS存档文件或HAR文件，是一个更高效的文件存档工具，它将文件存入HDFS块，在减少NameNode内存使用的同时，允许对文件进行透明的访问。具体说来，HDFS存档文件对内还是一个一个独立文件，对NameNode而言却是一个整体，减少了NameNode的内存。

+ 案例
  
  + 启动Yarn
  
  ```shell
  start-yarn.sh
  ```
  
  + 归档文件：把`/input`目录里面的所有文件归档成一个叫`input.har`的归档文件，并把归档后文件存储到`/output`路径下
  
  ```shell
  hadoop archive -archiveName input.har -p /input /output
  ```
  
  + 查看归档
  
  ```shell
  hadoop fs -ls /output/input.har
  
  hadoop fs -ls har:///output/input.har
  ```
  
  + 解归档文件
  
  ```shell
  hadoop fs -cp har:///output/input.har/*    /
  ```

# 6. 个人认为重要的几点

## 数据倾斜

+ `Group By`：某一个Key数目过多，`set hive.groupby.skewindata = true;`，为这个Key加上随机数打散，再用一个MR聚合
+ `Map Join`：Reduce端完成Join很慢，`set hive.auto.convert.join=true;`，将小表放入内存中，在Map端完成Join

## 小文件问题

+ 小文件归档：将小文件归档成一个`har`文件，对NN来说这是一个整体
+ `FileInputFormat`：每次切片时，都要判断切完剩下部分是否大于块的1.1倍，不大于1.1倍就只划分一块切片
+ `CombineTextInputFormat`：将多个小文件从逻辑上规划到一个切片中。使多个小文件交给一个MapTask处理，设置虚拟存储切片值`CombineTextInputFormat.setMaxInputSplitSize(job, 4194304); //4m`
