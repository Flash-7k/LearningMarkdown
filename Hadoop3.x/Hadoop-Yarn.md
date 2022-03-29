# 1. 基础架构

Yarn是一个资源调度平台，负责为运算程序提供服务器运算资源，相当于一个分布式的操作系统平台，而MapReduce等运算程序则相当于运行于操作系统之上的应用程序。

组成：**ResourceManager、NodeManager、ApplicationMaster、Container**

+ **ResourceManager**
  + 处理客户端请求
  + 监控NodeManager
  + 启动或监控ApplicationMaster
  + 资源的分配与调度
+ **NodeManager**
  + 管理单个节点上的资源
  + 处理来自ResourceManager的命令
  + 处理来自ApplicationMaster的命令
+ **ApplicationMaster**
  + 为应用程序申请资源并分配给内部的任务
  + 任务的监控与容错
+ **Container**
  + 资源抽象，封装了某个节点上的多维度资源，如内存、CPU、磁盘、网络等

# 2. 工作机制

1. MR程序提交到客户端所在节点，`jar`包`main`方法中执行`job.waitForCompletion()`，启动YarnRunner
2. YarnRunner 向 ResourceManager 申请一个Application
3. ResourceManager 将该MR程序的资源路径返回给 YarnRunner ：`hdfs://.../staging`以及`application_id`
4. YarnRunner 将运行所需资源提交到HDFS上（`job.submit()`后生成的`Job.split`  `Job.xml`  `wc.jar`）
5. 资源提交完毕后，YarnRunner 向 ResourceManager 申请运行 mrAppMaster
6. ResourceManager 将用户请求初始化成一个 Task ，放入FIFO调度队列中等待
7. 分配一个较空闲的 NodeManager1 从 ResourceManager 领取 Task 任务
8. NodeManager1 创建容器 Container ，并产生 MrAppMaster
9. NodeManager1 从 HDFS 上拷贝`Job.split`  `Job.xml`到本地
10. NodeManager1 上 MrAppMaster 向 ResourceManager 申请容器运行 MapTask
11. 根据`Job.split`切片数量，分配相应数量的 NodeManager2 、NodeManager3 ，分别领取 Task 任务并创建容器 Container （也可以是同一个 NodeManager 创建两个 Container   ）
12. NodeManager1 上 MrAppMaster 向接收到到任务的 NodeManager2 、NodeManager3 发送程序启动脚本，这两个 NodeManager 分别启动 MapTask ，并对结果数据分区排序
13. NodeManager1 上 MrAppMaster 等待所有 MapTask 运行完毕后，向 ResourceManager 申请容器，运行 ReduceTask
14. 根据不同业务需求设置的 Reducer 数量，启动相应的容器，领取 ReduceTask，并向 MapTask 获取相应分区的数据
15. 程序运行完毕后，NodeManager1 上 MrAppMaster 会向 ResourceManager 申请注销自己

# 3. 作业提交全过程

（1）作业提交

​	第1步：Client调用`job.waitForCompletion`方法，向整个集群提交MapReduce作业。

​	第2步：Client向RM申请一个作业id。

​	第3步：RM给Client返回该job资源的提交路径和作业id。

​	第4步：Client提交jar包、切片信息和配置文件到指定的资源提交路径。

​	第5步：Client提交完资源后，向RM申请运行MrAppMaster。

（2）作业初始化

​	第6步：当RM收到Client的请求后，将该job添加到容量调度器中。

​	第7步：某一个空闲的NM领取到该Job。

​	第8步：该NM创建Container，并产生MrAppMaster。

​	第9步：下载Client提交的资源到本地。

（3）任务分配

​	第10步：MrAppMaster向RM申请运行多个MapTask任务资源。

​	第11步：RM将运行MapTask任务分配给另外两个NodeManager，另两个NodeManager分别领取任务并创建容器。

（4）任务运行

​	第12步：MR向两个接收到任务的NodeManager发送程序启动脚本，这两个NodeManager分别启动MapTask，MapTask对数据分区排序。

​	第13步：MrAppMaster等待所有MapTask运行完毕后，向RM申请容器，运行ReduceTask。

​	第14步：ReduceTask向MapTask获取相应分区的数据。

​	第15步：程序运行完毕后，MR会向RM申请注销自己。

（5）进度和状态更新

​	YARN中的任务将其进度和状态(包括counter)返回给应用管理器, 客户端每秒(通过`mapreduce.client.progressmonitor.pollinterval`设置)向应用管理器请求进度更新, 展示给用户。

（6）作业完成

除了向应用管理器请求作业进度外, 客户端每5秒都会通过调用`waitForCompletion()`来检查作业是否完成。时间间隔可以通过`mapreduce.client.completion.pollinterval`来设置。作业完成之后, 应用管理器和Container会清理工作状态。作业的信息会被作业历史服务器存储以备之后用户核查。

# 4. 调度器和调度算法

Hadoop作业调度器

+ FIFO调度器
+ 容量调度器（Capacity Scheduler）
+ 公平调度器（Fair Scheduler）

> Apache Hadoop3.1.3 默认的资源调度器是 Capacity Scheduler
>
> CDH Hadoop 框架默认调度器是 Fair Scheduler

具体配置详见：`yarn-default.xml`

```xml
<property>
    <description>The class to use as the resource scheduler.</description>
    <name>yarn.resourcemanager.scheduler.class</name>
	<value>org.apache.hadoop.yarn.server.resourcemanager.scheduler.capacity.CapacityScheduler</value>
</property>
```

## 先进先出调度器：FIFO

FIFO调度器（First In First Out）：单队列，根据提交作业的先后顺序，先来先服务。

+ 优点：简单易懂
+ 缺点：不支持多队列，生产环境很少使用

## 容量调度器：Capacity Scheduler

**容量调度器特点：**

+ 多队列
  + 每个队列可配置一定的资源量，每个队列采用FIFO调度策略
+ 容量保证
  + 管理员可为每个队列设置资源最低保证和资源使用上限
+ 灵活性
  + 如果一个队列中的资源有剩余，可以暂时共享给其他需要资源的队列
  + 一旦该队列中有新的应用程序提交，则借调的资源会归还给该队列
+ 多租户
  + 支持多用户共享集群和或应用程序同时运行
  + 为防止同一个用户的作用独占队列中资源，**对同一用户提交的作业最大所占资源量进行限定**

**容量调度器资源分配算法：**

+ 队列资源分配
  + 从root节点开始，使用深度优先算法，优先**选择资源占用率最低的队列**分配资源（即选择资源所需最小的队列）
+ 作业资源分配
  + 默认按照提交作业的**优先级**和**提交时间**顺序分配资源
+ 容器资源分配
  + 按照**容器的优先级**分配资源
  + 若优先级相同，按照**数据本地性原则**：
    + 任务和数据在同一节点
    + 任务和数据在同一机架
    + 任务和数据在不在同一节点也不在同一机架

## 公平调度器：Fair Scheduler

**公平调度器特点：**

+ 与容量调度器相同点
  + 多队列：支持多队列、多作业
  + 容量保证：管理员可为每个队列设置资源最低保证和资源使用上限
  + 灵活性：如果一个队列中的资源有剩余，可以暂时共享给其他需要资源的队列。一旦该队列中有新的应用程序提交，则借调的资源会归还给该队列
  + 多租户：支持多用户共享集群和或应用程序同时运行。为防止同一个用户的作用独占队列中资源，对同一用户提交的作业最大所占资源量进行限定
+ 与容量调度器不同点
  + 核心调度策略不同
    + 容量调度器：优先选择**资源利用率低**的队列
    + 公平调度器：优先选择对资源的**缺额比例大**的（**缺额：平均分配应得的资源-实际拥有的资源**）
  + 每个队列可以单独设置资源分配方式
    + 容量调度器：FIFO、DRF
    + 公平调度器：FIFO、DRF、**FAIR**

**公平调度器队列资源分配方式**

+ FIFO策略：选择FIFO则与容量调度器相同

+ DRF策略：Dominant Resource Fairness。资源多样，不止考虑内存，包括CPU、网络带宽等。分配更加灵活。

+ Fair策略（默认选择，适用于对并行度要求更高的场景）

  + 一种基于最大最小公平算法实现的资源多路复用方式。默认情况下，每个队列内部采用该方法分配资源。如果一个队列中有两个应用程序同时运行，则每个应用程序可得到1/2的资源；如果三个应用程序同时运行，则每个可得到1/3的资源
  + 具体资源分配流程与容量调度器一致：选择队列、作业、容器。每一步都按照公平策略分配
  + 饥饿优先
    + 都饥饿：资源分配比小者优先。相同，按照提交时间顺序
    + 都不饥饿：资源使用权重比小者优先。相同，按照提交时间顺序

  > 实际最小资源份额：`mindshare = Min（资源需求量，配置的最小资源）`
  >
  > 是否饥饿：`isNeedy = 资源使用量 < mindshare`
  >
  > 资源分配比：`mindshareRatio = 资源使用量 / Max（mindshare，1）`
  >
  > 资源使用权重比：`useToWeightRatio = 资源使用量 / 权重`

**公平调度器资源分配算法**

+ 队列资源分配

  + 需求：集群总资源100，共有三个队列，A需要20，B需要50，C需要30

  + 计算步骤

    + 100/3 = 33.33

      + A=33.33，多13.33

      + B=33.33，少16.67
      + C=33.33，多3.33

    + （13.33+3.33）/ 1 = 16.67

      + B = 33.33+16.67=50

+ 作业资源分配
  + 不加权（关注Job的个数）
    + 需求：队列总资源12个，共有四个Job，A需要1，B需要2，C需要6，D需要5
    + 计算步骤（资源/个数）
    + 12/4 = 3
      + A=3，多2个
      + B=3，多1个
      + C=3，少3个
      + D=3，少2个
    + （2+1）/ 2 = 1.5
      + C=3+1.5=4.5
      + D=3+1.5=4.5
    + 一直算到没有空闲资源为止
  + 加权（关注Job的权重）
    + 需求：队列总资源16个，共有四个Job，A需要4（权重5），B需要2（权重8），C需要10（权重1），D需要4（权重2）
    + 计算步骤（资源/权重）
    + 16/（5+8+1+2）=1
      + A=1*5，多1
      + B=1*8，多6
      + C=1*1，少9
      + D=1*2，少2
    + （1+6）/（1+2）=7/3
      + C=1+2.33=3.33，少6.67
      + D=2+4.66=6.66，多2.66
    + C=3.33+2.66=6
    + 一直算到没有空闲资源为止

# 5. 常用命令



# 6. 生产环境核心参数

+ **ResourceManager相关**
  + `yarn.resourcemanager.scheduler.dass`
    + **配置调度器，默认容量调度器**
  + `yarn.resourcemanager.scheduler.client.thread-count`
    + ResourceManager处理调度器请求的线程数量，默认50 
+ **NodeManager相关**
  + `yarn.nodemanager.resource.detect-hardware-capabilities`
    + 是否让yarn自己检测硬件进行配置，**默认false**
  + `yarn.nodemanager.resource.count-logical-processors-as-cores`
    + 是否将虚拟核数当作CPU核数，默认false
  + `yarn.nodemanager.resource.pcores-vcores-multiplier`
    + 虚拟核数和物理核数乘数，例如：4核8线程，该参数就应设为2，默认**1.0**
  + `client yarn.nodemanager.resource.memory-mb`
    + **NodeManager使用内存，默认8G（需根据情况修改）**
  + `yarn.nodemanager.resource.system-reserved-memory-mb`
    + NodeManager为系统保留多少内存
    + 以上二个参数配置一个即可
  + `yarn.nodemanager.resource.cpu-vcores`
    + **NodeManager使用CPU核数，默认8个**
  + `yarn.nodemanager.pmem-check-enabled`
    + **是否开启物理内存检查限制container，默认打开（需关闭）**
  + `yarn.nodemanager.vmem-check-enabled`
    + 是否开启虚拟内存检查限制container，**默认打开**
  + `yarn.nodemanager.vmem-pmem-ratio`
    + **虚拟内存物理内存比例，默认2.1**
+ **Container相关（需根据情况修改）**
  + `yarn.scheduler.minimum-allocation-mb`
    + 容器最小内存，默认1G
  + + 容器最大内存，默认8G
  + `yarn.scheduler.minimum-allocation-vcores`
    + 容器最小CPU核数，默认1个
  + `yarn.scheduler.maximum-allocation-vcores`
    + 容器最大CPU核数，默认4个

## 参数配置案例

+ **注意：配置前先保持快照，以防崩溃，便于恢复**

+ 需求：从1G数据中，统计每个单词出现次数。服务器3台，每台配置4G内存，4核CPU，4线程。
+ 需求分析：1G/128M=8个MapTask，1个ReduceTask，一个MrAppMaster（10/3 = 4+3+3）
+ 修改`yarn-site.xml`，并分发到各个节点

```xml
<!-- 选择调度器，默认容量调度器 -->
<property>
	<description>The class to use as the resource scheduler.</description>
	<name>yarn.resourcemanager.scheduler.class</name>
	<value>org.apache.hadoop.yarn.server.resourcemanager.scheduler.capacity.CapacityScheduler</value>
</property>

<!-- ResourceManager处理调度器请求的线程数量,默认50；如果提交的任务数大于50，可以增加该值，但是不能超过3台 * 4线程 = 12线程（去除其他应用程序实际不能超过8） -->
<property>
	<description>Number of threads to handle scheduler interface.</description>
	<name>yarn.resourcemanager.scheduler.client.thread-count</name>
	<value>8</value>
</property>

<!-- 是否让yarn自动检测硬件进行配置，默认是false，如果该节点有很多其他应用程序，建议手动配置。如果该节点没有其他应用程序，可以采用自动 -->
<property>
	<description>Enable auto-detection of node capabilities such as
	memory and CPU.
	</description>
	<name>yarn.nodemanager.resource.detect-hardware-capabilities</name>
	<value>false</value>
</property>

<!-- 是否将虚拟核数当作CPU核数，默认是false，采用物理CPU核数 -->
<property>
	<description>Flag to determine if logical processors(such as
	hyperthreads) should be counted as cores. Only applicable on Linux
	when yarn.nodemanager.resource.cpu-vcores is set to -1 and
	yarn.nodemanager.resource.detect-hardware-capabilities is true.
	</description>
	<name>yarn.nodemanager.resource.count-logical-processors-as-cores</name>
	<value>false</value>
</property>

<!-- 虚拟核数和物理核数乘数，默认是1.0 -->
<property>
	<description>Multiplier to determine how to convert phyiscal cores to
	vcores. This value is used if yarn.nodemanager.resource.cpu-vcores
	is set to -1(which implies auto-calculate vcores) and
	yarn.nodemanager.resource.detect-hardware-capabilities is set to true. The	number of vcores will be calculated as	number of CPUs * multiplier.
	</description>
	<name>yarn.nodemanager.resource.pcores-vcores-multiplier</name>
	<value>1.0</value>
</property>

<!-- NodeManager使用内存数，默认8G，修改为4G内存 -->
<property>
	<description>Amount of physical memory, in MB, that can be allocated 
	for containers. If set to -1 and
	yarn.nodemanager.resource.detect-hardware-capabilities is true, it is
	automatically calculated(in case of Windows and Linux).
	In other cases, the default is 8192MB.
	</description>
	<name>yarn.nodemanager.resource.memory-mb</name>
	<value>4096</value>
</property>

<!-- nodemanager的CPU核数，不按照硬件环境自动设定时默认是8个，修改为4个 -->
<property>
	<description>Number of vcores that can be allocated
	for containers. This is used by the RM scheduler when allocating
	resources for containers. This is not used to limit the number of
	CPUs used by YARN containers. If it is set to -1 and
	yarn.nodemanager.resource.detect-hardware-capabilities is true, it is
	automatically determined from the hardware in case of Windows and Linux.
	In other cases, number of vcores is 8 by default.</description>
	<name>yarn.nodemanager.resource.cpu-vcores</name>
	<value>4</value>
</property>

<!-- 容器最小内存，默认1G -->
<property>
	<description>The minimum allocation for every container request at the RM	in MBs. Memory requests lower than this will be set to the value of this	property. Additionally, a node manager that is configured to have less memory	than this value will be shut down by the resource manager.
	</description>
	<name>yarn.scheduler.minimum-allocation-mb</name>
	<value>1024</value>
</property>

<!-- 容器最大内存，默认8G，修改为2G -->
<property>
	<description>The maximum allocation for every container request at the RM	in MBs. Memory requests higher than this will throw an	InvalidResourceRequestException.
	</description>
	<name>yarn.scheduler.maximum-allocation-mb</name>
	<value>2048</value>
</property>

<!-- 容器最小CPU核数，默认1个 -->
<property>
	<description>The minimum allocation for every container request at the RM	in terms of virtual CPU cores. Requests lower than this will be set to the	value of this property. Additionally, a node manager that is configured to	have fewer virtual cores than this value will be shut down by the resource	manager.
	</description>
	<name>yarn.scheduler.minimum-allocation-vcores</name>
	<value>1</value>
</property>

<!-- 容器最大CPU核数，默认4个，修改为2个 -->
<property>
	<description>The maximum allocation for every container request at the RM	in terms of virtual CPU cores. Requests higher than this will throw an
	InvalidResourceRequestException.</description>
	<name>yarn.scheduler.maximum-allocation-vcores</name>
	<value>2</value>
</property>

<!-- 虚拟内存检查，默认打开，修改为关闭 -->
<property>
	<description>Whether virtual memory limits will be enforced for
	containers.</description>
	<name>yarn.nodemanager.vmem-check-enabled</name>
	<value>false</value>
</property>

<!-- 虚拟内存和物理内存设置比例,默认2.1 -->
<property>
	<description>Ratio between virtual memory to physical memory when	setting memory limits for containers. Container allocations are	expressed in terms of physical memory, and virtual memory usage	is allowed to exceed this allocation by this ratio.
	</description>
	<name>yarn.nodemanager.vmem-pmem-ratio</name>
	<value>2.1</value>
</property>
```

# 7. 配置多队列调度器

## 配置多队列的容量调度器

+ 需求
  + default队列占总内存的40%，最大资源容量占总资源的60%。hive队列占总内存的60%，最大资源容量占总资源的80%
  + 配置队列优先级
+ 修改`capacity-scheduler.xml`：拷贝到Windows下，使用notepad修改方便

```xml
<!-- 指定多队列，增加hive队列 -->
<property>
    <name>yarn.scheduler.capacity.root.queues</name>
    <value>default,hive</value>
    <description>
      The queues at the this level (root is the root queue).
    </description>
</property>

<!-- 降低default队列资源额定容量为40%，默认100% -->
<property>
    <name>yarn.scheduler.capacity.root.default.capacity</name>
    <value>40</value>
</property>

<!-- 降低default队列资源最大容量为60%，默认100% -->
<property>
    <name>yarn.scheduler.capacity.root.default.maximum-capacity</name>
    <value>60</value>
</property>
```

+ 为新队列添加必要属性：复制default内容，替换为hive

```xml
<!-- 指定hive队列的资源额定容量 -->
<property>
    <name>yarn.scheduler.capacity.root.hive.capacity</name>
    <value>60</value>
</property>

<!-- 用户最多可以使用队列多少资源，1表示 -->
<property>
    <name>yarn.scheduler.capacity.root.hive.user-limit-factor</name>
    <value>1</value>
</property>

<!-- 指定hive队列的资源最大容量 -->
<property>
    <name>yarn.scheduler.capacity.root.hive.maximum-capacity</name>
    <value>80</value>
</property>

<!-- 启动hive队列 -->
<property>
    <name>yarn.scheduler.capacity.root.hive.state</name>
    <value>RUNNING</value>
</property>

<!-- 哪些用户有权向队列提交作业 -->
<property>
    <name>yarn.scheduler.capacity.root.hive.acl_submit_applications</name>
    <value>*</value>
</property>

<!-- 哪些用户有权操作队列，管理员权限（查看/杀死） -->
<property>
    <name>yarn.scheduler.capacity.root.hive.acl_administer_queue</name>
    <value>*</value>
</property>

<!-- 哪些用户有权配置提交任务优先级 -->
<property>
    <name>yarn.scheduler.capacity.root.hive.acl_application_max_priority</name>
    <value>*</value>
</property>

<!-- 任务的超时时间设置：yarn application -appId appId -updateLifetime Timeout
参考资料：https://blog.cloudera.com/enforcing-application-lifetime-slas-yarn/ -->

<!-- 如果application指定了超时时间，则提交到该队列的application能够指定的最大超时时间不能超过该值。 
-->
<property>
    <name>yarn.scheduler.capacity.root.hive.maximum-application-lifetime</name>
    <value>-1</value>
</property>

<!-- 如果application没指定超时时间，则用default-application-lifetime作为默认值 -->
<property>
    <name>yarn.scheduler.capacity.root.hive.default-application-lifetime</name>
    <value>-1</value>
</property>
```

+ 重启Yarn或者刷新队列，就可看到两条队列

```shell
yarn rmadmin -refreshQueues
```

+ 专门使用新的hive队列

  + hadoop jar方式

  ```shell
  hadoop jar share/hadoop/mapreduce/hadoop-mapreduce-examples-3.1.3.jar wordcount -D mapreduce.job.queuename=hive /input /output
  ```

  > **注意： -D表示运行时改变参数值**

  + 在jar包的Driver类设置（默认任务提交到default队列）

  ```java
  public class WcDrvier {
  
      public static void main(String[] args) throws IOException, ClassNotFoundException, InterruptedException {
  
          Configuration conf = new Configuration();
          
  		// 设置走hive
          conf.set("mapreduce.job.queuename","hive");
  
          //1. 获取一个Job实例
          Job job = Job.getInstance(conf);
  
          ... ...
  
          //6. 提交Job
          boolean b = job.waitForCompletion(true);
          System.exit(b ? 0 : 1);
      }
  }
  ```

### 任务优先级

容量调度器，支持任务优先级的配置。在资源紧张时，优先级高的任务将优先获取资源。默认情况，Yarn将所有任务的优先级限制为0，若想使用任务的优先级功能，须开放该限制。

+ 修改`yarn-site.xml`文件，增加以下参数

```xml
<property>
    <name>yarn.cluster.max-application-priority</name>
    <value>5</value>
</property>
```

+ 分发配置，并重启Yarn
+ 模拟资源紧张环境，可连续提交以下任务，直到新提交的任务申请不到资源为止

```shell
hadoop jar /opt/module/hadoop-3.1.3/share/hadoop/mapreduce/hadoop-mapreduce-examples-3.1.3.jar pi 5 2000000
```

+ 再次重新提交优先级高的任务

```shell
hadoop jar /opt/module/hadoop-3.1.3/share/hadoop/mapreduce/hadoop-mapreduce-examples-3.1.3.jar pi  -D mapreduce.job.priority=5 5 2000000
```

+ 可以通过以下命令修改正在执行的任务的优先级

```shell
yarn application -appID <ApplicationID> -updatePriority <优先级>
```

## 配置公平调度器

+ 需求：
  + 创建test和flash7k队列
  + 若用户提交任务时指定队列，则任务提交到指定队列运行
  + 若未指定队列，test用户提交的任务到`root.group.test`队列运行，flash7k用户提交的任务到`root.group.flash7k`
+ 文件
  + `yarn-site.xml`
  + `fair-scheduler.xml`（公平调度器队列分配文件，文件名可自定义）
+ 参考资料
  + 配置文件参考资料：https://hadoop.apache.org/docs/r3.1.3/hadoop-yarn/hadoop-yarn-site/FairScheduler.html
  + 任务队列放置规则参考资料：https://blog.cloudera.com/untangling-apache-hadoop-yarn-part-4-fair-scheduler-queue-basics/
+ 修改`yarn-site.xml`

```xml
<property>
    <name>yarn.resourcemanager.scheduler.class</name>
    <value>org.apache.hadoop.yarn.server.resourcemanager.scheduler.fair.FairScheduler</value>
    <description>配置使用公平调度器</description>
</property>

<property>
    <name>yarn.scheduler.fair.allocation.file</name>
    <value>/opt/module/hadoop-3.1.3/etc/hadoop/fair-scheduler.xml</value>
    <description>指明公平调度器队列分配配置文件</description>
</property>

<property>
    <name>yarn.scheduler.fair.preemption</name>
    <value>false</value>
    <description>禁止队列间资源抢占</description>
</property>
```

+ 配置`fair-scheduler.xml`

```xml
<?xml version="1.0"?>
<allocations>
  <!-- 单个队列中Application Master占用资源的最大比例,取值0-1 ，企业一般配置0.1 -->
  <queueMaxAMShareDefault>0.5</queueMaxAMShareDefault>
  <!-- 单个队列最大资源的默认值 test atguigu default -->
  <queueMaxResourcesDefault>4096mb,4vcores</queueMaxResourcesDefault>

  <!-- 增加一个队列test -->
  <queue name="test">
    <!-- 队列最小资源 -->
    <minResources>2048mb,2vcores</minResources>
    <!-- 队列最大资源 -->
    <maxResources>4096mb,4vcores</maxResources>
    <!-- 队列中最多同时运行的应用数，默认50，根据线程数配置 -->
    <maxRunningApps>4</maxRunningApps>
    <!-- 队列中Application Master占用资源的最大比例 -->
    <maxAMShare>0.5</maxAMShare>
    <!-- 该队列资源权重,默认值为1.0 -->
    <weight>1.0</weight>
    <!-- 队列内部的资源分配策略 -->
    <schedulingPolicy>fair</schedulingPolicy>
  </queue>
  <!-- 增加一个队列flash7k -->
  <queue name="flash7k" type="parent">
    <!-- 队列最小资源 -->
    <minResources>2048mb,2vcores</minResources>
    <!-- 队列最大资源 -->
    <maxResources>4096mb,4vcores</maxResources>
    <!-- 队列中最多同时运行的应用数，默认50，根据线程数配置 -->
    <maxRunningApps>4</maxRunningApps>
    <!-- 队列中Application Master占用资源的最大比例 -->
    <maxAMShare>0.5</maxAMShare>
    <!-- 该队列资源权重,默认值为1.0 -->
    <weight>1.0</weight>
    <!-- 队列内部的资源分配策略 -->
    <schedulingPolicy>fair</schedulingPolicy>
  </queue>

  <!-- 任务队列分配策略,可配置多层规则,从第一个规则开始匹配,直到匹配成功 -->
  <queuePlacementPolicy>
    <!-- 提交任务时指定队列,如未指定提交队列,则继续匹配下一个规则; false表示：如果指定队列不存在,不允许自动创建-->
    <rule name="specified" create="false"/>
    <!-- 提交到root.group.username队列,若root.group不存在,不允许自动创建；若root.group.user不存在,允许自动创建 -->
    <rule name="nestedUserQueue" create="true">
        <rule name="primaryGroup" create="false"/>
    </rule>
    <!-- 最后一个规则必须为reject或者default。Reject表示拒绝创建提交失败，default表示把任务提交到default队列 -->
    <rule name="reject" />
  </queuePlacementPolicy>
</allocations>
```

+ 分发配置并重启Yarn
+ 提交任务时指定队列：任务到指定的root.test队列

```shell
hadoop jar /opt/module/hadoop-3.1.3/share/hadoop/mapreduce/hadoop-mapreduce-examples-3.1.3.jar pi -Dmapreduce.job.queuename=root.test 1 1
```

+ 提交任务时不指定队列：任务到默认root.flash7k.flash7k队列

```shell
hadoop jar /opt/module/hadoop-3.1.3/share/hadoop/mapreduce/hadoop-mapreduce-examples-3.1.3.jar pi 1 1
```

# Tool接口案例



