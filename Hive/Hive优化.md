# 1. Explain查看执行计划

由于执行一段语句需要很长时间，因此只查看执行计划

+ 查看执行计划

```sql
EXPLAIN [EXTENDED | DEPENDENCY | AUTHORIZATION] query-sql;
```

+ 查看详细执行计划

```sql
EXPLAIN EXTENDED query-sql;
```

# 2. Hive建表优化

## 1）分区表：partitioned by

分区表实际上就是对应一个 HDFS 上独立的文件夹，该文件夹下是该分区所有的数据文件。Hive 中的分区就是分目录，把一个大的数据集根据业务需要分割成小的数据集。在查询时通过 WHERE 子句中的表达式选择查询所需要的指定的分区，避免全表扫描，提高查询效率。

**基本操作**

+ 创建分区表：`partitioned by`

```sql
create table dept_partition( deptno int, dname string )
partitioned by (day string)
row format delimited fields terminated by '\t';
```

> 注意：分区字段不能是表中已经存在的数据，可以将分区字段看作表的伪列

+ 加载数据

```sql
load data local inpath '/opt/module/hive/datas/dept_20200401.log'
into table dept_partition
partition(day='20200401');
```

> 注意：分区表加载数据时，必须指定分区

+ 增加分区

```sql
alter table dept_partition add partition(day='20200404');
```

+ 增加多个分区**（分区间空格隔开）**

```sql
alter table dept_partition add partition(day='20200405') partition(day='20200406');
```

+ 删除分区

```sql
alter table dept_partition drop partition (day='20200406');
```

+ 删除多个分区**（分区间`,`隔开）**

```sql
alter table dept_partition drop partition (day='20200404'), partition(day='20200405');
```

+ 查看分区数量

```sql
show partitions dept_partition;
```

+ 查看分区表结构

```sql
desc formatted dept_partition;
```

**二级分区**

+ 创建二级分区表

```sql
create table dept_partition2( deptno int, dname string, loc string
)
partitioned by (day string, hour string);
```

+ 直接加载数据：`load`

```sql
load data local inpath '/opt/module/hive/datas/dept_20200401.log'
into table dept_partition2
partition(day='20200401', hour='12');
```

+ Hadoop命令把数据直接上传到分区目录上后，让分区表与数据产生关联
  
  + 修复
  
  ```sql
  msck repair table dept_partition2;
  ```
  
  + 添加分区
  
  ```sql
  alter table dept_partition2 add partition(day='201709',hour='14');
  ```
  
  + `load`

**动态分区调整**

+ 开启动态分区功能（默认 true，开启）

```sql
hive.exec.dynamic.partition=true 
```

+ 设置为非严格模式（动态分区的模式，默认 strict，表示必须指定至少一个分区为静态分区；nonstrict 模式表示允许所有的分区字段都可以使用动态分区。）

```sql
hive.exec.dynamic.partition.mode=nonstrict 
```

+ 设置在<u>所有</u>执行 MR 的节点上，最大一共可以创建多少个动态分区。默认 1000

```sql
hive.exec.max.dynamic.partitions=1000 
```

+ 设置在<u>每个</u>执行 MR 的节点上，最大可以创建多少个动态分区。（该参数需要根据实际的数据来设定。比如：源数据中包含了一年的数据，即 day 字段有 365 个值，那么该参数就
  
  需要设置成大于 365，如果使用默认值 100，则会报错。）

```sql
hive.exec.max.dynamic.partitions.pernode=100 
```

+ 设置整个 MR Job 中，最大可以创建多少个 HDFS 文件。默认 100000

```sql
hive.exec.max.created.files=100000
```

+ 设置当有空分区生成时，是否抛出异常。一般不需要设置。默认 false

```sql
hive.error.on.empty.partition=false 
```

> 基本为默认值，设置动态分区时，只需`set hive.exec.dynamic.partition.mode = nonstrict;`

## 2）分桶表：clustered by

分区提供一个隔离数据和优化查询的便利方式。不过，并非所有的数据集都可形成合理的分区。对于一张表或者分区，Hive 可以进一步组织成桶，也就是更为细粒度的数据范围划分。

分桶是将数据集分解成更容易管理的若干部分的另一个技术。 **分区针对的是数据的存储路径；分桶针对的是数据文件。**

**基本操作**

+ 创建分桶表：`clustered by`

```sql
create table stu_buck(id int, name string)
clustered by(id)
into 4 buckets
row format delimited fields terminated by '\t';
```

> **分桶规则：Hive 的分桶采用对分桶字段的值进行哈希，然后除以桶的个数求余的方式决定该条记录存放在哪个桶当中**

**注意事项**

+ 设置reduce 的个数为-1,让 Job 自行决定需要用多少个 reduce 或者将 reduce 的个数设置为大于等于分桶表的桶数
+ 从HDFS 中 load 数据到分桶表中，避免本地文件找不到问题
+ 不要使用本地模式

**抽样查询**

对于非常大的数据集，有时用户需要使用的是一个具有代表性的查询结果而不是全部结 果。Hive 可以通过对表进行抽样来满足这个需求。

```sql
select * from stu_buck tablesample(bucket 1 out of 4 on id);
```

> 语法：`TABLESAMPLE(BUCKET x OUT OF y)`
> 
> 注意：x的值必须小于等于y
> 
> 解释：查询样本大小约为1/y，y需要是创建表时指定的桶数的倍数或因子。Hive将桶分为y个桶组，选择每个组的第x个桶

## 3）合适的文件格式

Hive 支持的存储数据的格式主要有：TEXTFILE 、SEQUENCEFILE、ORC、PARQUET。

+ 行存储
  + 查询满足条件的一整行数据的时候，列存储则需要去每个聚集的字段找到对应的每个列的值，行存储只需要找到其中一个值，其余的值都在相邻地方，所以此时行存储查询的速度更快
  + TEXTFILE 、 SEQUENCEFILE 
+ 列存储
  + 因为每个字段的数据聚集存储，在查询只需要少数几个字段的时候，能大大减少读取的数据量；每个字段的数据类型一定是相同的，列式存储可以针对性的设计更好的设计压缩算法。
  + ORC 、 PARQUET

**TextFile**

默认格式，数据不做压缩，磁盘开销大，数据解析开销大。

可结合 Gzip、Bzip2 使用。 但使用 Gzip ，Hive 不会对数据进行切分，从而无法对数据进行并行操作。

**ORC**

每个 Orc 文件由 1 个或多个 stripe 组成，每个 stripe 一般为 HDFS的块大小，每一个 stripe 包含多条记录，这些记录按照列进行独立存储，对应到 Parquet 中的 row group 的概念。每个Stripe 里有三部分组成，分别是 `Index Data`，`Row Data`，`Stripe Footer`

+ `Index Data`：一个轻量级的 index，**默认是每隔 1W 行做一个索引**。这里做的索引应该只是记录某行的各字段在 `Row Data `中的 `offset`。（防止单个字段行数过多，数据倾斜）
+ `Row Data`：存储具体的数据。先取部分行，然后对这些行按列进行存储。对每个列进行编码，分成多个 `Stream `来存储。
+ `Stripe Footer`：存的是各个 `Stream` 的类型，长度等信息。每个文件有一个 `File Footer`，存的是每个` Stripe `的行数，每个` Column` 的数据类型信息等。每个文件的尾部是一个 `PostScript`，记录了整个文件的压缩类型以及 `File Footer` 的长度信息等。在读取文件时，会 seek 到文件尾部读 `PostScript`，从里面解析到 `File Footer `长度，再读` File Footer`，从里面解析到各个 `Stripe `信息，再读各个 `Stripe`，即从后往前读。

**Parquet**

文件以二进制方式存储，所以**不可以直接读取**。

文件中包括该文件的数据和元数据，因此Parquet格式文件是**自解析的**。

## 4）合适的压缩格式

# 3. HQL语法优化

## 1）列裁剪与分区裁剪

+ 列裁剪：在查询时只读取需要的列
+ 分区裁剪：只读取需要的分区

## 2）Group By

默认情况下，Map阶段同一Key的数据分发给同一个Reduce，而当一个Key数据量过大时就发生数据倾斜。因此，并不是所有聚合操作都需要在Reduce端完成，可以现在Map端进行部分聚合，最后在Reduce端得出最终结果。

**开启Map端聚合参数设置**

+ 是否在 Map 端进行聚合，默认True

```sql
set hive.map.aggr = true;
```

+ 在 Map 端进行聚合操作的条目数目

```sql
set hive.groupby.mapaggr.checkinterval = 100000;
```

+ 有数据倾斜的时候进行负载均衡，默认False

```sql
set hive.groupby.skewindata = true;
```

> **当选项设定为 true时，生成的查询计划会有两个 MR Job。**
> 
> + 第一个 MR Job 中，Map 的输出结果会随机分布到 Reduce 中，每个 Reduce 做部分聚合操作，并输出结果。这样处理的结果是相同的 Group By Key 有可能被分发到不同的 Reduce中，从而达到负载均衡的目的
> + 第二个 MR Job 再根据预处理的数据结果按照 Group By Key 分布到 Reduce 中（这个过程可以保证相同的 Group By Key 被分布到同一个 Reduce 中），最后完成最终的聚合操作
> + 虽然能解决数据倾斜，但是不能让运行速度的更快（启动MR任务时间长）

## 3）Vectorization

矢量计算的技术，在计算类似`scan`,` filter`,` aggregation`的时候，`vectorization`技术以设置批处理的增量大小为 1024 行单次来达到比单条记录单次获得更高的效率。

```sql
set hive.vectorized.execution.enabled = true;
set hive.vectorized.execution.reduce.enabled = true;
```

## 4）多重模式

多行SQL的模式一样，从同一个表进行扫描。

```sql
insert int t_ptn partition(city=A). select id,name,sex, age from student 
where city= A;
insert int t_ptn partition(city=B). select id,name,sex, age from student 
where city= B;
insert int t_ptn partition(city=c). select id,name,sex, age from student 
where city= c;
```

等同于：

```sql
from student
insert int t_ptn partition(city=A) select id,name,sex, age where city= A
insert int t_ptn partition(city=B) select id,name,sex, age where city= B
insert int t_ptn partition(city=C) select id,name,sex, age where city= C
```

## 5）In/Exists

+ `In/Exists`

```sql
select a.id, a.name from a where a.id in (select b.id from b);
select a.id, a.name from a where exists (select id from b where a.id = 
b.id);
```

+ 用`Join`改写

```sql
select a.id, a.name from a join b on a.id = b.id;
```

+ `left semi join `实现

```sql
select a.id, a.name from a left semi join b on a.id = b.id;
```

## 6）CBO优化

Cost based Optimizer：对 HQL 执行计划进行优化，自动优化 HQL 中多个 Join 的顺序，并选择合适的 Join 算法

```sql
set hive.cbo.enable=true;
set hive.compute.query.using.stats=true;
set hive.stats.fetch.column.stats=true;
set hive.stats.fetch.partition.stats=true;
```

## 7）谓词下推

**将 SQL 语句中的 `where` 谓词逻辑都尽可能提前执行，减少下游处理的数据量**。对应逻辑优化器是 `PredicatePushDown`，配置项为 `hive.optimize.ppd`，默认为 true

```sql
set hive.optimize.ppd = true;
```

+ 先关联两张表，再用 where 条件过滤的执行计划

```sql
explain select o.id
from bigtable b join bigtable o
on o.id = b.id
where o.id <= 10;
```

+ 先子查询过滤，再关联

```sql
explain select b.id from bigtable b
join (select id from bigtable where id <= 10) o on b.id = o.id;
```

> 结果都相同，执行计划中都是先过滤，再关联，前提是`join`条件与`where`过滤条件相同

## 8）Map Join （大表 Join 小表）

Map Join 是将 Join 双方中比较小的表直接分发到各个 Map 进程的内存中，在 Map 进程中进行 Join 操 作，这样就不用进行 Reduce 步骤，从而提高速度。**（大表 `join`小 表，大表Map，小表进内存）**

默认情况下，在 Reduce 阶段完成 Join。容易发生数据倾斜。

+ 设置自动选择 Map Join ，默认true

```sql
set hive.auto.convert.join=true;
```

+ 大表小表的阈值设置，默认25M以下认为是小表

```sql
set hive.mapjoin.smalltable.filesize=25000000;
```

> 注意：**若小表(左连接)作为主表，所有数据都要写出去，此时会走 reduce，`map join`失效**
> 
> 正确用法：大表` join `小表

## 9）SMB Join （大表 Join 大表）

Sort Merge Bucket Join：创建大表时使用分桶，并使分桶数为倍数关系

## 10）笛卡尔积

Join 的时候不加 on 条件，或者无效的 on 条件，因为找不到 Join key，Hive 只能使用 1 

个 Reducer 来完成笛卡尔积

+ 设置严格模式，不允许出现笛卡尔积，默认nonstrict

```sql
set hive.mapred.mode=strict;
```

# 4. Hive Job优化

## 1）Map优化

### 复杂文件增加Map数

当 input 的文件都很大，任务逻辑复杂，map 执行非常慢的时候，可以考虑增加 Map 数来使得每个 map 处理的数据量减少，从而提高任务的执行效率。

> 计算公式：`computeSliteSize(Math.max(minSize,Math.min(maxSize,blocksize)))=blocksize=128M`
> 
> + 调小`maxSize`，增加map个数

### 小文件进行合并

**在 map 执行前合并小文件，减少 map 数**

```sql
set hive.input.format=org.apache.hadoop.hive.ql.io.CombineHiveInputFormat;
```

**在MR任务结束时合并小文件**

+ 在 map-only 任务结束时合并小文件，默认 true

```sql
set hive.merge.mapfiles = true;
```

+ 在 map-reduce 任务结束时合并小文件，默认 false

```sql
set hive.merge.mapredfiles = true;
```

+ 合并文件的大小，默认256M

```sql
set hive.merge.size.per.task = 268435456;
```

+ 当输出文件的平均大小小于该值时，启动一个独立的 map-reduce 任务进行文件 merge

```sql
set hive.merge.smallfiles.avgsize = 16777216;
```

### Map端聚合

相当于map端执行combiner

```sql
set hive.map.aggr=true;
```

### 推测执行

```sql
set mapred.map.tasks.speculative.execution = true;
```

## 2）Reduce优化

### 合理设置Reduce数

**调整reduce个数方法一**：

+ 每个 Reduce 处理的数据量默认是 256MB

```sql
set hive.exec.reducers.bytes.per.reducer = 256000000;
```

+ 每个任务最大的 reduce 数，默认为 1009

```sql
set hive.exec.reducers.max = 1009;
```

+ 计算 reducer 数的公式

```sql
N=min(参数 2，总输入数据量/参数 1)(参数 2 指的是上面的 1009，参数 1 值得是 256M)
```

**调整reduce个数方法二**：

```sql
set mapreduce.job.reduces = 15;
```

**reduce 个数并不是越多越好**：

+ 过多的启动和初始化 reduce 也会消耗时间和资源
+ 有多少个 reduce，就会有多少个输出文件，如果生成了很多个小文件，那么这些小文件作为下一个任务的输入，也会出现小文件过多的问题

## 3）Hive任务整体优化

### Fetch抓取

Hive 中对某些情况的查询可以不必使用 MapReduce 计算，例如：`SELECT \* FROM emp;`在这种情况下，Hive 可以简单地读取 `emp `对应的存储目录下的文件，然后输出查询结果到控制台

+ 修改`hive-default.xml.template`：`hive.fetch.task.conversion` 默认是 `more`，老版本 hive 默认是 `minimal`，该属性修改为` more` 以后，在全局查找、字段查找、limit 查找等都不走MR。

```xml
<property>
    <name>hive.fetch.task.conversion</name>
    <value>more</value>
    <description>
    Expects one of [none, minimal, more].
    Some select queries can be converted to single FETCH task minimizing latency.
    Currently the query should be single sourced not having any subquery and should not have any aggregations or distincts (which incurs RS), lateral views and joins.
    0. none : disable hive.fetch.task.conversion
    1. minimal : SELECT STAR, FILTER on partition columns, LIMIT only
    2. more : SELECT, FILTER, LIMIT only (support TABLESAMPLE and virtual columns)
    </description>
</property>
```

### 本地模式

大多数的 Hadoop Job 是需要 Hadoop 提供的完整的可扩展性来处理大数据集的。不过，有时 Hive 的输入数据量是非常小的。在这种情况下，为查询触发执行任务消耗的时间可能会比实际 job 的执行时间要多的多。对于大多数这种情况，Hive 可以通过本地模式在单台机器上处理所有的任务。对于小数据集，执行时间可以明显被缩短。

+ 自动启动本地模式

```sql
set hive.exec.mode.local.auto=true; //开启本地 mr

//设置 local mr 的最大输入数据量，当输入数据量小于这个值时采用 local mr 的方式，默认为 134217728，即 128M
set hive.exec.mode.local.auto.inputbytes.max=50000000;

//设置 local mr 的最大输入文件个数，当输入文件个数小于这个值时采用 local mr 的方式，默认为 4
set hive.exec.mode.local.auto.input.files.max=10;
```

## 4）并行执行

Hive 会将一个查询转化成一个或者多个阶段。这样的阶段可以是 MapReduce 阶段、抽样阶段、合并阶段、limit 阶段。或者 Hive 执行过程中可能需要的其他阶段。默认情况下，Hive 一次只会执行一个阶段。不过，某个特定的 job 可能包含众多的阶段，而这些阶段可能并非完全互相依赖的，也就是说有些阶段是可以并行执行的，这样可能使得整个 job 的执行时间缩短。不过，如果有更多的阶段可以并行执行，那么 job 可能就越快完成。

**前提：系统资源比较空闲**

```sql
set hive.exec.parallel=true; //打开任务并行执行，默认为 false
set hive.exec.parallel.thread.number=16; //同一个 sql 允许最大并行度，默认为 8
```

## 5）严格模式

禁止一些危险操作：

**分区表不使用分区过滤**

+ 除非 where 语句中含有分区字段过滤条件来限制范围，否则不允许执行（即不允许扫描所有分区）

```sql
set hive.strict.checks.no.partition.filter=true;
```

> 原因：通常分区表都拥有非常大的数据集，而且数据增加迅速。没有进行分区限制的查询可能会消耗令人不可接受的巨大资源来处理这个表

**使用`order by` 没有`limit`过滤**

+ 使用了 `order by `语句的查询，要求必须使用 `limit` 语句

```sql
set hive.strict.checks.orderby.no.limit=true;
```

> 原因：`order by `为了执行排序过程，会将所有的结果数据分发到同一个Reducer 中进行处理。强制要求用户增加`limit`语句，可以防止 Reducer 额外执行很长一段时间（**开启了 `limit` 可以在数据进入到 reduce 之前就减少一部分数据**）

**笛卡尔积**

# 5. Hive On Spark

# 6. 优化思路总结

## 建模、建表

+ 对表进行压缩
+ 设计模型适当冗余（？）
+ 合理使用表存储格式
+ 合理设置表的分区

## Insert语句

+ 同源多重insert（多重模式）
+ 避免频繁对一个表执行小数据量的insert

## 参数调优

+ Map端聚合

+ Group By开启负载均衡

+ 合理设置Reduce个数

+ 开启Map Join

+ CBO（Cost based Optimizer）

+ 输入输出小文件合并

+ Vectorization向量参数

+ 并行执行

+ JVM重用

## select语句

+ 总体select策略
  + 避免使用*
  + 分区表使用分区键过滤
  + 大表排序，使用`cluster by` 或者 `distribute by sort by`
  + 设置严格模式
+ 多表关联
  + 提前过滤（谓词下推）
  + 提前计算
  + 提前聚合
  + Map Join
  + SMB Join
  + left semi join
  + CBO优化
  + 避免笛卡尔积
+ 数据倾斜
  + Group By负载均衡
  + 开启Map端聚合
  + `count(distinct)`替代方案
