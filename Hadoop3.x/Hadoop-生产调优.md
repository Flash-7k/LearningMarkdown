# 1. HDFS-核心参数

## 1）NameNode内存生产配置

+ NameNode内存计算

  + 每个文件块大概占用150byte，一台服务器128G内存为例，能存储多少文件块？
  + 128 * 1024 * 1024 * 1024 / 150 byte ≈ 9.1亿

+ Hadoop2.x，配置NameNode内存

  + NN内存默认2000M。如果服务器内存4G，NN内存可以配置3G
  + 修改`hadoop-env.sh`

  ```sh
  HADOOP_NAMENODE_OPTS=-Xmx3072m
  ```

+ Hadoop3.x，配置NameNode内存

  + `hadoop-env.sh`，描述Hadoop内存是动态分配

  ```sh
  # The maximum amount of heap to use (Java -Xmx).  If no unit
  # is provided, it will be converted to MB.  Daemons will
  # prefer any Xmx setting in their respective _OPT variable.
  # There is no default; the JVM will autoscale based upon machine
  # memory size.（自动分配）
  # export HADOOP_HEAPSIZE_MAX=
  
  # The minimum amount of heap to use (Java -Xms).  If no unit
  # is provided, it will be converted to MB.  Daemons will
  # prefer any Xms setting in their respective _OPT variable.
  # There is no default; the JVM will autoscale based upon machine
  # memory size.（自动分配）
  # export HADOOP_HEAPSIZE_MIN=HADOOP_NAMENODE_OPTS=-Xmx102400m
  ```

  + 查看NameNode占用内存

  ```shell
  jmap -heap [NameNode进程端口号]
  ```

  + 经查看发现NameNode和DataNode占用内存都是自动分配，且相等，不合理

    > 参考：https://docs.cloudera.com/documentation/enterprise/6/release-notes/topics/rg_hardware_requirements.html#concept_fzz_dq4_gbb
    >
    > NameNode最小值：1G。每增加1000000个Block，增加1G内存
    >
    > DataNode最小值：4G。一个DN上的副本综述低于4000000，设为4G，每增加1000000，增加1G内存

  + 修改`hadoop-env.sh`

  ```sh
  #加在文件底部
  export HDFS_NAMENODE_OPTS="-Dhadoop.security.logger=INFO,RFAS -Xmx1024m"
  
  export HDFS_DATANODE_OPTS="-Dhadoop.security.logger=ERROR,RFAS -Xmx1024m"
  ```

## 2）NameNode心跳并发配置

+ `hdfs-site.xml`

```xml
<!--The number of Namenode RPC server threads that listen to requests from clients. If dfs.namenode.servicerpc-address is not configured then Namenode RPC server threads listen to requests from all nodes.

NameNode有一个工作线程池，用来处理不同DataNode的并发心跳以及客户端并发的元数据操作。
对于大集群或者有大量客户端的集群来说，通常需要增大该参数。默认值是10。-->

<property>
    <name>dfs.namenode.handler.count</name>
    <value>21</value>
</property>
```

+ 修改`dfs.namenode.handler.count`为21（企业经验：`dfs.namenode.handler.count=20*loge(Cluster Size)`，集群规模为3时，设置21台）

```shell
# 安装python
[flash7k@hadoop102 ~]$ sudo yum install -y python

# 进入python
[flash7k@hadoop102 ~]$ python
Python 2.7.5 (default, Apr 11 2018, 07:36:10) 
[GCC 4.8.5 20150623 (Red Hat 4.8.5-28)] on linux2
Type "help", "copyright", "credits" or "license" for more information.
>>> import math
>>> print int(20*math.log(3))
21
>>> quit()
```

## 3）开启回收站配置

开启回收站功能，可以将删除的文件在不超时的情况下，恢复原数据，起到防止误删除、备份等作用。

+ 参数说明

  + `fs.trash.interval`：默认为0，表示禁用回收站。设置为其他值表示设置文件的存活时间
  + `fs.trash.checkpoint.interval `：默认为0，检查回收站的间隔时间。若为0，则说明`fs.trash.interval`也为0
  + 要求：`fs.trash.checkpoint.interval `<=`fs.trash.interval`

+ 启用回收站

  + 修改`core-site.xml`，配置垃圾回收时间为1分钟

  ```xml
  <property>
      <name>fs.trash.interval</name>
      <value>1</value>
  </property>
  ```

+ 查看回收站

  + 回收站目录：hdfs:/user/flash7k/.Trash/...

  > 注意：
  >
  > 通过网页上直接删除的文件不会走回收站
  >
  > 通过程序删除的文件不会经过回收站，需要调用`moveToTrash()`才进入回收站
  >
  > ```java
  > Trash trash = New Trash(conf);
  > trash.moveToTrash(path);
  > ```
  >
  > **只有在命令行利用`hadoop fs -rm`命令删除的文件才会走回收站**

+ 恢复回收站数据：在存活时间内将回收站内数据复制到指定文件夹

# 2. HDFS-集群压测

HDFS的读写性能主要受**网络和磁盘**影响比较大。为了方便测试，将hadoop102、hadoop103、hadoop104虚拟机网络都设置为100Mbps。

> 100Mbps单位是bit；10M/s单位是byte ; 1byte=8bit，100Mbps/8=12.5M/s

+ 测试网速：在`/opt/module`目录，创建

```shell
python -m SimpleHTTPServer
```

## 1）测试HDFS写性能

+ 内容：向HDFS集群写10个128M的文件

```shell
hadoop jar /opt/module/hadoop-3.1.3/share/hadoop/mapreduce/hadoop-mapreduce-client-jobclient-3.1.3-tests.jar TestDFSIO -write -nrFiles 10 -fileSize 128MB

#结果：
2021-02-09 10:43:16,853 INFO fs.TestDFSIO: ----- TestDFSIO ----- : write
2021-02-09 10:43:16,854 INFO fs.TestDFSIO:  Date & time: Tue Feb 09 10:43:16 CST 2021
2021-02-09 10:43:16,854 INFO fs.TestDFSIO:         Number of files: 10
2021-02-09 10:43:16,854 INFO fs.TestDFSIO:  Total MBytes processed: 1280
2021-02-09 10:43:16,854 INFO fs.TestDFSIO:       Throughput mb/sec: 1.61
2021-02-09 10:43:16,854 INFO fs.TestDFSIO:  Average IO rate mb/sec: 1.9
2021-02-09 10:43:16,854 INFO fs.TestDFSIO:   IO rate std deviation: 0.76
2021-02-09 10:43:16,854 INFO fs.TestDFSIO:      Test exec time sec: 133.05
2021-02-09 10:43:16,854 INFO fs.TestDFSIO:
```

> 注意：nrFiles n为生成mapTask的数量，生产环境一般可通过hadoop103:8088查看CPU核数，设置为（CPU核数 - 1）
>
> + `Number of files`：生成mapTask数量，一般是集群中（ CPU核数-1），我们测试虚拟机就按照实际的物理内存-1分配即可
>
> + `Total MBytes processed`：单个map处理的文件大小
>
> + `Throughput mb/sec`：单个mapTak的吞吐量 
>
>   计算方式：处理的总文件大小/每一个mapTask写数据的时间累加
>
>   集群整体吞吐量：生成mapTask数量*单个mapTak的吞吐量
>
> + `Average IO rate mb/sec`：平均mapTak的吞吐量
>
>   计算方式：每个mapTask处理文件大小/每一个mapTask写数据的时间全部相加除以task数量
>
> + `IO rate std deviation`：方差、反映各个mapTask处理的差值，越小越均衡

+ 异常：在`yarn-site.xml`中设置关闭虚拟内存检测，分发配置并重启集群

```xml
<!--是否启动一个线程检查每个任务正使用的虚拟内存量，如果任务超出分配值，则直接将其杀掉，默认是true -->
<property>
     <name>yarn.nodemanager.vmem-check-enabled</name>
     <value>false</value>
</property>
```

## 2）测试HDFS读性能

+ 测试内容：读取HDFS集群10个128M的文件

```shell
hadoop jar /opt/module/hadoop-3.1.3/share/hadoop/mapreduce/hadoop-mapreduce-client-jobclient-3.1.3-tests.jar TestDFSIO -read -nrFiles 10 -fileSize 128MB

# 结果：
2021-02-09 11:34:15,847 INFO fs.TestDFSIO: ----- TestDFSIO ----- : read
2021-02-09 11:34:15,847 INFO fs.TestDFSIO:  Date & time: Tue Feb 09 11:34:15 CST 2021
2021-02-09 11:34:15,847 INFO fs.TestDFSIO:         Number of files: 10
2021-02-09 11:34:15,847 INFO fs.TestDFSIO:  Total MBytes processed: 1280
2021-02-09 11:34:15,848 INFO fs.TestDFSIO:       Throughput mb/sec: 200.28
2021-02-09 11:34:15,848 INFO fs.TestDFSIO:  Average IO rate mb/sec: 266.74
2021-02-09 11:34:15,848 INFO fs.TestDFSIO:   IO rate std deviation: 143.12
2021-02-09 11:34:15,848 INFO fs.TestDFSIO:      Test exec time sec: 20.83
```

+ 删除测试生成数据

```shell
hadoop jar /opt/module/hadoop-3.1.3/share/hadoop/mapreduce/hadoop-mapreduce-client-jobclient-3.1.3-tests.jar TestDFSIO -clean
```

# 3. HDFS-多目录

> 注意：在项目开启时创建多目录，因为增加目录需要删除之前全部数据

## 1）NameNode多目录配置

+ NN本地目录可配置多个，且**每个目录存放内容相同**，增加可靠性
+ 配置：`hdfs-site.xml`，增加name1和name2目录

```xml
<property>
     <name>dfs.namenode.name.dir</name>
     <value>file://${hadoop.tmp.dir}/dfs/name1,file://${hadoop.tmp.dir}/dfs/name2</value>
</property>
```

> 因为每台服务器节点的磁盘情况不同，所以这个配置配完之后，可以选择不分发

+ **停止集群，删除三台节点中data和logs目录**

```shell
rm -rf data/ logs/
```

+ **格式化集群并启动**

```shell
bin/hdfs namenode -format
sbin/start-dfs.sh
```

## 2）DataNode多目录配置

+ DN可以配置多个目录，每个目录存储的数据不一样（数据不是副本）

+ 配置：`hdfs-site.xml`，增加data1和data1目录

```xml
<property>
     <name>dfs.datanode.data.dir</name>
     <value>file://${hadoop.tmp.dir}/dfs/data1,file://${hadoop.tmp.dir}/dfs/data2</value>
</property>
```

+ **向集群上传一个文件，再次观察两个文件夹里面的内容发现不一致（一个有数一个没有）**

## 3）集群数据均衡：磁盘间数据均衡

生产环境，由于硬盘空间不足，往往需要增加一块硬盘。刚加载的硬盘没有数据时，可以执行磁盘数据均衡命令。（Hadoop3.x新特性）

+ 生成均衡计划（只有一块磁盘，不会生成计划）

```shell
hdfs diskbalancer -plan hadoop103
```

+ 执行均衡计划

```shell
hdfs diskbalancer -execute hadoop103.plan.json
```

+ 查看当前均衡任务的执行情况

```shell
hdfs diskbalancer -query hadoop103
```

+ 取消均衡任务

```shell
hdfs diskbalancer -cancel hadoop103.plan.json
```

# 4. HDFS-集群扩容及缩容

# 5. HDFS-存储优化

# 6. HDFS-集群迁移

# 7. HDFS-故障排除

## 1）NameNode故障处理

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


## 2）集群安全模式&磁盘修复

### 安全模式

+ 文件系统只接受读数据请求，而不接受删除、修改等变更请求

+ **进入安全模式场景**

  + NameNode在加载镜像文件和编辑日志期间处于安全模式
  + NameNode在接收DataNode注册时，处于安全模式

+ **退出安全模式条件**

  + `dfs.namenode.safemode.min.datanodes`：最小可用datanode数量，默认0
  + `dfs.namenode.safemode.threshold-pct`：副本数达到最小要求的block占系统总block数的百分比，默认0.999f。（只允许丢一个块）
  + `dfs.namenode.safemode.extension`：稳定时间，默认值30000毫秒，即30秒

+ 基本语法

  + **查看**安全模式状态

  ```shell
  bin/hdfs dfsadmin -safemode get
  ```

  + **进入**安全模式状态

  ```shell
  bin/hdfs dfsadmin -safemode enter
  ```

  + **离开**安全模式状态

  ```shell
  bin/hdfs dfsadmin -safemode leave
  ```

  + **等待**安全模式状态

  ```shell
  bin/hdfs dfsadmin -safemode wait
  ```

### 磁盘修复

+ 数据块损坏，进入安全模式，如何处理

## 3）慢磁盘监控

## 4）小文件归档

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

# 8. MapReduce生产经验

## MapReduce运行慢的原因

+ 计算机性能瓶颈：CPU、内存、磁盘、网络
+ I/O操作优化
  + 数据倾斜
  + Map运行时间太长，导致Reduce等待过久（？）
  + 小文件过多

## MapReduce常见调优参数

## 数据倾斜问题

+ 现象
  + 数据**频率**倾斜：某一个区域的数据量要远远大于其他区域
  + 数据**大小**倾斜——部分记录的大小远远大于平均值
+ 减少数据倾斜的方法
  + 检查是否空值过多造成的数据倾斜
    + 可直接过滤掉空值，若想保留空值就自定义分区，将空值加随机数打散，再二次聚合
  + 能在map阶段提前处理，最好先在Map阶段处理
    + Combiner、MapJoin
  + 设置多个reduce

# 9. Hadoop-Yarn生产经验

# 10. Hadoop综合调优

## 小文件优化

HDFS上每个文件都要在NameNode上创建对应的元数据，元数据的大小约为150byte。当小文件比较多的时候，就会产生很多的元数据文件，一方面会大量占用NameNode的内存空间，另一方面就是元数据文件过多，使得寻址索引速度变慢。

小文件过多，在进行MR计算时，会生成过多切片，需要启动过多的MapTask。每个MapTask处理的数据量小，导致MapTask的处理时间比启动时间还小，白白消耗资源。

**解决方案**

1. **在数据采集的时候，就将小文件或小批数据合成大文件再上传HDFS（数据源头）**

2. **Hadoop Archive（存储方向）**

   一个高效地将小文件放入HDFS块中的文件存档工具，能够将多个小文件打包成一个HAR文件，从而达到减少NameNode的内存使用

3. **CombineTextInputFormat（计算方向）**

   将多个小文件在切片过程中生成一个单独的切片或者少量的切片

4. **开启uber模式，实现JVM重用（计算方向）**

   默认情况下，每个Task任务都需要启动一个JVM来运行。如果Task任务计算的数据量很小，我们可以让同一个Job的多个Task运行在一个JVM中，不必为每个Task都开启一个JVM

   + 开启uber模式，修改`mapred-site.xml`，分发配置并重启

   ```xml
   <!--  开启uber模式，默认关闭 -->
   <property>
     	<name>mapreduce.job.ubertask.enable</name>
     	<value>true</value>
   </property>
   
   <!-- uber模式中最大的mapTask数量，可向下修改  --> 
   <property>
     	<name>mapreduce.job.ubertask.maxmaps</name>
     	<value>9</value>
   </property>
   <!-- uber模式中最大的reduce数量，可向下修改 -->
   <property>
     	<name>mapreduce.job.ubertask.maxreduces</name>
     	<value>1</value>
   </property>
   <!-- uber模式中最大的输入数据量，默认使用dfs.blocksize 的值，可向下修改 -->
   <property>
     	<name>mapreduce.job.ubertask.maxbytes</name>
     	<value></value>
   </property>
   ```

   
