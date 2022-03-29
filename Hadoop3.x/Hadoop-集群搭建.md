# Hadoop-集群搭建

## 第一章 Hadoop概述

## 1.1 Hadoop是什么

1. 由Apache基金会开发的**分布式系统基础架构**
2. 解决海量数据的**存储**和**分析计算**问题
3. 广义上指**Hadoop生态圈**

## 1.2 Hadoop发展历史

1. 创始人**Doug Cutting**
2. 为实现与Google类似的全文检索，在**Lucene**框架基础上优化升级
3. Lucene和Google存在同样问题：**存储海量数据困难，检索速度慢**
4. Google三篇论文
   + **GFS --> HDFS**
   + **Map-Reduce --> MR**
   + **Big Table --> HBase**

5. 2006年3月，Hadoop诞生

## 1.3 Hadoop三大发行版本

1. **Apache**：最原始（基础）版本，适合入门
2. **Cloudera**：内部集成很大大数据框架，**CDH**
3. **Hortonworks**：现已被Cloudera公司收购，推出新品牌**CDP**

## 1.4. Hadoop优势

### 1.4.1 高可靠性

​		Hadoop底层维护多个数据副本，即使某个计算元素或存储出现故障，也不会导致数据的丢失

### 1.4.2 高扩展性

​		在集群间分配任务数据，可方便的扩展数以千计的节点（高峰期可动态增加服务器）

### 1.4.3 高效性

​		在MapReduce思想下，Hadoop并行工作，加快任务处理速度

### 1.4.4 高容错性

​		能够自动将失败的任务重新分配

## 1.5 Hadoop组成（面试重点）

### 1.5.1 Hadoop1.x 和 2.x 3.x 区别

**1. Hadoop1.x**

+ MapReduce：计算+资源调度
+ HDFS：数据存储
+ Common：辅助工具

**2. Hadoop2.x 3.x**

+ MapReduce：计算
+ **Yarn**：资源调度
+ HDFS：数据存储
+ Common：辅助工具

### 1.5.2 HDFS架构概述

**Hadoop Distributed File System，HDFS，分布式文件系统**

**1. NameNode（nn）**

+ 存储文件的**元数据**：文件名，文件目录结构、文件属性（生产时间、副本数、文件权限）

+ 文件的**块列表**
+ **块所在的DataNode**

**2. DataNode（dn）**

+ 在本地文件系统**存储文件块数据**
+ 在本地文件系统**存储块数据的校验和**

**3. SecibdaryNameNode（2nn）**

+ **每隔一段对NameNode元数据备份**

### 1.5.3 YARN架构概述

**Yet Another Resource Negotiator，Yarn，Hadoop资源管理器**

**1. ResouceManeger（RM）**

+ 整个集群资源（内存、CPU等）的管理者

**2. NodeManeger（NM）**

+ 单个节点服务器资源的管理者

**3. ApplicationMaster（AM）**

+ 单个任务运行资源的管理者

**4. Container**

+ 容器，相当于一台独立的服务器
+ 封装任务运行所需要的资源，如内存、CPU、磁盘、网络等

**5. PS**

+ 客户端可以有多个
+ 集群上可以运行多个ApplicationMaster
+ 每个NodeManeger上可以有多个Container

### 1.5.4 MapReduce架构概述

**计算分为两个阶段：Map和Reduce**

+ Map并行处理输入数据
+ Reduce对Map结果汇总

### 1.5.5 HDFS、YARN、MapReduce三者关系

## 第二章 Hadoop运行环境搭建（开发重点）

### 2.1 模板虚拟机环境准备

1. **安装模板虚拟机**：静态IP地址192.168.10.100、主机名称hadoop100、内存2G、硬盘50G

   1. 主机名修改为hadoop100

   2. 设置root和普通用户，密码简单

   3. **网络配置（重点）**

      1. **设置虚拟机上网方式NAT**

      2. **编辑更改VMware网络配置**

         + 子网IP 192.168.10.0
         + NAT设置网关IP 192.168.10.2

      3. **修改Windows网络配置**

         + 点击更改适配器选项
         + 修改VMnet8  IPV4协议
         + 修改IP地址192.168.10.1，子网掩码255.255.255.0、默认网关192.168.10.2、DNS服务器192.168.10.2

      4. **修改虚拟机网络**

         + **修改网络IP为静态IP**

         ```linux
         [root@hadoop100 ~]#vim /etc/sysconfig/network-scripts/ifcfg-ens33
         ```

         + **配置ens33网卡文件**

         ```linux
         TYPE="Ethernet"    #网络类型（通常是Ethemet）
         PROXY_METHOD="none"
         BROWSER_ONLY="no"
         BOOTPROTO="static"   #IP的配置方法[none|static|bootp|dhcp]（引导时不使用协议|静态分配IP|BOOTP协议|DHCP协议）
         DEFROUTE="yes"
         IPV4_FAILURE_FATAL="no"
         IPV6INIT="yes"
         IPV6_AUTOCONF="yes"
         IPV6_DEFROUTE="yes"
         IPV6_FAILURE_FATAL="no"
         IPV6_ADDR_GEN_MODE="stable-privacy"
         NAME="ens33"   
         UUID="e83804c1-3257-4584-81bb-660665ac22f6"   #随机id
         DEVICE="ens33"   #接口名（设备,网卡）
         ONBOOT="yes"   #系统启动的时候网络接口是否有效（yes/no）
         #IP地址
         IPADDR=192.168.10.100  
         #网关  
         GATEWAY=192.168.10.2      
         #域名解析器，选择当前网络运营商的域名解析器
         DNS1=
         ```

         + **重启虚拟机**

         ```linux
         [root@hadoop100 ~]# systemctl restart network
         ```

         + **查看当前IP是否修改成功 **

         ```linux
         [root@hadoop100 ~]# ifconfig
         ```

         + **确保Linux系统ifcfg-ens33文件中IP地址、虚拟网络编辑器地址和Windows系统VM8网络IP地址相同**

      5. **修改主机名和host文件**

         1. **修改主机名称**

         ```linux
         [root@hadoop100 ~]# vim /etc/hostname
         hadoop100
         ```

         2. **配置Linux克隆机主机名称映射hosts文件，打开/etc/hosts**

         ```linux
         192.168.10.100 hadoop100
         192.168.10.101 hadoop101
         192.168.10.102 hadoop102
         192.168.10.103 hadoop103
         192.168.10.104 hadoop104
         192.168.10.105 hadoop105
         192.168.10.106 hadoop106
         192.168.10.107 hadoop107
         192.168.10.108 hadoop108
         ```

         3. **重启**

         4. **修改Windows主机映射文件（hosts文件）**

            + 进入C:\Windows\System32\drivers\etc
            + 打开hosts，添加以下内容

            ```txt
            192.168.10.100 hadoop100
            192.168.10.101 hadoop101
            192.168.10.102 hadoop102
            192.168.10.103 hadoop103
            192.168.10.104 hadoop104
            192.168.10.105 hadoop105
            192.168.10.106 hadoop106
            192.168.10.107 hadoop107
            192.168.10.108 hadoop108
            ```

            + 拷贝hosts到桌面再覆盖原来的hosts

2. **虚拟机基础配置**

   1. **检查是否能ping外网**
   2. **关闭防火墙，关闭防火墙开机自启**

   ```linux
   [root@hadoop100 ~]# systemctl stop firewalld
   [root@hadoop100 ~]# systemctl disable firewalld.service
   ```

   3. **配置普通用户具有root权限，方便sudo执行root权限命令**

   ```linux
   [root@hadoop100 ~]# vim /etc/sudoers
   
   #修改/etc/sudoers文件，在%wheel这行下面添加一行
   
   ## Allow root to run any commands anywhere
   root    ALL=(ALL)     ALL
   
   ## Allows people in group wheel to run all commands
   %wheel  ALL=(ALL)       ALL
   flash7k   ALL=(ALL)     NOPASSWD:ALL
   ```

    4. **在/opt目录下创建文件夹，并修改所属主和所属组**

       1. 在/opt目录下创建module、software文件夹

       ```linux
       [root@hadoop100 ~]# mkdir /opt/module
       [root@hadoop100 ~]# mkdir /opt/software
       ```

       ​	2. 修改module、software文件夹的所有者和所属组均为flash7k用户 

       ```linux
       [root@hadoop100 ~]# chown flash7k:flash7k /opt/module 
       [root@hadoop100 ~]# chown flash7k:flash7k /opt/software
       ```

       3. 查看module、software文件夹的所有者和所属组

       ```linux
       [root@hadoop100 ~]# cd /opt/
       [root@hadoop100 opt]# ll
       ```
   5. **卸载虚拟机自带的JDK**

   ```linux
   [root@hadoop100 ~]# rpm -qa | grep -i java | xargs -n1 rpm -e --nodeps 
   ```

   6. **重启虚拟机**

### 2.2 克隆虚拟机

### 2.3 在hadoop102安装JDK

1. **卸载虚拟机自带的JDK**
2. **用 XShell 传输工具将 JDK 导入到 /opt/software文件夹下面**
3. **在Linux系统下的opt目录中查看软件包是否导入成功**

```linux
[flash7k@hadoop102 ~]$ ls /opt/software/
jdk-8u212-linux-x64.tar.gz
```

4. **解压JDK到 /opt/module目录下**

```linux
[flash7k@hadoop102 software]$ tar -zxvf jdk-8u212-linux-x64.tar.gz -C /opt/module/
```

5. **配置JDK环境变量**

   + **新建/etc/profile.d/my_env.sh文件（需要root权限）**

   ```sh
   [flash7k@hadoop102 ~]$ sudo vim /etc/profile.d/my_env.sh
   
   #添加以下内容
   #JAVA_HOME
   export JAVA_HOME=/opt/module/jdk1.8.0_212
   export PATH=$PATH:$JAVA_HOME/bin
   ```

   + **source /etc/profile文件，让新的环境变量PATH生效（不要忘记）**

   ```linux
   [flash7k@hadoop102 ~]$ source /etc/profile
   ```

6. **测试JDK是否安装成功**

```linux
[flash7k@hadoop102 ~]$ java -version
java version "1.8.0_212"
```

### 2.4 在hadoop102安装Hadoop

1. **XShell上传到 /opt/software**

2. **解压Hadoop安装包到 /opt/module**
3. **添加Hadoop环境变量**

```linux
[flash7k@hadoop102 hadoop-3.1.3]$ sudo vim /etc/profile.d/my_env.sh

#在文件末尾添加以下内容，注意比JAVA_HOME多一个sbin

#HADOOP_HOME
export HADOOP_HOME=/opt/module/hadoop-3.1.3
export PATH=$PATH:$HADOOP_HOME/bin
export PATH=$PATH:$HADOOP_HOME/sbin
```

4. **测试是否安装成功**

### 2.5 Hadoop目录结构

1. **查看目录结构**

```linux
[flash7k@hadoop102 hadoop-3.1.3]$ ll
总用量 52
drwxr-xr-x. 2 flash7k flash7k  4096 5月  22 2017 bin
drwxr-xr-x. 3 flash7k flash7k  4096 5月  22 2017 etc
drwxr-xr-x. 2 flash7k flash7k  4096 5月  22 2017 include
drwxr-xr-x. 3 flash7k flash7k  4096 5月  22 2017 lib
drwxr-xr-x. 2 flash7k flash7k  4096 5月  22 2017 libexec
-rw-r--r--. 1 flash7k flash7k 15429 5月  22 2017 LICENSE.txt
-rw-r--r--. 1 flash7k flash7k   101 5月  22 2017 NOTICE.txt
-rw-r--r--. 1 flash7k flash7k  1366 5月  22 2017 README.txt
drwxr-xr-x. 2 flash7k flash7k  4096 5月  22 2017 sbin
drwxr-xr-x. 4 flash7k flash7k  4096 5月  22 2017 share
```

2. **重要目录**
   1. bin：存放对Hadoop相关服务（hdfs，yarn，mapred）进行操作的脚本
   2. etc：Hadoop的配置文件目录，存放Hadoop的配置文件
   3. lib：存放Hadoop的本地库（对数据进行压缩解压缩功能）
   4. sbin：存放启动或停止Hadoop相关服务的脚本（自己写的脚本）
   5. share目录：存放Hadoop的依赖jar包、文档、和官方案例WordCount

## 第三章 Hadoop运行模式

+ **本地运行模式**
+ **伪分布式模式**
+ **完全分布式模式**

### 完全分布式运行模式（开发重点）

#### 3.1 配置流程

1. 准备三台客户机（关闭防火墙、静态IP、主机名称）
2. 安装JDK
3. 配置环境变量
4. 安装Hadoop
5. 配置环境变量
6. 配置集群
7. 单点启动
8. 配置ssh
9. 群起并测试集群

#### 3.2 编写集群分发脚本xsync

1. **scp（secure copy）安全拷贝**

   1. **定义**

      scp可以实现服务器与服务器之间的数据拷贝（from server1 to server2）

   2. **前提**

      在hadoop102、hadoop103、hadoop104 都已经创建好

      /opt/module、/opt/software 两个目录

      并且已经把这两个目录修改为flash7k:flash7k

   3. **拷贝给别的服务器**：在hadoop102上，将 hadoop102 中 /opt/module/jdk1.8.0_212 目录拷贝到 hadoop103 

   ```linux
   [flash7k@hadoop102 ~]$ scp -r /opt/module/jdk1.8.0_212  flash7k@hadoop103:/opt/module
   ```

   	4. **从别的服务器抓取拷贝到自己服务器**：在 hadoop103 上，将 hadoop102 中 /opt/module/hadoop-3.1.3 目录拷贝到 hadoop103 

   ```linux
   [flash7k@hadoop103 ~]$ scp -r flash7k@hadoop102:/opt/module/hadoop-3.1.3 /opt/module/
   ```

   	5. **拷贝目录下所有目录**

   ```linux
   [flash7k@hadoop103 opt]$ scp -r atguigu@hadoop102:/opt/module/* flash7k@hadoop104:/opt/module
   ```

2. **rsync远程同步工具**

   1. **定义**

      rsync主要用于备份和镜像。优点：速度快、避免复制相同内容、支持符号链接

   2. **优点**

      + 速度快
      + 避免复制相同内容
      + 支持符号链接

   3. **操作说明**

   ```linux
   [flash7k@hadoop102 module]$ rsync -av hadoop-3.1.3/ flash7k@hadoop103:/opt/module/hadoop-3.1.3/
   
   -a 归档拷贝
   -v 显示复制过程
   ```

3. **xsync集群分发脚本**

   1. **需求**

      + 循环复制文件到所有节点的相同目录下
      + 脚本在任何路径都能使用

   2. **脚本实现**

      1. **在/home/flash7k/bin目录下创建xsync文件**

      ```linux
      [flash7k@hadoop102 opt]$ cd /home/flash7k
      [flash7k@hadoop102 ~]$ mkdir bin
      [flash7k@hadoop102 ~]$ cd bin
      [flash7k@hadoop102 bin]$ vim xsync
      
      #在该文件中编写如下代码
      
      #!/bin/bash
      
      #1. 判断参数个数
      if [ $# -lt 1 ]
      then
          echo Not Enough Arguement!
          exit;
      fi
      
      #2. 遍历集群所有机器
      for host in Cloud101 Cloud102 Cloud103
      do
          echo ====================  $host  ====================
          #3. 遍历所有目录，挨个发送
      
          for file in $@
          do
              #4. 判断文件是否存在
              if [ -e $file ]
                  then
                      #5. 获取父目录
                      pdir=$(cd -P $(dirname $file); pwd)
      
                      #6. 获取当前文件的名称
                      fname=$(basename $file)
                      ssh $host "mkdir -p $pdir"
                      rsync -av $pdir/$fname $host:$pdir
                  else
                      echo $file does not exists!
              fi
          done
      done
      ```

      2. **修改脚本 xsync 具有执行权限**

      ```linux
      [flash7k@hadoop102 bin]$ chmod +x xsync
      ```

      3. **测试脚本**

      ```linux
      [flash7k@hadoop102 ~]$ xsync /home/flash7k/bin
      ```

      4. **将脚本复制到/bin中，以便全局调用**

      ```linux
      [flash7k@hadoop102 bin]$ sudo cp xsync /bin/
      ```

      5. **同步环境变量配置（root所有者）**

         *注意：*使用sudo，一定要把 xsync 的路径补全

      ```linux
      [flash7k@hadoop102 ~]$ sudo ./bin/xsync /etc/profile.d/my_env.sh
      ```

      6. **让环境变量生效**

      ```linux
      [flash7k@hadoop103 bin]$ source /etc/profile
      [flash7k@hadoop104 opt]$ source /etc/profile
      ```

#### 3.3 SSH无密登录配置

1. **免密登录原理**
   1. A服务器执行 ssh-keygen **生成密钥对**：公钥、私钥
   2. A**公钥拷贝**到B服务器
   3. A服务器ssh访问B，数据使用**私钥A加密**
   4. B服务器接收数据，查找并使用**公钥A解密**数据
   5. B服务器返回数据，数据使用**公钥A加密**
2. **生成密钥对**

```linux
[flash7k@hadoop102 .ssh]$ pwd
/home/flash7k/.ssh

[flash7k@hadoop102 .ssh]$ ssh-keygen -t rsa
```

3. **将公钥拷贝到要免密登录的目标机器上**

```linux
[flash7k@hadoop102 .ssh]$ ssh-copy-id hadoop102
[flash7k@hadoop102 .ssh]$ ssh-copy-id hadoop103
[flash7k@hadoop102 .ssh]$ ssh-copy-id hadoop104
```

4. **还需在其他服务器和root账号上配置免密登录**
5. **.ssh文件夹下（~/.ssh）的文件功能解释**

文件名|功能
---|:---
known_hosts|记录ssh访问过计算机的公钥（public key）
id_rsa|生成的私钥
id_rsa.pub|生成的公钥
authorized_keys|存放授权过的无密登录服务器公钥

#### 3.4 集群配置

1. **集群部署规划**
   + NameNode 和 SecondaryNameNode不要安装在同一台服务器
   + ResourceManager也很消耗内存，不要和NameNode、SecondaryNameNode配置在同一台机器上

|      | hadoop102   | hadoop103       | hadoop104         |
| ---- | ----------- | --------------- | ----------------- |
|      | NameNode    | DataNode        | SecondaryNameNode |
|      |             | ResourceManager | DataNode          |
|      | NodeManager |                 | NodeManager       |

2. **配置文件说明**

   1. 默认配置文件
      + core-default.xml
      + hdfs-default.xml
      + yarn-default.xml
      + mapred-default.xml
   2. 自定义配置文件（位于 **$HADOOP_HOME/etc/hadoop** ）
      + **core-site.xml**
      + **hdfs-site.xml**
      + **yarn-site.xml**
      + **mapred-site.xml**

3. **配置集群**

   1. **core-site.xml**

   ```xml
   [flash7k@hadoop102 ~]$ cd $HADOOP_HOME/etc/hadoop
   [flash7k@hadoop102 hadoop]$ vim core-site.xml
   
   
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

   2. **hdfs-site.xml**

   ```xml
   [flash7k@hadoop102 hadoop]$ vim hdfs-site.xml
   
   
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

   3. **yarn-site.xml**

   ```linux
   [flash7k@hadoop102 hadoop]$ vim yarn-site.xml
   
   
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
   </configuration>
   ```

   4. **mapred-site.xml**

   ```linux
   [flash7k@hadoop102 hadoop]$ vim mapred-site.xml
   
   
   
   <?xml version="1.0" encoding="UTF-8"?>
   <?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
   
   <configuration>
   	<!-- 指定MapReduce程序运行在Yarn上 -->
       <property>
           <name>mapreduce.framework.name</name>
           <value>yarn</value>
       </property>
   </configuration>
   ```

4. **在集群上分发配置好的Hadoop配置文件**

```linux
[flash7k@hadoop102 hadoop]$ xsync /opt/module/hadoop-3.1.3/etc/hadoop
```

5. **去103和104上查看文件分发情况**

```linux
[flash7k@hadoop103 ~]$ cat /opt/module/hadoop-3.1.3/etc/hadoop/core-site.xml
[flash7k@hadoop104 ~]$ cat /opt/module/hadoop-3.1.3/etc/hadoop/core-site.xml
```

#### 3.5 群起集群

1. **配置workers**

```linux
[flash7k@hadoop102 hadoop]$ vim /opt/module/hadoop-3.1.3/etc/hadoop/workers

#增加以下内容：
hadoop102
hadoop103
hadoop104
```

**注意：该文件中添加的内容结尾不允许有空格，文件中不允许有空行**

同步所有节点配置文件

```linux
[flash7k@hadoop102 hadoop]$ xsync /opt/module/hadoop-3.1.3/etc
```

2. **启动集群**

   1. **如果是第一次启动**，需要在hadoop102节点格式化NameNode

      > **注意：格式化NameNode，会产生新的集群id，导致NameNode和DataNode的集群id不一致，集群找不到已往数据。如果集群在运行过程中报错，需要重新格式化NameNode的话，一定要先停止namenode和datanode进程，并且要删除所有机器的data和logs目录，然后再进行格式化**

   ```linux
   [flash7k@hadoop102 hadoop-3.1.3]$ hdfs namenode -format
   ```

   2. **启动HDFS**

   ```linux
   [flash7k@hadoop102 hadoop-3.1.3]$ sbin/start-dfs.sh
   ```

   3. **启动YARN：在配置了ResourceManager的节点**

   ```linux
   [flash7k@hadoop103 hadoop-3.1.3]$ sbin/start-yarn.sh
   ```

   4. **Web端查看HDFS的NameNode**
      + 浏览器中输入：http://hadoop102:9870
      + 查看HDFS上存储的数据信息
   5. **Web端查看YARN的ResourceManager**
      + 浏览器中输入：http://hadoop103:8088
      + 查看YARN上运行的Job信息

3. **集群基本测试**

   1. **上传文件到集群**

      + 上传小文件

      ```linux
      [flash7k@hadoop102 ~]$ hadoop fs -mkdir /input
      [flash7k@hadoop102 ~]$ hadoop fs -put $HADOOP_HOME/wcinput/word.txt /input
      ```

      + 上传大文件

      ```linux
      [flash7k@hadoop102 ~]$ hadoop fs -put  /opt/software/jdk-8u212-linux-x64.tar.gz  /
      ```

   2. **查看上传文件存放在什么位置**

      + 查看HDFS文件存储路径

      ```linux
      [flash7k@hadoop102 subdir0]$ pwd
      /opt/module/hadoop-3.1.3/data/dfs/data/current/BP-1436128598-192.168.10.102-1610603650062/current/finalized/subdir0/subdir0
      ```

      + 查看HDFS在磁盘存储文件内容

      ```linux
      [flash7k@hadoop102 subdir0]$ cat blk_1073741825
      ```

   3. **拼接**

      **注意：由于一个block块存储128Mb，jdk大于128M所以被分成两块**

      ```linux
      [flash7k@hadoop102 subdir0]$ cat blk_1073741836>>tmp.tar.gz
      [flash7k@hadoop102 subdir0]$ cat blk_1073741837>>tmp.tar.gz
      [flash7k@hadoop102 subdir0]$ tar -zxvf tmp.tar.gz
      ```

   4. **下载**

   ```linux
   [flash7k@hadoop104 software]$ hadoop fs -get /jdk-8u212-linux-x64.tar.gz ./
   ```

   5. **执行wordcount程序**

   ```linux
   [flash7k@hadoop102 hadoop-3.1.3]$ hadoop jar share/hadoop/mapreduce/hadoop-mapreduce-examples-3.1.3.jar wordcount /input /output
   ```

#### 3.6 配置历史服务器

**作用：查看程序的历史运行情况**

1. **配置mapred-site.xml**

```linux
[flash7k@hadoop102 hadoop]$ vim mapred-site.xml


#增加如下配置
<!-- 历史服务器端地址 -->
<property>
    <name>mapreduce.jobhistory.address</name>
    <value>hadoop102:10020</value>
</property>

<!-- 历史服务器web端地址 -->
<property>
    <name>mapreduce.jobhistory.webapp.address</name>
    <value>hadoop102:19888</value>
</property>
```

2. **分发配置**

```linux
[flash7k@hadoop102 hadoop]$ xsync $HADOOP_HOME/etc/hadoop/mapred-site.xml
```

3. **在hadoop102启动历史服务器**

```linux
[flash7k@hadoop102 hadoop]$ mapred --daemon start historyserver
```

4. **查看历史服务器是否启动**

```linux
[flash7k@hadoop102 hadoop]$ jps
```

5. **查看JobHistory**

   http://hadoop102:19888/jobhistory

#### 3.7 配置日志的聚集

+ **概念**：应用运行完成以后，将程序运行日志信息上传到HDFS系统上
+ **好处**：可以方便的查看到程序运行详情，方便开发调试

1. **配置yarn-site.xml**

```linux
[flash7k@hadoop102 hadoop]$ vim yarn-site.xml


#增加如下配置
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
```

2. **分发配置**

```linux
[flash7k@hadoop102 hadoop]$ xsync $HADOOP_HOME/etc/hadoop/yarn-site.xml
```

#### 3.8 集群启动/停止方式总结

1. **各个模块分开启动/停止**

   1. 整体启动/停止HDFS

   ```linux
   start-dfs.sh/stop-dfs.sh
   ```

   2. 整体启动/停止HDFS

   ```linux
   start-yarn.sh/stop-yarn.sh
   ```

2. **各个服务组件逐一启动/停止**

   1. 分别启动/停止HDFS组件

   ```linux
   hdfs --daemon start/stop namenode/datanode/secondarynamenode
   ```

   2. 启动/停止YARN

   ```linux
   yarn --daemon start/stop  resourcemanager/nodemanager
   ```

#### 3.9 编写Hadoop集群常用脚本

1. **Hadoop集群启停脚本（包含HDFS，Yarn，Historyserver）：myhadoop.sh**

```linux
[flash7k@hadoop102 ~]$ cd /home/flash7k/bin
[flash7k@hadoop102 bin]$ vim myhadoop.sh


#输入以下内容

#!/bin/bash

if [ $# -lt 1 ]
then
    echo "No Args Input..."
    exit ;
fi

case $1 in
"start")
        echo " =================== 启动 hadoop集群 ==================="

        echo " --------------- 启动 hdfs ---------------"
        ssh Clou101 "/opt/module/hadoop-3.1.3/sbin/start-dfs.sh"
        echo " --------------- 启动 yarn ---------------"
        ssh Clou102 "/opt/module/hadoop-3.1.3/sbin/start-yarn.sh"
        echo " --------------- 启动 historyserver ---------------"
        ssh Clou101 "/opt/module/hadoop-3.1.3/bin/mapred --daemon start historyserver"
;;
"stop")
        echo " =================== 关闭 hadoop集群 ==================="

        echo " --------------- 关闭 historyserver ---------------"
        ssh Clou101 "/opt/module/hadoop-3.1.3/bin/mapred --daemon stop historyserver"
        echo " --------------- 关闭 yarn ---------------"
        ssh Clou102 "/opt/module/hadoop-3.1.3/sbin/stop-yarn.sh"
        echo " --------------- 关闭 hdfs ---------------"
        ssh Clou101 "/opt/module/hadoop-3.1.3/sbin/stop-dfs.sh"
;;
*)
    echo "Input Args Error..."
;;
esac
```

+ 保存后退出，然后赋予脚本执行权限

```linux
[flash7k@hadoop102 bin]$ chmod +x myhadoop.sh
```

2. **查看三台服务器Java进程脚本：jpsall**

```linux
[flash7k@hadoop102 ~]$ cd /home/atguigu/bin
[flash7k@hadoop102 bin]$ vim jpsall


#输入以下内容

#!/bin/bash

for host in Clou101 Clou102 Clou103
do
        echo =============== $host ===============
        ssh $host jps 
done
```

+ 保存后退出，然后赋予脚本执行权限

```linux
[flash7k@hadoop102 bin]$ chmod +x jpsall
```

3. **分发配置**

```linux
[flash7k@hadoop102 ~]$ xsync /home/flash7k/bin/
```

#### 3.10 常用端口号说明

端口名称|Hadoop2.x|Hadoop3.x
:-|:---|:--
NameNode内部通信端口|8020/9000|8020/9000/9820
NameNode HTTP UI|50070|9870
MapReduce查看执行任务端口|8088|8088
历史服务器通信端口|19888|19888

