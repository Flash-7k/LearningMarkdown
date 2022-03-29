# Hadoop-MapReduce

## 第1章 MapReduce概述

### 1.1 MapReduce定义

+ **定义**：**分布式运算程序**编程框架，用户开发“基于Hadoop的数据分析应用”的核心框架
+ **核心功能**：将 **用户编写的业务逻辑代码** 和 **自带默认组件** 整合成一个完整的 **分布式运算程序** ，并发运行在Hadoop集群上

### 1.2 MapReduce优缺点

+ **优点**
  + **易于编程**
    + 简单的实现一些接口，就可以完成一个分布式运算程序
  + **良好的扩展性**
    + 可简单的增加机器，解决计算资源不足问题
  + **高容错性**
    + 自动将挂掉节点上的计算任务转移到另一个节点上运行，不至于任务运行失败
  + **适合PB级以上海量数据的离线处理**
    + 可实现上千台服务器集群并发工作，提供数据处理能力
+ **缺点**
  + **不擅长实时计算**
    + 无法在毫秒内或者秒级内返回结果
  + **不擅长流式计算**
    + 输入数据集是静态的，不能动态变化
  + **不擅长DAG（有向无环图）计算**
    + 多个应用程序存在依赖关系，后一个应用程序的输入为前一个的输出
    + 进行DAG计算，由于每个MapReduce作业的输出结果都会写入到磁盘，会造成大量的磁盘IO，导致性能非常的低下（Spark利用内存解决）

### 1.3 MapReduce核心思想

**分布式的运算程序往往需要分成至少2个阶段：Map和Reduce**

+ **MapTask并发实例**：完全并行运行，互不相干
+ **ReduceTask并发实例**：完全并行运行，互不相干。*但是他们的数据依赖于上一个阶段的所有MapTask并发实例的输出*

> **MapReduce编程模型只能包含一个Map阶段和一个Reduce阶段**
>
> **如果用户的业务逻辑非常复杂，那就只能多个MapReduce程序，串行运行**

### 1.4 MapReduce进程

**一个完整的MapReduce程序在分布式运行时有三类实例进程：MrAppMaster、MapTask、ReduceTask**

+ **MrAppMaster**：负责整个程序的过程调度及状态协调
+ **MapTask**：负责Map阶段的整个数据处理流程
+ **ReduceTask**：负责Reduce阶段的整个数据处理流程

### 1.5 官方WordCount源码

采用反编译工具反编译源码，发现WordCount案例有**Map类、Reduce类和驱动类**。且数据的类型是**Hadoop自身封装的序列化类型**

### 1.6 常用数据序列化类型

Java类型|Hadoop Writable 类型
:-|:--
Boolean|BooleanWritable
Byte|ByteWritable
Int|IntWritable
Float|FloatWritable
Long|LongWritable
Double|DoubleWritable
**String**|**Text**
Map|MapWritable
Array|ArrayWritable
Null|NullWritable

### 1.7 MapReduce编程规范

**用户编写的程序分成三个部分：Mapper、Reducer和Driver**

1. **Mapper**
   + 用户自定义的Mapper要继承**org.apache.hadoop.mapreduce.Mapper**
   + 输入数据是**自定义<K,V>对**的形式
   + 业务逻辑写在**map()**方法中
   + 输出数据是**自定义<K,V>对**的形式
   + map()方法（MapTask进程）对**每个输入**的<K,V>调用一次
2. **Reduce**
   + 用户自定义的Reducer要继承**org.apache.hadoop.mapreduce.Reducer**
   + 输入数据对应**Mapper输出的<K,V>对**
   + 业务逻辑写在**reduce()**方法中
   + reduce()方法（ReduceTask进程）对**每一组相同K**的<K,V>调用一次
3. **Drive**
   + 相当于YARN集群的客户端，用于提交整个程序到YARN集群
   + 提交job对象：封装了MapReduce程序相关运行参数

### 1.8 WordCount案例实操

#### 1.本地测试

+ **需求**：在给定的文本文件中统计输出每一个单词出现的总次数

+ **分析**

  + **Mapper**
    + 将Text文本内容转化为String
    + 根据空格将一行内容切分为单词
    + 输出<单词，1>
  + **Reduce**
    + 汇总各个key的个数：<单词，1>
    + 输出该key的总次数：<单词，n>
  + **Driver**
    + 获取配置信息，获取job对象实例
    + 指定本程序的jar包所在的本地路径
    + 关联Mapper/Reducer业务类
    + 指定Mapper输出数据kv类型
    + 指定Reducer输出数据kv类型
    + 指定job的输入原始文件所在目录
    + 指定job的输出结果文件所在目录
    + 提交作业job

+ **环境准备**

  + 创建Maven工程
  + 在pom.xml引入依赖

  ```xml
  <dependencies>
      <dependency>
          <groupId>org.apache.hadoop</groupId>
          <artifactId>hadoop-client</artifactId>
          <version>3.1.3</version>
      </dependency>
      
      <dependency>
          <groupId>junit</groupId>
          <artifactId>junit</artifactId>
          <version>4.12</version>
      </dependency>
      
      <dependency>
          <groupId>org.slf4j</groupId>
          <artifactId>slf4j-log4j12</artifactId>
          <version>1.7.30</version>
      </dependency>
  </dependencies>
  ```

  + 在项目 src/main/resources 目录下，新建“log4j.properties”，在文件中填入

  ```properties
  log4j.rootLogger=INFO, stdout  
  log4j.appender.stdout=org.apache.log4j.ConsoleAppender  
  log4j.appender.stdout.layout=org.apache.log4j.PatternLayout  
  log4j.appender.stdout.layout.ConversionPattern=%d %p [%c] - %m%n  
  log4j.appender.logfile=org.apache.log4j.FileAppender  
  log4j.appender.logfile.File=target/spring.log  
  log4j.appender.logfile.layout=org.apache.log4j.PatternLayout  
  log4j.appender.logfile.layout.ConversionPattern=%d %p [%c] - %m%n
  ```

  + 创建包 com.flash7k.mapreduce.wordcount

+ **编写程序**

  + **Mapper**

  ```java
  package com.flash7k.mapreduce.wordcount;
  import java.io.IOException;
  import org.apache.hadoop.io.IntWritable;
  import org.apache.hadoop.io.LongWritable;
  import org.apache.hadoop.io.Text;
  import org.apache.hadoop.mapreduce.Mapper;
  
  public class WordCountMapper extends Mapper<LongWritable, Text, Text, IntWritable>{
  	
  	Text k = new Text();
  	IntWritable v = new IntWritable(1);
  	
  	@Override
  	protected void map(LongWritable key, Text value, Context context)	throws IOException, InterruptedException {
  		
  		// 1 获取一行文本内容，并将Text转换为String
  		String line = value.toString();
  		
  		// 2 切割
  		String[] words = line.split(" ");
  		
  		// 3 输出
  		for (String word : words) {
  			k.set(word);
  			context.write(k, v);
  		}
  	}
  }
  ```

  + **Reducer**

  ```java
  package com.flash7k.mapreduce.wordcount;
  import java.io.IOException;
  import org.apache.hadoop.io.IntWritable;
  import org.apache.hadoop.io.Text;
  import org.apache.hadoop.mapreduce.Reducer;
  
  public class WordCountReducer extends Reducer<Text, IntWritable, Text, IntWritable>{
  
  	int sum;
  	IntWritable v = new IntWritable();
  
  	@Override
  	protected void reduce(Text key, Iterable<IntWritable> values,Context context) throws IOException, InterruptedException {
  		
  		// 1 累加求和
  		sum = 0;
  		for (IntWritable count : values) {
  			sum += count.get();
  		}
  		
  		// 2 输出
          v.set(sum);
  		context.write(key,v);
  	}
  }
  ```

  + **Driver**

  ```java
  package com.flash7k.mapreduce.wordcount;
  import java.io.IOException;
  import org.apache.hadoop.conf.Configuration;
  import org.apache.hadoop.fs.Path;
  import org.apache.hadoop.io.IntWritable;
  import org.apache.hadoop.io.Text;
  import org.apache.hadoop.mapreduce.Job;
  import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
  import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;
  
  public class WordCountDriver {
  
  	public static void main(String[] args) throws IOException, ClassNotFoundException, InterruptedException {
  
  		// 1 获取配置信息以及获取job对象
  		Configuration conf = new Configuration();
  		Job job = Job.getInstance(conf);
  
  		// 2 关联本Driver程序的jar
  		job.setJarByClass(WordCountDriver.class);
  
  		// 3 关联Mapper和Reducer的jar
  		job.setMapperClass(WordCountMapper.class);
  		job.setReducerClass(WordCountReducer.class);
  
  		// 4 设置Mapper输出的kv类型
  		job.setMapOutputKeyClass(Text.class);
  		job.setMapOutputValueClass(IntWritable.class);
  
  		// 5 设置最终输出kv类型
  		job.setOutputKeyClass(Text.class);
  		job.setOutputValueClass(IntWritable.class);
  		
  		// 6 设置输入和输出路径
          // arg[0]: hadoop集群的第1个参数，输入文件路径
          // arg[1]: hadoop集群的第2个参数，输出文件路径
  		FileInputFormat.setInputPaths(job, new Path(args[0]));
  		FileOutputFormat.setOutputPath(job, new Path(args[1]));
  
  		// 7 提交job
  		boolean result = job.waitForCompletion(true);
  		System.exit(result ? 0 : 1);
  	}
  }
  ```

+ **本地测试**
  + 需首先配置好HADOOP_HOME变量以及Windows运行依赖
  + 在IDEA/Eclipse上运行程序

#### 2. 提交到集群测试

+ **用Maven打jar包，在pom.xml引入打包插件依赖**

```xml
<build>
    <plugins>
        <plugin>
            <artifactId>maven-compiler-plugin</artifactId>
            <version>3.6.1</version>
            <configuration>
                <source>1.8</source>
                <target>1.8</target>
            </configuration>
        </plugin>
        <plugin>
            <artifactId>maven-assembly-plugin</artifactId>
            <configuration>
                <descriptorRefs>
                    <descriptorRef>jar-with-dependencies</descriptorRef>
                </descriptorRefs>
            </configuration>
            <executions>
                <execution>
                    <id>make-assembly</id>
                    <phase>package</phase>
                    <goals>
                        <goal>single</goal>
                    </goals>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>

```

+ **将不带依赖的jar包命名为wc.jar，并拷贝到Hadoop集群的/opt/module/hadoop-3.1.3路径**
+ **启动Hadoop集群**
+ **执行WordCount程序**

> **语法：hadoop jar jar包命 Driver全类名 输入文件路径 输出文件路径**
>
> **注意：输出文件路径不能已存在**

```linux
[flash7k@hadoop102 hadoop-3.1.3]$ hadoop jar  wc.jar com.flash7k.mapreduce.wordcount.WordCountDriver /user/flash7k/input /user/flash7k/output
```

## 第2章 Hadoop序列化

### 2.1 序列化概述

* **定义**
  * **序列化**：把**内存中的对象**，转换成**字节序列**（或其他数据传输协议）以便于存储到**磁盘（持久化）**和**网络传输**
  * **反序列化**：将收到**字节序列**（或其他数据传输协议）或者是**磁盘的持久化数据**，转换成内存中的对象
* **功能**
  * 持久化存储内存中的对象，便于将对象发送到远程计算机
* **Hadoop序列化与Java序列化比较**
  * Java：Serializable，重量级序列化框架。一个对象被序列化后，会附带很多额外的信息（各种校验信息，Header，继承体系等），不便于在网络中高效传输
  * Hadoop：Writable
* **Hadoop序列化特点**
  * **紧凑 ：**高效使用存储空间
  * **快速：**读写数据的额外开销小
  * **互操作：**支持多语言的交互

### 2.2 自定义Bean对象实现序列化接口（Writable）

**常用的基本序列化类型不能满足所有需求，需要自定义Bean对象，在Hadoop框架内部传递**

**序列化步骤**

+ Bean对象必须实现Writable接口
+ 空参构造：反序列化时，需要反射调用空参构造函数

```java
public FlowBean() {
	super();
}
```

+ 重写序列化方法

```java
@Override
public void write(DataOutput out) throws IOException {
	out.writeLong(upFlow);
	out.writeLong(downFlow);
	out.writeLong(sumFlow);
}
```

+ 重写反序列化方法

```java
@Override
public void readFields(DataInput in) throws IOException {
	upFlow = in.readLong();
	downFlow = in.readLong();
	sumFlow = in.readLong();
}
```

> **注意：反序列化的顺序和序列化的顺序完全一致**

+ 如果需要将自定义Bean放在key中传输，则还需要实现Comparable接口

> **原因：MapReduce框中的Shuffle过程要求对key必须能排序**

```java
@Override
public int compareTo(FlowBean o) {
	// 倒序排列，从大到小
	return this.sumFlow > o.getSumFlow() ? -1 : 1;
}
```

### 2.3 序列化案例实操

+ 需求：统计每个手机号耗费的总上行流量、总下行流量、总流量

+ 分析

  + Map
    + 读取一行数据，切分字段
    + 抽取手机号、上行流量、下行流量
    + 以手机号为key，bean对象为value输出
    + bean对象传输必须实现序列化接口
  + Reduce
    + 累加上行流量和下行流量得到总流量

+ 编写MapReduce程序

  + **编写Bean对象**

  ```java
  package com.flash7k.mapreduce.writable;
  
  import org.apache.hadoop.io.Writable;
  import java.io.DataInput;
  import java.io.DataOutput;
  import java.io.IOException;
  
  //1 继承Writable接口
  public class FlowBean implements Writable {
  
      private long upFlow; //上行流量
      private long downFlow; //下行流量
      private long sumFlow; //总流量
  
      //2 提供无参构造
      public FlowBean() {
      }
  
      //3 提供三个参数的getter和setter方法
      public long getUpFlow() {
          return upFlow;
      }
  
      public void setUpFlow(long upFlow) {
          this.upFlow = upFlow;
      }
  
      public long getDownFlow() {
          return downFlow;
      }
  
      public void setDownFlow(long downFlow) {
          this.downFlow = downFlow;
      }
  
      public long getSumFlow() {
          return sumFlow;
      }
  
      public void setSumFlow(long sumFlow) {
          this.sumFlow = sumFlow;
      }
  
      public void setSumFlow() {
          this.sumFlow = this.upFlow + this.downFlow;
      }
  
      //4 实现序列化和反序列化方法,注意顺序一定要保持一致
      @Override
      public void write(DataOutput dataOutput) throws IOException {
          dataOutput.writeLong(upFlow);
          dataOutput.writeLong(downFlow);
          dataOutput.writeLong(sumFlow);
      }
  
      @Override
      public void readFields(DataInput dataInput) throws IOException {
          this.upFlow = dataInput.readLong();
          this.downFlow = dataInput.readLong();
          this.sumFlow = dataInput.readLong();
      }
  
      //5 重写ToString
      @Override
      public String toString() {
          return upFlow + "\t" + downFlow + "\t" + sumFlow;
      }
  }
  ```

  + 编写Mapper

  ```java
  package com.flash7k.mapreduce.writable;
  
  import org.apache.hadoop.io.LongWritable;
  import org.apache.hadoop.io.Text;
  import org.apache.hadoop.mapreduce.Mapper;
  import java.io.IOException;
  
  public class FlowMapper extends Mapper<LongWritable, Text, Text, FlowBean> {
      private Text outK = new Text();
      private FlowBean outV = new FlowBean();
  
      @Override
      protected void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException {
  
          //1 获取一行数据,转成字符串
          String line = value.toString();
  
          //2 切割数据
          String[] split = line.split("\t");
  
          //3 抓取我们需要的数据:手机号,上行流量,下行流量
          String phone = split[1];
          String up = split[split.length - 3];
          String down = split[split.length - 2];
  
          //4 封装outK outV
          outK.set(phone);
          outV.setUpFlow(Long.parseLong(up));
          outV.setDownFlow(Long.parseLong(down));
          outV.setSumFlow();
  
          //5 写出outK outV
          context.write(outK, outV);
      }
  }
  ```

  + 编写Reducer类

  ```java
  package com.flash7k.mapreduce.writable;
  
  import org.apache.hadoop.io.Text;
  import org.apache.hadoop.mapreduce.Reducer;
  import java.io.IOException;
  
  public class FlowReducer extends Reducer<Text, FlowBean, Text, FlowBean> {
      private FlowBean outV = new FlowBean();
      @Override
      protected void reduce(Text key, Iterable<FlowBean> values, Context context) throws IOException, InterruptedException {
  
          long totalUp = 0;
          long totalDown = 0;
  
          //1 遍历values,将其中的上行流量,下行流量分别累加
          for (FlowBean flowBean : values) {
              totalUp += flowBean.getUpFlow();
              totalDown += flowBean.getDownFlow();
          }
  
          //2 封装outKV
          outV.setUpFlow(totalUp);
          outV.setDownFlow(totalDown);
          outV.setSumFlow();
  
          //3 写出outK outV
          context.write(key,outV);
      }
  }
  ```

  + 编写Driver驱动类

  ```java
  package com.flash7k.mapreduce.writable;
  
  import org.apache.hadoop.conf.Configuration;
  import org.apache.hadoop.fs.Path;
  import org.apache.hadoop.io.Text;
  import org.apache.hadoop.mapreduce.Job;
  import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
  import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;
  import java.io.IOException;
  
  public class FlowDriver {
      public static void main(String[] args) throws IOException, ClassNotFoundException, InterruptedException {
  
          //1 获取job对象
          Configuration conf = new Configuration();
          Job job = Job.getInstance(conf);
  
          //2 关联本Driver类
          job.setJarByClass(FlowDriver.class);
  
          //3 关联Mapper和Reducer
          job.setMapperClass(FlowMapper.class);
          job.setReducerClass(FlowReducer.class);
          
  		//4 设置Map端输出KV类型
          job.setMapOutputKeyClass(Text.class);
          job.setMapOutputValueClass(FlowBean.class);
          
  		//5 设置程序最终输出的KV类型
          job.setOutputKeyClass(Text.class);
          job.setOutputValueClass(FlowBean.class);
          
  		//6 设置程序的输入输出路径
          FileInputFormat.setInputPaths(job, new Path("D:\\inputflow"));
          FileOutputFormat.setOutputPath(job, new Path("D:\\flowoutput"));
          
  		//7 提交Job
          boolean b = job.waitForCompletion(true);
          System.exit(b ? 0 : 1);
      }
  }
  ```

## 第3章 MapReduce框架原理

### 3.1 InputFormat数据输入

#### 3.1.1 MapTask并行度决定机制

**MapTask的并行度决定Map阶段的任务处理并发度，进而影响到整个Job的处理速度**

+ **MapTask并行度决定机制**
  + **数据块**：HDFS在**物理上**把数据分块。数据块是HDFS存储数据单位
  + **数据切片**：MapRduce在**逻辑上**把数据分块。数据切片是MapReduce程序计算输入数据的单位，一个切片对应启动一个MapTask
  + **决定机制**
    + 一个Job的MapTask由Client在提交Job时的切片数决定
    + 一个切片对应一个MapTask
    + 默认情况下，切片大小=数据块大小=BlockSize
    + 切片时不考虑数据集整体，逐个针对每个文件单独切片

#### 3.1.2 Job提交流程和切片（源码解析）

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
  + Driver：Job.waitForCompletion(true)
  + Job.submit()
  + JobSubmiter：Cluster成员，proxy -> 判断是MR是在本地还是YARN
    + 创建stagingDir
      + 本地：File://.../staging
      + YARN：hdfs://.../staging
    + 获取jobid
      + 本地：File://.../staging/jobid
      + YARN：hdfs://.../staging/jobid
    + 调用FileInputFormat.getSplits()获取切片规划，并序列化成文件：**Job.split**
    + 将Job相关参数写到文件：**Job.xml**
    + 如果是YARN，还需要获取Job的jar包：**xxx.jar**

#### 3.1.3 FileInputFormat切片机制

+ **切片机制**
  + 简单地按照文件的内容长度进行切片
  + 切片大小默认等于Block大小
  + 切片时**不考虑数据集整体**，而是逐个**针对每个文件单独切片**

+ **源码解析（input.getSplits(job))**

  + 程序找到数据存储的目录

  + 遍历处理目录下的每一个文件（规划切片）

  + 处理第一个文件xx.txt

    + 获取文件大小 fs.sizeOf (xx.txt)

    + 计算切片大小 computeSplitSize (Math.max **(**minSize, Math.min (maxSize,blockSize)**)**) = blockSize

    + 默认情况下，切片大小即为数据块大小

      > 可通过修改默认为Long.MAXValue 的maxSize调小切片，修改默认为1的minSize调大切片

    + 开始切片，形成第1个切片：0-128M，第2个切片：128-200M

      > 每次切片时，都要判断切完剩下部分是否大于块的1.1倍，不大于1.1倍就划分一块切片

    + 将切片信息写到一个切片规划文件Job.split中

    + 整个切片的核心过程在getSplits()方法中完成

    + InputSplit只记录了切片的元数据信息（起始位置、长度、所在节点列表）

  + 提交切片规划文件到YARN上，MrAppMaster根据切片规划文件计算开启MapTask数量

+ **参数配置**

  + 源码中计算切片大小的公式

    + Math.max ( **minSize** , Math.min ( **maxSize** , **blockSize** ))
    + mapreduce.input.fileinputformat.split.minsize = 1
    + mapreduce.input.fileinputformat.split.maxsize = Long.MAXValue

  + 切片大小设置

    + maxsize：如果比blocksize小，则让切片变小，值为maxsize
    + minsize：如果比blocksize大，则让切片变大，值为minsize

  + 获取切片信息API

    ```java
    //获取切片的文件名称（便于处理不同类型文件）
    String name = inputSplit.getPath().getName();
    
    //根据文件类型获取切片信息
    FileSplit inputSplit = (FileSplit) context.getInputSplit();
    ```

#### 3.1.4 TextInputFormat

+ **FileInputFormat实现类**

  常见接口实现类：TextInputFormat、KeyValueTextInputFormat、NLineInputFormat、CombineTextInputFormat 和 自定义InputFormat等
  
+ **TextInputFormat**

  FileInputFormat默认实现类。按行读取每条记录，**key**（LongWritable类型）是该行在整个文件中的起始字节**偏移量**，**value**（Text类型）是这行的**内容**（不包括换行符）

#### 3.1.5 CombineTextInputFormat切片机制

> 默认切片机制：**按文件规划切片**。不管文件多小，都会是一个单独的切片，产生一个MapTask
>
> 缺点：**如果有大量小文件，就会产生大量MapTask，处理效率低下**

+ **CombineTextInputFormat**

  + 应用场景：大量小文件

  + 作用：将多个小文件从逻辑上规划到一个切片中。使多个小文件交给一个MapTask处理

  + 方式：设置虚拟存储切片最大值

    ```java
    CombineTextInputFormat.setMaxInputSplitSize(job, 4194304);// 4m
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

      > 比较 输入目录下所有文件大小 和 设置的 setMaxInputSplitSize值
      >
      > 文件<=设置最大值，逻辑上划分一个块；
      >
      > 设置最大值<文件<=最大值*2，将文件均分成2个虚拟存储块（防止出现太小切片）
      >
      > 文件>设置最大值*2，以最大值切割一块

    + 切片过程

      > 虚拟存储文件大小>=设置的 setMaxInputSplitSize值，单独形成一个切片
      >
      > 否则跟下一个虚拟存储文件进行合并，共同形成一个切片

#### 3.1.6 CombineTextInputFormat案例实操

+ 不做任何处理，执行WordCount案例程序，切片数量为4

  ```java
  number of splits:4
  ```

+ 在**WordCountDriver**中增加如下代码，执行程序，切片数量为3

  ```java
  // 如果不设置InputFormat，它默认用的是TextInputFormat.class
  job.setInputFormatClass(CombineTextInputFormat.class);
  
  //虚拟存储切片最大值设置4m
  CombineTextInputFormat.setMaxInputSplitSize(job, 4194304);
  ```

  ```java
  number of splits:3
  ```

+ 在**WordCountDriver**中增加如下代码，执行程序，切片数量为3

  ```java
  // 如果不设置InputFormat，它默认用的是TextInputFormat.class
  job.setInputFormatClass(CombineTextInputFormat.class);
  
  //虚拟存储切片最大值设置20m
  CombineTextInputFormat.setMaxInputSplitSize(job, 20971520);
  ```

  ```java
  number of splits:1
  ```

### 3.2 MapReduce工作流程

+ **MapReduce**

1. 待处理文件

2. 客户端submit()前，获取待处理数据的信息，配置参数，形成任务分配规划

3. 提交信息：Job.split（切片信息）、wc.jar（程序jar包）、Job.xml（配置信息）

   > 提交完后判断是提交到YARN还是本地运行

4. 计算出MapTask数量

   > Mrappmaster与NodeManager协作

5. MapTask通过InputFormat实现类，RecorderReader，以键值对形式读取待处理文件

6. Mapper执行逻辑运算

   > map（k，v）
   >
   > Context.write（k，v）

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

    > Reduce（k，v）
    >
    > Context.write（k，v）

15. GroupingComparaton（k，knext）分组

16. OutputFormat，RecordWriter，Write（k，v）

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
> 2. 缓冲区大小设置：mapreduce.task.io.sort.mb默认100M

### 3.3 Shuffle

### 3.3.1 Shuffle机制

Shuffle：Map方法之后，Reduce方法之前的数据处理过程

### 3.3.2 Partition分区

+ **需求**：将统计结果按照条件**输出到不同文件**中（分区）

+ **默认**：默认分区根据key的hashcode对ReduceTasks个数取模得到，用户没法控制key存储到哪个分区

+ **解决**

  + **自定义Partitioner**，重写getPartitioner（）
  + 在**Driver中设置**自定义Partitioner，并设置相应数量的ReduceTask

+ **编写MapReduce程序**

  + 在2.3序列化案例实操的基础上，增加一个**Partitioner类**

  ```java
  package com.flash7k.mapreduce.partitioner;
  import org.apache.hadoop.io.Text;
  import org.apache.hadoop.mapreduce.Partitioner;
  
  public class ProvincePartitioner extends Partitioner<Text, FlowBean> {
  
      @Override
      public int getPartition(Text text, FlowBean flowBean, int numPartitions) {
          //获取手机号前三位prePhone
          String phone = text.toString();
          String prePhone = phone.substring(0, 3);
  
          //定义一个分区号变量partition,根据prePhone设置分区号
          int partition;
  
          if("136".equals(prePhone)){
              partition = 0;
          }else if("137".equals(prePhone)){
              partition = 1;
          }else if("138".equals(prePhone)){
              partition = 2;
          }else if("139".equals(prePhone)){
              partition = 3;
          }else {
              partition = 4;
          }
  
          //最后返回分区号partition
          return partition;
      }
  }
  ```

  + 在Driver中增加**自定义数据分区设置和ReduceTask设置**

  ```java
  //8 指定自定义分区器
  job.setPartitionerClass(ProvincePartitioner.class);
  
  //9 同时指定相应数量的ReduceTask
  job.setNumReduceTasks(5);
  ```

+ **总结**
  + 如果 **ReduceTask数量 > getPartitioner结果数**，则会多产生空的输出文件part-r-000xx
  + 如果 **1 < ReduceTask数量 < getPartitioner结果数**，则有一部分分区数据无处安放，会Exception
  + 如果 **ReduceTask数量 = 1**，则不管MapTask输出多少个分区文件，最终结果都交给这一个RuduceTask，产生一个输出文件part-r-00000
  + 分区号从0开始，逐一累加

### 3.3.3 WritableComparable排序

+ **排序**

  MapTask和RuduceTask均会对数据按照key进行排序。Hadoop默认行为，不管逻辑上是否需要。默认排序按照字典顺序排序，排序方式是快速排序

  + **MapTask**，<u>将处理的结果暂时放到环形缓冲区</u>
    + 当<u>环形缓冲区使用率达到一定阈值</u>，**对缓冲区中数据进行快速排序**，并将这些有序数据**溢写到磁盘**
    + 当<u>数据处理完毕后</u>，**对磁盘上所有文件进行归并排序**
  + **ReduceTask**，<u>它从每个MapTask远程拷贝相应的数据文件</u>
    + 如果<u>文件大小超过一定阈值</u>，则**溢写到磁盘上**；否则存储在内存中
    + 如果<u>磁盘上文件数目达到一定阈值</u>，则进行**归并排序**，以生成一个更大文件
    + 如果<u>内存中文件大小或数目超过一定阈值</u>，则进行**合并后将数据溢写到磁盘上**
    + 当<u>所有数据拷贝完毕后</u>，统一**对内存和磁盘上的所有数据进行归并排序**

+ **排序分类**

  + 部分排序

    根据输入记录的kv数据集排序，保证输出的每个文件内部有序

  + 全排序

    最终输出结果只有一个文件，文件内部有序。即只设置一个ReduceTask

  + 辅助排序

    在Reduce端对key进行分组。应用：在接收的key为bean对象时，想让一个或几个字段相同的key进入到同一个reduce方法中

  + 二次排序

    自定义排序过程中，compareTo中的判断条件为两个

+ 编写MapReduce程序

  + **自定义Bean实现WritableComparable，重写compareTo()**

  ```java
  @Override
  public int compareTo(FlowBean bean) {
  
  	int result;
  		
  	// 按照总流量大小，倒序排列
  	if (this.sumFlow > bean.getSumFlow()) {
  		result = -1;
  	}else if (this.sumFlow < bean.getSumFlow()) {
  		result = 1;
  	}else {
  		result = 0;
  	}
  
  	return result;
  }
  ```

  + 编写Mapper类：输出类型<Bean，Text>

  ```java
  package com.flash7k.mapreduce.writablecompable;
  
  import org.apache.hadoop.io.LongWritable;
  import org.apache.hadoop.io.Text;
  import org.apache.hadoop.mapreduce.Mapper;
  import java.io.IOException;
  
  public class FlowMapper extends Mapper<LongWritable, Text, FlowBean, Text> {
      private FlowBean outK = new FlowBean();
      private Text outV = new Text();
  
      @Override
      protected void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException {
  
          //1 获取一行数据
          String line = value.toString();
  
          //2 按照"\t",切割数据
          String[] split = line.split("\t");
  
          //3 封装outK outV
          outK.setUpFlow(Long.parseLong(split[1]));
          outK.setDownFlow(Long.parseLong(split[2]));
          outK.setSumFlow();
          outV.set(split[0]);
  
          //4 写出outK outV
          context.write(outK,outV);
      }
  }
  ```

  + 编写Reducer类

  ```java
  package com.flash7k.mapreduce.writablecompable;
  
  import org.apache.hadoop.io.Text;
  import org.apache.hadoop.mapreduce.Reducer;
  import java.io.IOException;
  
  public class FlowReducer extends Reducer<FlowBean, Text, Text, FlowBean> {
      @Override
      protected void reduce(FlowBean key, Iterable<Text> values, Context context) throws IOException, InterruptedException {
  
          //遍历values集合,循环写出,避免总流量相同的情况
          for (Text value : values) {
              //调换KV位置,反向写出
              context.write(value,key);
          }
      }
  }
  ```

### 3.3.4 Combiner合并

+ **Combiner**

  + Combiner是MR程序中Mapper和Reducer之外的一种组件
  + Combiner继承Reducer
  + Combiner和Reducer区别在于运行的位置
    + Combiner运行在每一个MapTask所在节点
    + Reducer接收所有Mapper的输出结果
  + Combiner的意义：对每个MapTask的输出进行局部汇总，以减小网络传输量
  + Combiner应用前提是不能影响最终的业务逻辑，且输出kv要和Reducer输入kv对应

+ 编写MapReduce程序

  + **自定义Combiner，继承Reducer**，重写reduce方法
  + 在Driver类中设置

  ```java
  job.setCombinerClass(WordCountCombiner.class);
  ```

  > 方案一：创建一个新的Combiner类
  >
  > 方案二：在Driver中设置Combiner为原来的Reducer

### 3.4 OutputFormat数据输出

+ **OutputFormat接口实现类**

  + NullOutputFormat
  + FileOutputFormat
    + MapFileOutputFormat
    + SequenceFileOutputFormat
    + TextOutputFormat
  + FilterOutputFormat
  + DBOutputFormat
  + 自定义OutputFormat

+ **自定义OutputFormat案例实操**

  + **需求**

    过滤输入的log日志，包含flash7k的网站输出到 D:/flash7k.log，不包含的网站输出到D:/other.log

  + **需求分析**

    + 自定义OutputFormat类

      + 创建LogRecorderWriter类，继承RecorderWriter

        > 创建两个文件的输出流：flash7kOut、otherOut
        >
        > 如果输入数据包含flash7k，输出到flash7kOut；否则输出到otherOut

    + 驱动类Driver

      + 设置自定义的OutputFormat

  + 编写MapReduce程序

    + 编写LogMapper

    ```java
    package com.flash7k.mapreduce.outputformat;
    
    import org.apache.hadoop.io.LongWritable;
    import org.apache.hadoop.io.NullWritable;
    import org.apache.hadoop.io.Text;
    import org.apache.hadoop.mapreduce.Mapper;
    
    import java.io.IOException;
    
    public class LogMapper extends Mapper<LongWritable, Text,Text, NullWritable> {
        @Override
        protected void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException {
            //不做任何处理,直接写出一行log数据
            context.write(value,NullWritable.get());
        }
    }
    ```

    + 编写LogReducer

    ```java
    package com.flash7k.mapreduce.outputformat;
    
    import org.apache.hadoop.io.NullWritable;
    import org.apache.hadoop.io.Text;
    import org.apache.hadoop.mapreduce.Reducer;
    
    import java.io.IOException;
    
    public class LogReducer extends Reducer<Text, NullWritable,Text, NullWritable> {
        @Override
        protected void reduce(Text key, Iterable<NullWritable> values, Context context) throws IOException, InterruptedException {
            // 防止有相同的数据,对values迭代写出
            for (NullWritable value : values) {
                context.write(key,NullWritable.get());
            }
        }
    }
    ```

    + 自定义LogOutputFormat

    ```java
    package com.flash7k.mapreduce.outputformat;
    
    import org.apache.hadoop.io.NullWritable;
    import org.apache.hadoop.io.Text;
    import org.apache.hadoop.mapreduce.RecordWriter;
    import org.apache.hadoop.mapreduce.TaskAttemptContext;
    import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;
    
    import java.io.IOException;
    
    public class LogOutputFormat extends FileOutputFormat<Text, NullWritable> {
        @Override
        public RecordWriter<Text, NullWritable> getRecordWriter(TaskAttemptContext job) throws IOException, InterruptedException {
            //创建一个自定义的RecordWriter返回
            LogRecordWriter logRecordWriter = new LogRecordWriter(job);
            return logRecordWriter;
        }
    }
    ```

    + 编写LogRecordWriter

    ```java
    package com.flash7k.mapreduce.outputformat;
    
    import org.apache.hadoop.fs.FSDataOutputStream;
    import org.apache.hadoop.fs.FileSystem;
    import org.apache.hadoop.fs.Path;
    import org.apache.hadoop.io.IOUtils;
    import org.apache.hadoop.io.NullWritable;
    import org.apache.hadoop.io.Text;
    import org.apache.hadoop.mapreduce.RecordWriter;
    import org.apache.hadoop.mapreduce.TaskAttemptContext;
    
    import java.io.IOException;
    
    public class LogRecordWriter extends RecordWriter<Text, NullWritable> {
    
        private FSDataOutputStream flash7kOut;
        private FSDataOutputStream otherOut;
    
        public LogRecordWriter(TaskAttemptContext job) {
            try {
                //获取文件系统对象
                FileSystem fs = FileSystem.get(job.getConfiguration());
                //用文件系统对象创建两个输出流对应不同的目录
                flash7kOut = fs.create(new Path("D:/hadoop/flash7k.log"));
                otherOut = fs.create(new Path("D:/hadoop/other.log"));
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    
        @Override
        public void write(Text key, NullWritable value) throws IOException, InterruptedException {
            String log = key.toString();
            //根据一行的log数据是否包含flash7k,判断两条输出流输出的内容
            if (log.contains("flash7k")) {
                flash7kOut.writeBytes(log + "\n");
            } else {
                otherOut.writeBytes(log + "\n");
            }
        }
    
        @Override
        public void close(TaskAttemptContext context) throws IOException, InterruptedException {
            //关流
            IOUtils.closeStream(flash7kOut);
            IOUtils.closeStream(otherOut);
        }
    }
    ```

    + 编写LogDriver

    ```java
    package com.flash7k.mapreduce.outputformat;
    
    import org.apache.hadoop.conf.Configuration;
    import org.apache.hadoop.fs.Path;
    import org.apache.hadoop.io.NullWritable;
    import org.apache.hadoop.io.Text;
    import org.apache.hadoop.mapreduce.Job;
    import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
    import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;
    
    import java.io.IOException;
    
    public class LogDriver {
        public static void main(String[] args) throws IOException, ClassNotFoundException, InterruptedException {
    
            Configuration conf = new Configuration();
            Job job = Job.getInstance(conf);
    
            job.setJarByClass(LogDriver.class);
            job.setMapperClass(LogMapper.class);
            job.setReducerClass(LogReducer.class);
    
            job.setMapOutputKeyClass(Text.class);
            job.setMapOutputValueClass(NullWritable.class);
    
            job.setOutputKeyClass(Text.class);
            job.setOutputValueClass(NullWritable.class);
    
            //设置自定义的outputformat
            job.setOutputFormatClass(LogOutputFormat.class);
    
            FileInputFormat.setInputPaths(job, new Path("D:\\input"));
            //虽然我们自定义了outputformat，但是因为我们的outputformat继承自fileoutputformat
            //而fileoutputformat要输出一个_SUCCESS文件，所以在这还得指定一个输出目录
            FileOutputFormat.setOutputPath(job, new Path("D:\\logoutput"));
    
            boolean b = job.waitForCompletion(true);
            System.exit(b ? 0 : 1);
        }
    }
    ```


### 3.5 MapReduce内核源码解析

### 3.5.1 MapTask工作机制

1. **Read阶段**

   MapTask 通过 **InputFormat** 获得的 **RecordReader**，从输入 InputSplit 中解析出一个个key / value

2. **Map阶段**

   将解析出的 key / value 交给用户编写的 **map()** 函数处理，并产生一系列新的 key / value

3. **Collect阶段**

   用户编写的 map() 函数数据处理完成后，一般会调用 **OutputCollector.collect()** 输出结果，将产生新的 key / value，**分区（调用Partitioner）**，并**写入一个环形内存缓冲区**中

4. **Spill阶段（溢写）**

   环形缓冲区达到80%后，MapReduce 会将数据写到本地磁盘上，生成一个临时文件。注意：数据写入本地磁盘前，先对数据进行一次本地排序，并在必要时进行合并、压缩

   + 使用快速排序算法**对缓冲区内的数据进行排序**。先按照分区编号，再按照 key。排序后，数据以分区为单位聚集在一起，**同一分区内所有数据按照 key 有序**
   + 按照分区编号由小到大，依次将每个分区中的**数据写入任务工作目录下的临时文件output / spillN.out** （N表示当前溢写次数）。<u>如果用户设置了 Combiner，则写入文件之前，对每个分区中的数据进行一次聚集操作</u>
   + 将分区数据的元**信息写到内存索引数据结构 SpillRecord** 中，元信息包括在临时文件中的偏移量、压缩前数据大小、压缩后数据大小。<u>如果当前内存索引大小超过1MB，则将内存索引写到文件 output / spillN.out.index中</u>

5. **Merge阶段**

   当所有数据处理完成后，MapTask **对所有临时文件进行一次合并**，以确保最终只会生产一个数据文件，并将其保存到文件 output / file.out 中，同时生成相应的索引文件 output / file.out.index

   > **合并方式**：MapTask 以分区为单位进行合并。对于分区采用**多轮递归合并**，每轮合并 mapreduce.task.io.sort.factor（默认10）个文件，并将产生的文件重新加入待合并列表中。排序后，重复以上过程，直到最终获得一个大文件
   >
   > 
   >
   > **好处**：避免同时打开大量文件和同时随机读取大量小文件带来的开销

### 3.5.2 ReduceTask工作机制

1. **Copy阶段**

   ReduceTask 从各个 MapTask 上远程拷贝一片数据，如果数据大小不超过一定阈值，则直接放到内存中，否则写到磁盘上

2. **Sort阶段**

   在远程拷贝数据过程中，ReduceTask 启动两个后台线程对内存和磁盘上的文件进行合并，以防止内存使用过多或磁盘上文件过多

   > 用户编写 reduce()，输入数据根据key进行聚集，为了将 key 相同的数据聚在一起，Hadoop 采用基于排序的策略。由于各个 MapTask 已经对处理结果进行了局部排序，ReduceTask 只需对所有数据进行一次归并排序即可

3. **Reduce阶段**

   reduce() 将计算结果写到 HDFS 上

### 3.5.3 ReduceTask并行度决定机制

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

### 3.6 Join应用

### 3.6.1 Reduce Join

+ Map端主要工作：对来自不同表或文件的 key / value，打标签以区别不同来源的记录，然后连接字段作为key，其余部分和新加的标志作为value，输出
+ Reduce端主要工作：在每一个分组当中将不同来源文件的记录（在Map阶段已标记）分开，最后进行合并

### 3.6.2 Reduce Join 案例实操

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

### 3.6.3 Map Join

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

### 3.6.4 Map Join 案例实操

+ 需求分析

  + DistributedCacheDriver缓存文件
    1. 加载缓存数据 job.addCacheDriver(new URI(文件路径))；
    2. Map Join不需要Reduce阶段，设置 job.setNumReduceTask(0);
  + 读取缓存的文件数据
    + setup()
      1. 获取缓存的文件
      2. 循环读取缓存文件中的一行
      3. 切割
      4. 缓存数据到集合
      5. 关流
    + map()
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
    			//01	小米
                String[] split = line.split("\t");
                pdMap.put(split[0], split[1]);
            }
    
            //关流
            IOUtils.closeStream(reader);
        }
    
        @Override
        protected void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException {
    
            //读取大表数据    
    		//1001	01	1
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

### 3.7 数据清洗ELT

ELT，Extract-Transform-Load，描述将数据从来源端经过抽取（Extract）、转换（Transform）、加载（Load）至目的端的过程。在运行核心业务MapReduce之前，往往要先对数据进行清洗，去除不符合要求的数据。**清理的过程通常只需要Mapper，不需要Reducer**

+ 需求：去除日志中字段个数小于等于11的日志

+ 代码

  + WebLogMapper

    ```java
    package com.atguigu.mapreduce.weblog;
    import java.io.IOException;
    import org.apache.hadoop.io.LongWritable;
    import org.apache.hadoop.io.NullWritable;
    import org.apache.hadoop.io.Text;
    import org.apache.hadoop.mapreduce.Mapper;
    
    public class WebLogMapper extends Mapper<LongWritable, Text, Text, NullWritable>{
    	
    	@Override
    	protected void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException {
    		
    		// 1 获取1行数据
    		String line = value.toString();
    		
    		// 2 解析日志
    		boolean result = parseLog(line,context);
    		
    		// 3 日志不合法退出
    		if (!result) {
    			return;
    		}
    		
    		// 4 日志合法就直接写出
    		context.write(value, NullWritable.get());
    	}
    
    	// 2 封装解析日志的方法
    	private boolean parseLog(String line, Context context) {
    
    		// 1 截取
    		String[] fields = line.split(" ");
    		
    		// 2 日志长度大于11的为合法
    		if (fields.length > 11) {
    			return true;
    		}else {
    			return false;
    		}
    	}
    }
    ```

  + WebLogDriver

    设置reducetask个数为0：job.setNumReduceTasks(0);

### 3.8 MapReduce开发总结

+ 输入数据接口：InputFormat

  + 默认实现类：TextInputFormat
  + TextInputFormat：一次读取一行，起始偏移量作为key，行内容作为value
  + CombineTextInputFormat：可以把多个小文件合并成一个数据切片，减少MapTask数量，提高处理效率

+ 逻辑处理接口：Mapper

  + 用户根据业务需求实现三个方法：map、setup、cleanup

+ Partitioner分区

  + 默认实现HashPartitioner，根据key的哈希值和numReduces来返回分区号：key.hashCode() & Integer.MAXVALUE % numReduces
  + 如果业务上有特殊需求，可自定义分区（如输出到不同的文件中）

+ Comparable排序接口

  + 自定义对象作为Mapper输出的key，必须实现WritableComparable接口，重写compareTo方法

    > 部分排序：对最终输出的每个文件进行内部排序
    >
    > 全排序：对所有数据进行排序，通常只有一个Reduce
    >
    > 二次排序：排序的条件有两个

+ Combiner合并

  + 运行在MapTask节点
  + 对每个MapTask的输出进行局部汇总，以减小网络传输量
  + 使用时不能影响原有的业务处理结果

+ 逻辑处理接口：Reducer

  + 用户根据业务需求实现三个方法：reduce、setup、cleanup

+ 输出数据接口：OutputFormat

  + 默认实现类：TextOutputFormat
  + TextOutputFormat：将每一个KV对，向目标文本文件输出一行
  + 用户可自定义OutputFormat

## 第4章 Hadoop数据压缩

## 4.1 概述

+ 优缺点
  + 优点：减少磁盘IO、减少磁盘存储空间
  + 缺点：增加CPU开销
+ 压缩原则
  + 运算密集型的Job，少用压缩
  + IO密集型的Job，多用压缩

## 4.2 MR支持的压缩编码

+ 压缩算法对比介绍
  + DEFLATE
    + Hadoop自带
    + 算法：DEFLATE
    + 文件扩展名：.deflate
    + 不可切片
    + 压缩后原程序不需修改
  + Gzip
    + Hadoop自带
    + 算法：DEFLATE
    + 文件扩展名：.gz
    + 不可切片
    + 压缩后原程序不需修改
  + bzip2
    + Hadoop自带
    + 算法：bzip2
    + 文件扩展名：.bz2
    + 可切片
    + 压缩后原程序不需修改
  + LZ0
    + Hadoop不自带，需安装
    + 算法：LZ0
    + 文件扩展名：.lzo
    + 可切片
    + 压缩后原程序需要建索引，还需指定输入格式
  + Snappy
    + Hadoop自带
    + 算法：Snappy
    + 文件扩展名：.snappy
    + 不可切片
    + 压缩后原程序不需修改
+ 压缩性能比较
  + gzip：压缩后文件较小，压缩速度较快，解压速度较快
  + bzip2：压缩后文件最小，压缩速度极慢，解压速度极慢
  + LZO：压缩后文件较大，压缩速度极快，解压速度极快

## 4.3 压缩方式选择

考虑因素：**压缩/解压缩速度**、**压缩率（压缩后存储大小）**、**压缩后是否可以切片**

+ 压缩位置选择

  压缩可以在MapReduce的任意阶段启用

  + Map输入端
    + 数据量小于块大小，重点考虑压缩/解压缩速度较快的LZO/Snappy
    + 数据量非常大，重点考虑支持切片的Bzip2和LZO
  + Map输出端
    + 为减少MapTask和ReduceTask之间的网络IO，重点考虑压缩/解压缩速度较快的LZO/Snappy
  + Reduce输出端
    + 若数据永久保存，重点考虑压缩率高的Bzip2/Gzip
    + 若作为下一个MapReduce输入，重点考虑数据量和是否支持切片

## 4.4 压缩参数配置

+ 为支持多种压缩/解压缩算法，Hadoop引入了编码/解码器
  + org.apache.hadoop.io.compress.DefaultCodec
  + org.apache.hadoop.io.compress.GzipCodec
  + org.apache.hadoop.io.compress.BzipCodec
  + com.hadoop.compression.lzo.LzopCodec
  + org.apache.hadoop.io.compress.SnappyCodec
+ 配置文件
  + core-site.xml
    + io.compression.codec：默认值无，mapper输入端压缩，Hadoop使用文件扩展名判断是否支持解编码
  + mapred-site.xml
    + mapreduce.map.output.compress：默认值false，mapper输出端压缩，设为true则启用压缩
    + mapreduce.map.output.compress.codec：默认值org.apache.hadoop.io.compress.DefaultCodec，mapper输出端压缩，更改解编码器（多用LZO\Snappy）
    + mapreduce.output.fileoutputformat.compress：默认值false，reducer输出端压缩，设为true则启用压缩
    + mapreduce.output.fileoutputformat.compress.codec：默认值org.apache.hadoop.io.compress.DefaultCodec，更改解编码器（多用gzip/bzip2）

## 4.5 压缩案例实操

### 4.5.1 Mapper输出端压缩

+ Driver

```java
package com.flash7k.mapreduce.compress;
import java.io.IOException;
import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.IntWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.io.compress.BZip2Codec;	
import org.apache.hadoop.io.compress.CompressionCodec;
import org.apache.hadoop.io.compress.GzipCodec;
import org.apache.hadoop.mapreduce.Job;
import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;

public class WordCountDriver {

	public static void main(String[] args) throws IOException, ClassNotFoundException, InterruptedException {

		Configuration conf = new Configuration();

		// 开启map端输出压缩
		conf.setBoolean("mapreduce.map.output.compress", true);

		// 设置map端输出压缩方式
		conf.setClass("mapreduce.map.output.compress.codec", BZip2Codec.class,CompressionCodec.class);

		Job job = Job.getInstance(conf);

		job.setJarByClass(WordCountDriver.class);

		job.setMapperClass(WordCountMapper.class);
		job.setReducerClass(WordCountReducer.class);

		job.setMapOutputKeyClass(Text.class);
		job.setMapOutputValueClass(IntWritable.class);

		job.setOutputKeyClass(Text.class);
		job.setOutputValueClass(IntWritable.class);

		FileInputFormat.setInputPaths(job, new Path(args[0]));
		FileOutputFormat.setOutputPath(job, new Path(args[1]));

		boolean result = job.waitForCompletion(true);

		System.exit(result ? 0 : 1);
	}
}
```

+ Mapper和Reducer保持不变

### 4.5.2 Reducer输出端压缩

+ Driver

```java
package com.flash7k.mapreduce.compress;
import java.io.IOException;
import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.IntWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.io.compress.BZip2Codec;
import org.apache.hadoop.io.compress.DefaultCodec;
import org.apache.hadoop.io.compress.GzipCodec;
import org.apache.hadoop.io.compress.Lz4Codec;
import org.apache.hadoop.io.compress.SnappyCodec;
import org.apache.hadoop.mapreduce.Job;
import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;

public class WordCountDriver {

	public static void main(String[] args) throws IOException, ClassNotFoundException, InterruptedException {
		
		Configuration conf = new Configuration();
		
		Job job = Job.getInstance(conf);
		
		job.setJarByClass(WordCountDriver.class);
		
		job.setMapperClass(WordCountMapper.class);
		job.setReducerClass(WordCountReducer.class);
		
		job.setMapOutputKeyClass(Text.class);
		job.setMapOutputValueClass(IntWritable.class);
		
		job.setOutputKeyClass(Text.class);
		job.setOutputValueClass(IntWritable.class);
		
		FileInputFormat.setInputPaths(job, new Path(args[0]));
		FileOutputFormat.setOutputPath(job, new Path(args[1]));
		
		// 设置reduce端输出压缩开启
		FileOutputFormat.setCompressOutput(job, true);

		// 设置压缩的方式
	    FileOutputFormat.setOutputCompressorClass(job, BZip2Codec.class); 
//	    FileOutputFormat.setOutputCompressorClass(job, GzipCodec.class); 
//	    FileOutputFormat.setOutputCompressorClass(job, DefaultCodec.class); 
	    
		boolean result = job.waitForCompletion(true);
		
		System.exit(result?0:1);
	}
}
```
