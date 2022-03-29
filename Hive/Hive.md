# 1. Hive基本概念

## 简介

+ 由Facebook 开源用于解决海量**结构化日志**的数据统计工具
+ 基于 Hadoop 的一个数据仓库工具，可以**将结构化的数据文件映射为一张表**，并 提供类 SQL 查询功能

## 本质

+ 将HQL转化成 MapReduce 程序
+ 数据存储在HDFS
+ 数据分析由MapReduce实现
+ 运行在Yarn

## 优缺点

+ 优点
  + 操作接口采用类 SQL 语法，简单、易上手
  + 避免写MR，减少学习成本（但调优还需要MR知识）
  + 常用于数据分析，对实时性要求不高的场合，执行延迟高
  + 支持用户自定义函数
+ 缺点
  + HQL表达能力有限
    + 迭代式算法无法表达
    + 不擅长数据挖掘，由于MR流程限制，无法实现效率更高的算法
  + 效率低
    + 自动生成MR，通常不够智能
    + 调优困难，粒度较粗

## 架构原理

用户接口：Client

+ CLI（Command-line interface）
+ JDBC/ODBC（JDBC访问 Hive）
+ WEBUI（浏览器访问 Hive）

元数据：Metastore

+ 表名、表所属的数据库（默认是 default）、表的拥有者、列/分区字段、 表的类型（是否是外部表）、表的数据所在目录等
+ 默认存储在自带的 derby 数据库中，推荐使用 MySQL 存储 Metastore

Hadoop

+ 使用 HDFS 进行存储，使用 MapReduce 进行计算

驱动器：Driver

+ 解析器（SQL Parser）：将 SQL 字符串转换成抽象语法树 AST，这一步一般用第 三方工具库完成，如  Antlr ；对 AST 进行语法分析，比如表是否存在、字段是否存在、SQL 语义是否有误
+ 编译器（Physical Plan）：将 AST 编译生成逻辑执行计划
+ 优化器（Query Optimizer）：对逻辑执行计划进行优化
+ 执行器（Execution）：把逻辑执行计划转换成可以运行的物理计划。对于 Hive 来 说，就是 MR/Spark

## 与数据库比较

查询语言：类SQL

数据更新：

+ 数据仓库的内容读多写少
+ Hive 中 不建议对数据的改写，所有的数据都是在加载的时候确定好的
+ 数据库中的数据通常是需要经常进行修改

执行延迟：

+ Hive无索引，查询时需要扫描整个表，延迟较高
+ 启动MapReduce本身也具有较高延迟
+ 数据库的执行延迟较低，然而数据量小

数据规模：

+ Hive可利用MR进行并行计算，支持数据规模大
+ 数据库可支持数据规模小

# 2. Hive安装

## Hive安装

+ 解压 `apache-hive-3.1.2-bin.tar.gz` 到`/opt/module/`目录

```linux
tar -zxvf /opt/software/apache-hive-3.1.2- bin.tar.gz -C /opt/module/
```

+ 修改`/etc/profile.d/my_env.sh`，添加环境变量

```linux
sudo vim /etc/profile.d/my_env.sh

#HIVE_HOME
export HIVE_HOME=/opt/module/hive
export PATH=$PATH:$HIVE_HOME/bin
```

+ 解决日志Jar包冲突

```linux
mv $HIVE_HOME/lib/log4j-slf4j-impl- 2.10.0.jar $HIVE_HOME/lib/log4j-slf4j-impl-2.10.0.bak
```

+ 初始化元数据库

```linux
bin/schematool -dbType derby -initSchema
```

## MySQL安装

+ 检查当前系统是否安装过 MySQL

```linux
[flash7k@hadoop102 ~]$ rpm -qa|grep mariadb
mariadb-libs-5.5.56-2.el7.x86_64

//如果存在通过如下命令卸载
[flash7k @hadoop102 ~]$ sudo rpm -e --nodeps mariadb-libs
```

+ 在`/opt/software`目录下解压MySQL安装包

```linux
tar -xf mysql-5.7.28-1.el7.x86_64.rpm- bundle.tar
```

+ 在安装目录下执行 rpm 安装

```linux
sudo rpm -ivh mysql-community-common-5.7.28-1.el7.x86_64.rpm
sudo rpm -ivh mysql-community-libs-5.7.28-1.el7.x86_64.rpm
sudo rpm -ivh mysql-community-libs-compat-5.7.28-1.el7.x86_64.rpm
sudo rpm -ivh mysql-community-client-5.7.28-1.el7.x86_64.rpm
sudo rpm -ivh mysql-community-server-5.7.28-1.el7.x86_64.rpm
```

> 注意:按照顺序依次执行

+ 清除原来数据：

> 删除`/etc/my.cnf` 文件中 `datadir` 指向的目录下的所有内容
> 
> 如果有内容的情况下:，查看 `datadir` 的值，并删除此目录下内容：

```linux
[mysqld]
datadir=/var/lib/mysql
```

+ 初始化数据库

```linux
sudo mysqld --initialize --user=mysql
```

+ 查看临时生成的 root 用户的密码

```linux
sudo cat /var/log/mysqld.log
```

+ 启动 MySQL 服务

```linux
sudo systemctl start mysqld
```

+ 登录 MySQL 数据库

```linux
mysql -uroot -p
```

+ 若拒绝访问`ERROR 1045 (28000): Access denied for user 'root'@'localhost' (using password: YES)`，则可修改my.in/my.cnf

```linux
//在[mysqld]下添加skip-grant-tables
[mysqld]
skip-grant-tables
```

+ 重启 MySQL服务，免密登录，修改root密码即可

+ 修改 MySQL库下的 user 表中的 root 用户允许任意 ip 连接

```mysql
mysql> update mysql.user set host='%' where user='root'; 
mysql> flush privileges;
```

## Hive元数据配置到MySQL

+ **拷贝驱动**
  
  将 MySQL 的 JDBC 驱动拷贝到 Hive 的 lib 目录下

```linux
cp /opt/software/mysql-connector-java- 5.1.37.jar $HIVE_HOME/lib
```

+ **配置 Metastore 到 MySQL**
  
  + 在`$HIVE_HOME/conf` 目录下新建` hive-site.xml `文件
  
  ```xml
  vim $HIVE_HOME/conf/hive-site.xml
  
  <?xml version="1.0"?>
  <?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
  <configuration>
  
      <!-- jdbc 连接的 URL -->
      <property>
        <name>javax.jdo.option.ConnectionURL</name>
        <value>jdbc:mysql://hadoop102:3306/metastore?useSSL=false</value>
      </property>
  
      <!-- jdbc 连接的 Driver-->
      <property>
        <name>javax.jdo.option.ConnectionDriverName</name>
        <value>com.mysql.jdbc.Driver</value>
      </property>
  
      <!-- jdbc 连接的 username-->
      <property>
        <name>javax.jdo.option.ConnectionUserName</name>
        <value>root</value>
      </property>
  
      <!-- jdbc 连接的 password -->
      <property>
        <name>javax.jdo.option.ConnectionPassword</name>
        <value>123456</value>
      </property>
  
      <!-- Hive 元数据存储版本的验证 -->
      <property>
      <name>hive.metastore.schema.verification</name>
      <value>false</value>
      </property>
  
      <!--元数据存储授权-->
      <property>
      <name>hive.metastore.event.db.notification.api.auth</name>
      <value>false</value>
      </property>
  
      <!-- Hive 默认在 HDFS 的工作目录 -->
      <property>
      <name>hive.metastore.warehouse.dir</name>
      <value>/user/hive/warehouse</value>
      </property>
  </configuration>
  ```
  
  + 登陆 MySQL
  
  ```linux
  mysql -uroot -p123456
  ```
  
  + 新建 Hive 元数据库
  
  ```mysql
  mysql> create database metastore;
  mysql> quit;
  ```
  
  + 初始化 Hive 元数据库
  
  ```linux
  schematool -initSchema -dbType mysql -verbose
  ```

+ **使用元数据服务方式访问Hive**
  
  + 在 `hive-site.xml` 文件中添加如下配置信息
  
  ```xml
  <!-- 指定存储元数据要连接的地址 -->
  <property>
    <name>hive.metastore.uris</name>
    <value>thrift://hadoop102:9083</value>
  </property>
  ```
  
  + 启动 metastore
  
  ```linux
  hive --service metastore
  ```
  
  > 注意: 启动后窗口不能再操作，需打开一个新的 shell 窗口做别的操作

+ **使用 JDBC 方式访问 Hive**
  
  + 在` hive-site.xml` 文件中添加如下配置信息
  
  ```xml
  <!-- 指定 hiveserver2 连接的 host -->
  <property>
    <name>hive.server2.thrift.bind.host</name>
    <value>hadoop102</value>
  </property>
  
  <!-- 指定 hiveserver2 连接的端口号 -->
  <property>
    <name>hive.server2.thrift.port</name>
    <value>10000</value>
  </property>
  ```
  
  + 启动hiveserver2
  
  ```linux
  bin/hive --service hiveserver2
  ```

## Hive服务启动脚本

```sh
#!/bin/bash
HIVE_LOG_DIR=$HIVE_HOME/logs
if [ ! -d $HIVE_LOG_DIR ]
then
    mkdir -p $HIVE_LOG_DIR
fi
#检查进程是否运行正常，参数 1 为进程名，参数 2 为进程端口
function check_process()
{
    pid=$(ps -ef 2>/dev/null | grep -v grep | grep -i $1 | awk '{print$2}')
    ppid=$(netstat -nltp 2>/dev/null | grep $2 | awk '{print $7}' | cut -d '/' -f 1)
    echo $pid
    [[ "$pid" =~ "$ppid" ]] && [ "$ppid" ] && return 0 || return 1
}

function hive_start()
{
    metapid=$(check_process HiveMetastore 9083)
    cmd="nohup hive --service metastore >$HIVE_LOG_DIR/metastore.log 2>&1&"
    [ -z "$metapid" ] && eval $cmd || echo "Metastroe 服务已启动"
    server2pid=$(check_process HiveServer2 10000)
    cmd="nohup hiveserver2 >$HIVE_LOG_DIR/hiveServer2.log 2>&1 &"
    [ -z "$server2pid" ] && eval $cmd || echo "HiveServer2 服务已启动"
}

function hive_stop()
{
    metapid=$(check_process HiveMetastore 9083)
    [ "$metapid" ] && kill $metapid || echo "Metastore 服务未启动"
    server2pid=$(check_process HiveServer2 10000)
    [ "$server2pid" ] && kill $server2pid || echo "HiveServer2 服务未启动"
}

case $1 in
"start")
    hive_start
    ;;
"stop")
    hive_stop
    ;;
"restart")
    hive_stop
    sleep 2
    hive_start
    ;;
"status")
    check_process HiveMetastore 9083 >/dev/null && echo "Metastore 服务运行正常" || echo "Metastore 服务运行异常"
    check_process HiveServer2 10000 >/dev/null && echo "HiveServer2 服务运行正常" || echo "HiveServer2 服务运行异常"
    ;;
*)
    echo Invalid Args!
    echo 'Usage: '$(basename $0)' start|stop|restart|status'
    ;;
esac

//添加执行权限
chmod +x [文件名]
```

## Hive常见属性配置

**运行日志配置**

> 默认在`/tmp/flash7k/hive.log` 目录下（当前用户名下）
> 
> 修改`/opt/module/hive-3.1.2/conf/hive-log4j2.properties.template` 文件名称为`hive-log4j2.properties`，在`hive-log4j2.properties `文件中修改`log`存放位置

```linux
[flash7k@hadoop102 conf]$ pwd
/opt/module/hive-3.1.2/conf
[flash7k@hadoop102 conf]$ mv hive-log4j2.properties.template hive-log4j2.properties

[flash7k@hadoop102 conf]$ vim hive-log4j2.properties
hive.log.dir=/opt/module/hive/logs
```

**打印当前库和表头**

在 `hive-site.xml `中加入如下两个配置

```xml
<property>
  <name>hive.cli.print.header</name>
  <value>true</value>
</property>

<property>
  <name>hive.cli.print.current.db</name>
  <value>true</value>
</property>
```

# 3. Hive数据类型

## 基本数据类型

| Hive数据类型  | Java数据类型 | 长度              | 例子      |
|:---------:|:--------:|:---------------:|:-------:|
| TINYINT   | byte     | 1byte 有符号整数     | 20      |
| SMALINT   | short    | 2byte 有符号整数     | 20      |
| INT       | int      | 4byte 有符号整数     | 20      |
| BIGINT    | long     | 8byte 有符号整数     | 20      |
| BOOLEAN   | boolean  | 布尔类型，true/false | true    |
| FLOAT     | float    | 单精度浮点数          | 3.14159 |
| DOUBLE    | double   | 双精度浮点数          | 3.14159 |
| STRING    | string   | 字符串，可用单引号或双引号   |         |
| TIMESTAMP |          | 时间类型            |         |
| BINARY    |          | 字节数组            |         |

## 集合数据类型

| 数据类型   | 描述                                                              | 语法实例                              |
| ------ | --------------------------------------------------------------- | --------------------------------- |
| STRUCT | STRUCT{first STRING, last STRING}<br />第 1 个元素可以通过字段.first 来 引用 | struct<street:string,city:string> |
| MAP    | 一组键值对元组集合                                                       | map<string, int>                  |
| ARRAY  | 一组具有相同类型和名称的变量的集合                                               | array<string>                     |

## 数据实操

+ 源数据

```json
{
    "name": "songsong",                 //String
    "friends": ["bingbing" , "lili"] ,         //列表Array
    "children": {                        //键值对Map
        "xiao song": 18 , "xiaoxiao song": 19
    }
    "address": {                    //结构Struct
        "street": "hui long guan",
        "city": "beijing"
    }
}
```

+ 创建对应的`test.txt`文件，用于向Hive中导入数据

```sql
songsong,bingbing_lili,xiao song:18_xiaoxiao song:19,hui long guan_beijing
yangyang,caicai_susu,xiao yang:18_xiaoxiao yang:19,chao yang_beijing
```

+ 在Hive中创建表

```sql
create table test(
    name string,
    children map<string, int>,
    address struct<street:string, city:string>
)
row format delimited
fields terminated by ','
collection items terminated by '_'
map keys terminated by ':'
lines terminated by '\n';
```

+ 导入数据

```sql
load data local inpath '/opt/module/hive-3.1.2/datas/test.txt' into table test;
```

# 4. DDL数据定义

## 数据库操作

### 创建数据库

```sql
CREATE DATABASE [IF NOT EXISTS] database_name [COMMENT database_comment]
[LOCATION hdfs_path]
[WITH DBPROPERTIES (property_name=property_value, ...)];
```

+ 数据库在HDFS 上的默认存储路径是`/user/hive/warehouse/*.db`

```sql
create database db_hive;
```

+ 避免要创建的数据库已经存在错误，增加` if not exists `判断（标准写法）

```sql
create database if not exists db_hive;
```

+ 创建一个数据库，指定数据库在 HDFS 上存放的位置

```sql
create database db_hive2 location '/db_hive2.db';
```

### 查询数据库

+ 显示数据库

```sql
show databases;
```

+ 显示数据库信息

```sql
desc database db_hive;
```

+ 显示数据库详细信息：`extended`

```sql
desc database extended db_hive;
```

+ 切换数据库

```sql
use db_hive; 
```

### 修改数据库

```sql
alter database db_hive
set dbproperties('createtime'='20220304');
```

### 删除数据库

+ 删除空数据库

```sql
drop database db_hive2;
```

+ 避免要删除的数据库不存在，增加`if exists` 判断

```sql
drop database if exists db_hive2;
```

+ 删除的数据库不为空，强制删除：`cascade`

```sql
drop database db_hive cascade;
```

## 表操作

### 创建表

```sql
CREATE [EXTERNAL] TABLE [IF NOT EXISTS] table_name
[(col_name data_type [COMMENT col_comment], ...)] [COMMENT table_comment]
[PARTITIONED BY (col_name data_type [COMMENT col_comment], ...)] [CLUSTERED BY (col_name, col_name, ...)
[SORTED BY (col_name [ASC|DESC], ...)] INTO num_buckets BUCKETS] [ROW FORMAT row_format]
[STORED AS file_format] [LOCATION hdfs_path]
[TBLPROPERTIES (property_name=property_value, ...)] [AS select_statement]
```

> **`EXTERNAL`**：外部表，在删除表的时候，内部表的元数据和数据会被一起删除，而外部表只删除元数据，不删除数据。
> 
> **`PARTITIONED BY`**： 分区表
> 
> **`CLUSTERED BY`**：分桶表

+ 普通创建表

```sql
create table if not exists student(
    id int, name string
)
row format delimited
fields terminated by '\t'
stored as textfile
location '/user/hive/warehouse/student';
```

+ 根据查询结果创建表（查询的结果会添加到新创建的表中）：`as`

```sql
create table if not exists student2
as select id, name from student;
```

+ 根据已经存在的表结构创建表：`like`

```sql
create table if not exists student3
like student;
```

+ 查询表类型：`desc formatted`

```sql
desc formatted student2;
```

### 管理表和外部表的互相转换

+ 修改内部表 student2 为外部表

```sql
alter table student2 set tblproperties('EXTERNAL'='TRUE'); 
```

+ 修改外部表 student2 为内部表

```sql
alter table student2 set tblproperties('EXTERNAL'='FALSE'); 
```

> 注意：`('EXTERNAL'='TRUE')`和`('EXTERNAL'='FALSE')`为固定写法，区分大小写！

### 修改表

+ 重命名表

```sql
ALTER TABLE table_name RENAME TO new_table_name;
```

+ 修改列信息：`CHANGE`

```sql
ALTER TABLE table_name CHANGE [COLUMN] col_old_name col_new_name column_type [COMMENT col_comment] [FIRST|AFTER column_name]
```

+ 增加|替换列：`ADD`、`REPLACE`

```sql
ALTER TABLE table_name ADD|REPLACE COLUMNS (col_name data_type [COMMENT col_comment], ...)
```

> `ADD `代表新增一字段，字段位置在所有列后面（partition 列前）
> 
> `REPLACE` 表示替换表中所有字段

### 删除表

```sql
drop table table_name;
```

# 5. DML数据操作

## 数据导入

### 向表中装载数据：Load

```sql
load data [local] inpath 'data_path' [overwrite] into table
table_name [partition (partcol1=val1,…)];
```

### 通过查询语句向表中插入数据：Insert

> **`insert into`**：以追加数据的方式插入到表或分区，原有数据不会删除
> 
> **`insert overwrite`**：会覆盖表中已存在的数据

+ 基本插入数据

```sql
insert into table student_par values(1,'wangwu'),(2,'zhaoliu');
```

+ 插入单张表查询结果

```sql
insert overwrite table student_par
select id, name from student where month='201709';
```

+ 插入多张表查询结果

```sql
from student
insert overwrite table student partition(month='201707')
select id, name where month='201709'
insert overwrite table student partition(month='201706')
select id, name where month='201709';
```

### 创建表时加载查询语句结果数据：As Select

```sql
create table if not exists student3
as select id, name from student;
```

### 创建表时通过Location指定加载数据路径

```sql
create external table if not exists student4( id int, name string
)
row format delimited
fields terminated by '\t'
location '/student;
# 需要提前把文件放到HDFS相应目录下
```

### Import数据到指定Hive表

```sql
import table student2
from '/user/hive/warehouse/export/student';
```

> 注意：先用 export 导出后，再将数据导入

## 数据导出

### Insert导出

+ 将查询的结果导出到本地

```sql
insert overwrite local directory '/opt/module/hive/data/export/student'
select * from student;
```

+ 将查询的结果格式化导出到本地

```sql
insert overwrite local directory '/opt/module/hive/data/export/student1'
ROW FORMAT DELIMITED
FIELDS TERMINATED BY '\t'
select * from student;
```

+ 将查询的结果导出到 HDFS 上（没有 local）

```sql
insert overwrite directory '/user/flash7k/student2'
ROW FORMAT DELIMITED
FIELDS TERMINATED BY '\t'
select * from student;
```

### Hadoop命令导出到本地

```linux
dfs -get /user/hive/warehouse/student/student.txt /opt/module/data/export/student3.txt;
```

### Hive Shell命令导出到本地

基本语法：`hive -f/-e  执行语句或者脚本 > file`

> `-e`：跟随hql语句
> 
> `-f`：跟随hql文件

```linux
bin/hive -e 'select * from default.student;' >
/opt/module/hive/data/export/student4.txt;
```

### Export导出到HDFS上

```sql
export table default.student to 'hdfs_path'
```

> export 和 import 主要用于两个 Hadoop 平台集群之间 Hive 表迁移

### Sqoop导出

## 清除表中数据：Truncate

```sql
truncate table student;
```

> 注意：Truncate 只能删除管理表，不能删除外部表中数据

# 6. 查询

```sql
SELECT [ALL | DISTINCT] select_expr, select_expr, ...
FROM table_reference
[WHERE where_condition]
[GROUP BY col_list]
[ORDER BY col_list]
[CLUSTER BY col_list | [DISTRIBUTE BY col_list]
[SORT BY col_list]
[LIMIT number]
```

> 注意：基本与MySQL类似，因此只记录HQL中的特殊部分

## Like和RLike

选择条件：

+ % 代表任意个字段
+ _ 代表一个字段

> **RLike**：一个Hive中功能的扩展，可以通过 <u>Java 的正则表达式</u>这来指定匹配条件。

## Distribute By

对于 distribute by 进行测试，一定要分配多 reduce 进行处理，否则无法看到 distribute by 的效果。

```sql
hive (default)> set mapreduce.job.reduces=3;

hive (default)> insert overwrite local directory '/opt/module/data/distribute-result'
select * from emp
distribute by deptno
sort by empno desc;
```

> + distribute by 的分区规则是根据分区字段的 hash 码与 reduce 的个数进行模除后， 取余数相同的分到一个区
> + Hive 要求 DISTRIBUTE BY 语句要写在 SORT BY 语句之前

## Cluster By

当 distribute by 和 sorts by 字段相同时，可以使用 cluster by 方式，但只能是升序排序（ASC），不能指定排序规则

+ 以下两种写法等价

```sql
hive (default)> select * from emp cluster by deptno;
hive (default)> select * from emp distribute by deptno sort by deptno;
```

# 7. 分区表

分区表实际上就是对应一个 HDFS 上独立的文件夹，该文件夹下是该分区所有的数据文件。Hive 中的分区就是分目录，把一个大的数据集根据业务需要分割成小的数据集。在查询时通过 WHERE 子句中的表达式选择查询所需要的指定的分区，避免全表扫描，提高查询效率。

## 基本操作

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

## 二级分区

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

## 动态分区调整

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

# 8. 分桶表

分区提供一个隔离数据和优化查询的便利方式。不过，并非所有的数据集都可形成合理的分区。对于一张表或者分区，Hive 可以进一步组织成桶，也就是更为细粒度的数据范围划分。

分桶是将数据集分解成更容易管理的若干部分的另一个技术。 **分区针对的是数据的存储路径；分桶针对的是数据文件。**

## 基本操作

+ 创建分桶表

```sql
create table stu_buck(id int, name string)
clustered by(id)
into 4 buckets
row format delimited fields terminated by '\t';
```

> **分桶规则：Hive 的分桶采用对分桶字段的值进行哈希，然后除以桶的个数求余的方式决定该条记录存放在哪个桶当中**

## 注意事项

+ 设置reduce 的个数为-1,让 Job 自行决定需要用多少个 reduce 或者将 reduce 的个数设置为大于等于分桶表的桶数
+ 从HDFS 中 load 数据到分桶表中，避免本地文件找不到问题
+ 不要使用本地模式

## 抽样查询

对于非常大的数据集，有时用户需要使用的是一个具有代表性的查询结果而不是全部结 果。Hive 可以通过对表进行抽样来满足这个需求。

```sql
select * from stu_buck tablesample(bucket 1 out of 4 on id);
```

> 语法：`TABLESAMPLE(BUCKET x OUT OF y)`
> 
> 注意：x的值必须小于等于y
> 
> 解释：查询样本大小约为1/y，y需要是创建表时指定的桶数的倍数或因子。Hive将桶分为y个桶组，选择每个组的第x个桶

# 9. 函数

## 系统内置函数

+ 查看系统自带的函数

```sql
show functions;
```

+ 显示自带的函数的用法

```sql
desc function upper;
```

+ 详细显示自带的函数的用法

```sql
desc function extended upper;
```

## 常用内置函数

### 空字段赋值：NVL

给值为 NULL 的数据赋值

格式：` NVL( value，default_value )`

功能：如果 value 为 NULL，则 NVL 函数返回 default_value 的值，否则返回 value 的值，如果两个参数 都为 NULL  ，则返回 NULL。

### CASE WHEN THEN ELSE END

```sql
select dept_id,
sum(case sex when '男' then 1 else 0 end) male_count,
sum(case sex when '女' then 1 else 0 end) female_count
from emp_sex
group by dept_id;
```

### 行转列：CONCAT

`CONCAT(string A/col, string B/col…)`：返回输入字符串连接后的结果，支持任意个输入字 符串

`CONCAT_WS(separator, str1, str2,...)`：第一个参数是剩余参数间的分隔符。分隔符可以是与剩余参数一样的字符串。如果分隔符是 NULL，返回值也将为 NULL。这个函数会跳过分隔符参数后的任何 NULL 和空字符串。分隔符将被加到被连接的字符串之间

`COLLECT_SET(col)`：函数只接受基本数据类型，它的主要作用是将某字段的值进行去重汇总，产生 Array 类型字段。

+ 样例
  
  + 数据准备
    
    | name | constellation | blood_type |
    | ---- | ------------- | ---------- |
    | 孙悟空  | 白羊座           | A          |
    | 大海   | 射手座           | A          |
    | 宋宋   | 白羊座           | B          |
    | 猪八戒  | 白羊座           | A          |
    | 柏原崇  | 射手座           | A          |
    | 崔宇植  | 白羊座           | B          |
  
  + 需求：将星座和血型一样的人归类到一起
    
    ```txt
    射手座,A    大海|柏原崇
    白羊座,A    孙悟空|猪八戒
    白羊座,B    宋宋|崔宇植
    ```
  
  + SQL：先拼接字段，再按字段分组
    
    ```sql
    SELECT
        t1.c_b, CONCAT_WS("|",collect_set(t1.name))
    FROM (
        SELECT
            NAME,
            CONCAT_WS(',',constellation,blood_type) c_b FROM person_info
        )t1
    GROUP BY t1.c_b;
    ```

### 列转行：EXPLODE

`EXPLODE(col)`：将 Hive 一列中复杂的 Array 或者 Map 结构拆分成多行。

`LATERAL VIEW udtf(expression) tableAlias AS columnAlias`：和 split, explode 等 UDTF 一起使用，它能够将一列数据拆成多行数据，在此基础上可以对拆分后的数据进行聚合。

+ 样例
  
  + 数据准备
    
    | movie       | category       |
    | ----------- | -------------- |
    | 《疑犯追踪》      | 悬疑,动作,科幻,剧情    |
    | 《Lie to me》 | 悬疑,警匪,动作,心理,剧情 |
    | 《战狼 2》      | 战争,动作,灾难       |
  
  + 需求
    
    ```txt
    《疑犯追踪》    悬疑
    《疑犯追踪》    动作
    《疑犯追踪》    科幻
    《疑犯追踪》    剧情
    《Lie to me》    悬疑
    《Lie to me》    警匪
    《Lie to me》    动作
    《Lie to me》    心理
    《Lie to me》    剧情
    《战狼 2》    战争
    《战狼 2》    动作
    《战狼 2》    灾难
    ```
  
  + SQL：先对字段做切片，再拆分成行
    
    ```sql
    SELECT
        movie, category_name
    FROM
        movie_info
    lateral VIEW
        explode(split(category,",")) movie_info_tmp AS category_name;
    ```

### 窗口函数（开窗）

`OVER()`：指定分析函数工作的数据窗口大小，这个数据窗口大小可能会随着行的变而变化。

+ `CURRENT ROW`：当前行
+ `n PRECEDING`：往前 n 行数据
+ `n FOLLOWING`：往后 n 行数据
+ `UNBOUNDED`：起点
  + `UNBOUNDED PRECEDING`：表示从前面的起点
  + `UNBOUNDED FOLLOWING`：表示到后面的终点

`LAG(col,n,default_val)`：往前第 n 行数据，若无数据则为`default_val`

`LEAD(col,n, default_val)`：往后第 n 行数据，若无数据则为`default_val`

`NTILE(n)`：把有序窗口的行分发到指定数据的组中，各个组有编号，编号从 1 开始，对于每一行，NTILE 返回此行所属的组的编号。（注意：n 必须为 int 类型）

**样例**：

+ 数据准备：name，orderdate，cost
  
  ```txt
  jack,2017-01-01,10
  tony,2017-01-02,15
  jack,2017-02-03,23
  tony,2017-01-04,29
  jack,2017-01-05,46
  jack,2017-04-06,42
  tony,2017-01-07,50
  jack,2017-01-08,55
  mart,2017-04-08,62
  mart,2017-04-09,68
  neil,2017-05-10,12
  mart,2017-04-11,75
  neil,2017-06-12,80
  mart,2017-04-13,94
  ```

+ 需求
  
  + 查询在 2017 年 4 月份购买过的顾客及总人数
  + 查询顾客的购买明细及月购买总额
  + 在上述的场景,  将每个顾客的 cost 按照日期进行累加
  + 查询每个顾客上次的购买时间
  + 查询前 20%时间的订单信息

+ 实现
  
  + 查询在 2017 年 4 月份购买过的顾客及总人数
    
    ```sql
    select name,count(*) over ()
    from business
    where substring(orderdate,1,7) = '2017-04'
    group by name;
    ```
    
    > `over`默认窗口大小为当前表大小
  
  + 查询顾客的购买明细及月购买总额
    
    ```sql
    select 
        name,orderdate,cost,
        sum(cost) over(partition by month(orderdate))
    from business;
    ```
    
    > `partition by month`：按月分区
  
  + 在上述的场景,  将每个顾客的 cost 按照日期进行累加
    
    ```sql
    select
        name,orderdate,cost,
        sum(cost) over(partition by name order by orderdate), ——按name分区，组内数据累加
        sum(cost) over(partition by name order by orderdate rows between UNBOUNDED PRECEDING and current row ), ——和上一组一样，由起点到当前行（默认）
        sum(cost) over(partition by name order by orderdate rows between 1 PRECEDING and current row), ——从前一行到当前行聚合
        sum(cost) over(partition by name order by orderdate rows between 1 PRECEDING AND 1 FOLLOWING ), ——从前一行到后一行
        sum(cost) over(partition by name order by orderdate rows between current row and UNBOUNDED FOLLOWING ) --从当前行到后面所有行
    from business;
    ```
    
    > rows 必须跟在 order by  子句之后，对排序的结果进行限制，使用固定的行数来限制分 区中的数据行数量
  
  + 查询每个顾客上次的购买时间
    
    ```sql
    select
        name,orderdate,cost,
        lag(orderdate,1,'1900-01-01') over(partition by name order by orderdate ) as time1,
        lag(orderdate,2) over (partition by name order by orderdate) as time2
    from business;
    ```
    
    > 若向前无数据，则默认取当前行数据
  
  + 查询前 20%时间的订单信息
    
    ```sql
    select * from (
        select
            name,orderdate,cost,
            ntile(5) over(order by orderdate) sorted
        from business
    ) t
    where sorted = 1;
    ```

### Rank

`RANK()`：排序相同时会重复，总数不会变

`DENSE_RANK()`：排序相同时会重复，总数会减少

`ROW_NUMBER()`：会根据顺序计算

+ 样例
  
  + 需求：计算每门学科成绩排名
  
  + 实现
    
    ```sql
    select
        name,subject, score,
        rank() over(partition by subject order by score desc) rp,
        dense_rank() over(partition by subject order by score desc) drp,
        row_number() over(partition by subject order by score desc) rmp
    from score;
    
    #查询结果如下：
    name    subject    score    rp    drp    rmp
    孙悟空        数学        95    1    1    1
    宋宋        数学        85    2    2    2
    婷婷        数学        65    3    3    3
    大海        数学        65    3    3    4
    猪八戒        数学        60    5    4    5
    小明        数学        55    6    5    6
    ```

## 用户自定义函数

# 10. Hadoop压缩配置

## 开启Map输出阶段压缩（MR引擎）

开启 Map 输出阶段压缩可以减少 Job 中 MapTask 和 ReduceTask 间数据传输量

+ 开启 Hive 中间传输数据压缩功能

```sql
set hive.exec.compress.intermediate=true;
```

+ 开启 MapReduce 中 map 输出压缩功能

```sql
set mapreduce.map.output.compress=true;
```

+ 设置 MapReduce 中 map 输出数据的压缩方式

```sql
set mapreduce.map.output.compress.codec= org.apache.hadoop.io.compress.SnappyCodec;
```

> 通过历史服务器log查看是否压缩

## 开启Reduce输出阶段压缩

当 Hive将输出写入到表中时，输出内容同样可以进行压缩。属性`hive.exec.compress.output `控制着这个功能。用户保持默认设置文件中的默认值 `false`， 默认的输出就是非压缩的纯文本文件。用户可以通过在查询语句或执行脚本中设置这个值为 true，来开启输出结果压缩功能。

+ 开启 Hive 最终输出数据压缩功能

```sql
set hive.exec.compress.output=true; 
```

+ 开启 MapReduce 最终输出数据压缩

```sql
set mapreduce.output.fileoutputformat.compress=true; 
```

+ 设置 MapReduce 最终数据输出压缩方式

```sql
set mapreduce.output.fileoutputformat.compress.codec = org.apache.hadoop.io.compress.SnappyCodec;
```

+ 设置 MapReduce 最终数据输出压缩为块压缩

```sql
set mapreduce.output.fileoutputformat.compress.type=BLOCK;
```

## 文件存储格式

Hive 支持的存储数据的格式主要有：TEXTFILE 、SEQUENCEFILE、ORC、PARQUET。

+ 行存储
  + 查询满足条件的一整行数据的时候，列存储则需要去每个聚集的字段找到对应的每个列的值，行存储只需要找到其中一个值，其余的值都在相邻地方，所以此时行存储查询的速度更快
  + TEXTFILE 、 SEQUENCEFILE 
+ 列存储
  + 因为每个字段的数据聚集存储，在查询只需要少数几个字段的时候，能大大减少读取的数据量；每个字段的数据类型一定是相同的，列式存储可以针对性的设计更好的设计压缩算法。
  + ORC 、 PARQUET

### TextFile

默认格式，数据不做压缩，磁盘开销大，数据解析开销大。

可结合 Gzip、Bzip2 使用。 但使用 Gzip ，Hive 不会对数据进行切分，从而无法对数据进行并行操作。

### ORC

每个 Orc 文件由 1 个或多个 stripe 组成，每个 stripe 一般为 HDFS的块大小，每一个 stripe 包含多条记录，这些记录按照列进行独立存储，对应到 Parquet 中的 row group 的概念。每个Stripe 里有三部分组成，分别是 `Index Data`，`Row Data`，`Stripe Footer`

+ `Index Data`：一个轻量级的 index，**默认是每隔 1W 行做一个索引**。这里做的索引应该只是记录某行的各字段在 `Row Data `中的 `offset`。（防止单个字段行数过多，数据倾斜）

+ `Row Data`：存储具体的数据。先取部分行，然后对这些行按列进行存储。对每个列进行编码，分成多个 `Stream `来存储。

+ `Stripe Footer`：存的是各个 `Stream` 的类型，长度等信息。每个文件有一个 `File Footer`，存的是每个` Stripe `的行数，每个` Column` 的数据类型信息等。每个文件的尾部是一个 `PostScript`，记录了整个文件的压缩类型以及 `File Footer` 的长度信息等。在读取文件时，会 seek 到文件尾部读 `PostScript`，从里面解析到 `File Footer `长度，再读` File Footer`，从里面解析到各个 `Stripe `信息，再读各个 `Stripe`，即从后往前读。

## 主流文件存储格式对比

+ 创建表，存储数据格式为 TEXTFILE

```sql
create table log_text (
    track_time string,
    url string,
    session_id string,
    referer string,
    ip string,
    end_user_id string,
    city_id string
)
row format delimited
fields terminated by '\t'
stored as textfile; #指定存储格式
```

> 表大小约为18.13M

+ 创建表，存储数据格式为 ORC

```sql
create table log_orc (
    track_time string,
    url string,
    session_id string,
    referer string,
    ip string,
    end_user_id string,
    city_id string
)
row format delimited
fields terminated by '\t'
stored as orc
tblproperties("orc.compress"="NONE"); #设置orc存储不使用压缩
tblproperties("orc.compress"="SNAPPY"); #设置orc存储使用SNAPPY压缩
```

> 表大小约为7.7M

+ 创建表，存储数据格式为 parquet

```sql
create table log_parquet (
    track_time string,
    url string,
    session_id string,
    referer string,
    ip string,
    end_user_id string,
    city_id string
)
row format delimited
fields terminated by '\t'
stored as parquet; #指定存储格式
```

> 表大小约为13.1M

+ 文件大小：ORC >  Parquet >  TEXTFILE
+ 查询速度相近

# HQL练习

**准备工作**

+ 创建原始数据表：`gulivideo_ori`，`gulivideo_user_ori`

```sql
create table gulivideo_ori(
    videoId string,
    uploader string,
    age int,
    category array<string>,
    length int,
    views int,
    rate float,
    ratings int,
    comments int,
    relatedId array<string>)
row format delimited
fields terminated by "\t"
collection items terminated by "&"
stored as textfile;
```

```sql
create table gulivideo_user_ori(
    uploader string,
    videos int,
    friends int)
row format delimited fields terminated by "\t";
```

+ 创建最终表：`gulivideo_orc`，`gulivideo_user_orc`

```sql
create table gulivideo_orc(
    videoId string,
    uploader string,
    age int,
    category array<string>,
    length int,
    views int,
    rate float,
    ratings int,
    comments int,
    relatedId array<string>)
stored as orc;
```

```sql
create table gulivideo_user_orc(
    uploader string,
    videos int,
    friends int)
row format delimited fields terminated by "\t"
stored as orc;
```

**插入数据**

+ 向 `ori` 表插入数据

```sql
load data local inpath "/opt/module/data/video" into table gulivideo_ori; load data local inpath "/opt/module/data/user" into table gulivideo_user_ori;
```

+ 向 `orc` 表插入数据

```sql
insert into table gulivideo_orc select * from gulivideo_ori;
insert into table gulivideo_user_orc select * from gulivideo_user_ori;
```

**数据结构**

+ 视频表

| 字段        | 备注                     | 描述               |
|:--------- | ---------------------- | ---------------- |
| videoId   | 视频唯一 id（String）        | 11 位字符串          |
| uploader  | 视频上传者（String）          | 上传视频的用户名         |
| age       | 视频年龄（int）              | 视频在平台上的整数天       |
| category  | 视频类别（Array<String>）    | 上传视频指定的视频分类      |
| length    | 视频长度（Int）              | 整形数字标识的视频长度      |
| views     | 观看次数（Int）              | 视频被浏览的次数         |
| rate      | 视频评分（Double）           | 满分 5 分           |
| Ratings   | 流量（Int）                | 视频的流量，整型数字       |
| conments  | 评论数（Int）               | 一个视频的整数评论数       |
| relatedId | 相关视频 id（Array<String>） | 相关视频的 id，最多 20 个 |

+ 用户表

| 字段       | 备注     | 字段类型   |
| -------- | ------ | ------ |
| uploader | 上传者用户名 | string |
| videos   | 上传视频数  | int    |
| friends  | 朋友数量   | int    |

**需求**

+ 统计视频观看数Top10

```sql
select videoId,`views`
from gulivideo_orc
order by views desc
limit 10;
```

+ 统计视频类别热度Top10

```sql
# 自己写
select category, count(videoId) from gulivideo_orc
group by category
order by count(videoId)
limit 10;
```

> 思路：
> 
> （1）即统计每个类别有多少个视频，显示出包含视频最多的前 10 个类别。
> 
> （2）我们需要按照类别 group by 聚合，然后 count 组内的 videoId 个数即可。
> 
> （3）因为当前表结构为：一个视频对应一个或多个类别。所以如果要 group by 类别， 需要先将类别进行列转行(展开)，然后再进行 count 即可。
> 
> （4）最后按照热度排序，显示前 10 条。

```sql
# 列转行
select videoId, category_name
from gulivideo_orc
lateral View explode(category) gulivideo_orc_tmp as category_name

# 最终查询
select t1.category_name, count(t1.videoId) hot
from (
select videoId, category_name
from gulivideo_orc
lateral View explode(category) gulivideo_orc_tmp as category_name) t1
group by t1.category_name
order by hot desc
limit 10;
```

+ 统计出视频观看数最高的 20 个视频的所属类别以及类别包含Top20视频的个数

```sql
# 找到观看数最高的20个视频
select videoId, category, views 
from gulivideo_orc
order by views desc
limit 20;

# 按类别打散
select t1.videoId, t1.category_name
from (select videoId, category, views 
from gulivideo_orc
order by views desc
limit 20) t1
lateral view explode(category) t1_tmp as category_name;

# 统计个数
select t2.category_name, count(t2.videoId) video_sum
from 
(select t1.videoId, category_name
from (
    select videoId, category, views 
    from gulivideo_orc
    order by views desc
    limit 20) t1
lateral view explode(category) t1_tmp as category_name) t2
group by t2.category_name;

# 使用窗口函数改写
select distinct category_name, count(t1.videoId) over(partition by category_name) video_sum
from (
    select videoId, category, views 
    from gulivideo_orc
    order by views desc
    limit 20) t1
lateral view explode(t1.category) t1_tmp as category_name;
```

+ 统计视频观看数Top50所关联视频的所属类别排序

```sql
# 找到观看数最高的50个视频和相关视频
select videoId, relatedId
from gulivideo_orc
order by views desc
limit 50;

# 相关视频打散
select related_id
from (
    select videoId, relatedId
    from gulivideo_orc
    order by views desc
    limit 50
) t1
lateral view explode(t1.relatedId) ti_tmp as related_id;

# 找到相关视频的类别
select t2.related_id, t3.category
from (
    select related_id
    from (
        select videoId, relatedId
        from gulivideo_orc
        order by views desc
        limit 50
    ) t1
    lateral view explode(t1.relatedId) ti_tmp as related_id
) t2 join gulivideo_orc t3
on t2.related_id = t3.videoId;

# 类别打散
select t4.related_id, category_name
from (
    select t2.related_id, t3.category
    from (
        select related_id
        from (
            select videoId, relatedId
            from gulivideo_orc
            order by views desc
            limit 50
        ) t1
        lateral view explode(t1.relatedId) ti_tmp as related_id
    ) t2 join gulivideo_orc t3
    on t2.related_id = t3.videoId
) t4
lateral view explode(t4.category) t4_tmp as category_name;

# 计算每个类别中视频个数并排序
# 报错：over里不能用另一个over的别名
select 
    distinct t5.category_name,
    count(t5.related_id) over(partition by t5.category_name) video_sum,
    rank() over(order by video_sum) rank
from (
    select t4.related_id, category_name
    from (
        select t2.related_id, t3.category
        from (
            select related_id
            from (
                select videoId, relatedId, views
                from gulivideo_orc
                order by views desc
                limit 50
            ) t1
            lateral view explode(t1.relatedId) ti_tmp as related_id
        ) t2 join gulivideo_orc t3
        on t2.related_id = t3.videoId
    ) t4
    lateral view explode(t4.category) t4_tmp as category_name
) t5;

# 参考答案
select
    t6.category_name,
    t6.video_sum,
    rank() over(order by t6.video_sum desc) rank
from (
    select t5.category_name, count(t5.relatedid_id) video_sum
    from (
        select t4.relatedid_id, category_name
        from (
            select t2.relatedid_id, t3.category
            from (
                select relatedid_id
                from (
                    select videoId, views, relatedId
                    from gulivideo_orc
                    order by views desc
                    limit 50
                ) t1
                lateral VIEW explode(t1.relatedId) t1_tmp AS relatedid_id
            ) t2 join gulivideo_orc t3
            on t2.relatedid_id = t3.videoId
        ) t4
        lateral VIEW explode(t4.category) t4_tmp AS category_name
    ) t5
    group by t5.category_name
    order by t5.video_sum desc
) t6;
```

+ 统计每个类别中的视频热度Top10，以Music为例

```sql
# 按类别展开
select videoId, views, category_name
from gulivideo_orc
lateral view explode(category) gulivideo_orc_tmp as category_name;

# 统计Music类别，并排序
select t1.videoId, t1.views
from (
    select videoId, views, category_name
    from gulivideo_orc
    lateral view explode(category) gulivideo_orc_tmp as category_name
) t1
where t1.category_name = "Music"
order by t1.views desc
limit 10;
```

+ 统计每个类别视频观看数Top10

```sql
# 按类型打散并排名
select
    t1.videoId, t1.views,t1.category_name
    rank() over(partition by t1.category_name order by t1.views) rk
from (
    select videoId, views, category_name,
    from gulivideo_orc
    lateral view explode(category) gulivideo_orc_tmp as category_name
) t1

# 取排名前10
select t2.videoId, t2.views, t2.category_name, t2.rk
from (
    select
        t1.videoId, t1.views, t1.category_name,
        rank() over(partition by t1.category_name order by t1.views) rk
    from (
        select videoId, views, category_name
        from gulivideo_orc
        lateral view explode(category) gulivideo_orc_tmp as category_name
    ) t1
) t2
where t2.rk <= 10;
```

+ 统计上传视频最多的用户Top10以及他们上传的视频观看次数在前20的视频

```sql
# 上传视频最多的用户Top10
select uploader, videos
from gulivideo_user_orc
order by videos
limit 10;

select t2.videoId, t1.uploader, t2.views
from (
    select uploader, videos
    from gulivideo_user_orc
    order by videos desc
    limit 10
) t1 join gulivideo_orc t2
on t1.uploader = t2.uploader
order by t2.views desc
limit 20;
```
