---
layout: post
title: 部署伪分布式HBase
categories: HBase
description: 部署伪分布式HBase。
keywords: HBase, Hadoop
---

## 部署伪分布式HBase

花了很久在公司的linux系统上部署伪分布式hbase，总是会遇到各种各样的bug。系统是64位的ubuntu 16.04 LTS，可能之前装的版本比较低导致各种bug，目前使用的是hadoop-2.6.0，hbase-0.98.8，JDK 1.6.0_35，主要参考的是[HBase - Installation](https://www.tutorialspoint.com/hbase/hbase_installation.htm)，但还需要自己改改东西。

我的安装目录一般在/opt/soft/，且设为一般用户权限而非root权限，后面假设所有安装的软件都安装在这个目录。

### 1. 安装JDK 1.6

  略。

### 2. 安装Hadoop-2.6.0
编译过程比较麻烦，我之前直接下过各种版本的hadoop，但都遇到各种问题，于是只能自己编译，可以参考[编译本地64位版本的hadoop-2.6.0](http://www.cnblogs.com/hanganglin/p/4349919.html)，具体过程如下：

#### 1. 下载并编译安装[Protobuf](https://github.com/google/protobuf)
必须要2.5.0版。下载解压后的命令分别运行```./configure --prefix=/opt/soft/protoc``` 和 ```make && make install```，然后把/opt/soft/protoc/bin合并到环境变量的path里（修改~/.bashrc，后略）。

#### 2. 下载并编译安装[findbugs-2.0.3-source.zip](https://sourceforge.net/projects/findbugs/files/findbugs/2.0.3/)
这里用2.0.3而不是3.0.0是因为后者依赖JDK1.7。解压，进入目录后运行```ant build```，再把这个路径命名为FINDBUGS_HOME并加到环境变量中。

#### 3. 下载[hadoop-2.6.0-src.tar.gz](https://archive.apache.org/dist/hadoop/core/hadoop-2.6.0/hadoop-2.6.0-src.tar.gz)
并解压到某个位置，进去以后修改hadoop-common-project/hadoop-auth/pom.xml，在第55行左右添加以下内容：
```xml
<dependency>
  <groupId>org.mortbay.jetty</groupId>
  <artifactId>jetty-util</artifactId>
  <scope>test</scope>
</dependency>
```

然后进入到源代码所在目录，运行```mvn package -DskipTests -Pdist,native,docs```，编译后的项目在hadoop-2.6.0-src/hadoop-dist/target/hadoop-2.6.0中，将这个目录移动到/opt/soft/。

#### 4. 下载[hbase-0.98.8-hadoop2-bin.tar.gz](https://archive.apache.org/dist/hbase/hbase-0.98.8/hbase-0.98.8-hadoop2-bin.tar.gz)
并解压到/opt/soft/。

#### 5. 修改环境变量
即修改~/.bashrc，我改完大概是这样：
```bash
export JAVA_HOME=/opt/soft/jdk1.6.0_35

export HADOOP_HOME=/opt/soft/hadoop-2.6.0
export HADOOP_MAPRED_HOME=$HADOOP_HOME
export HADOOP_COMMON_HOME=$HADOOP_HOME
export HADOOP_HDFS_HOME=$HADOOP_HOME
export YARN_HOME=$HADOOP_HOME
export HADOOP_COMMON_LIB_NATIVE_DIR=$HADOOP_HOME/lib/native
export HADOOP_INSTALL=$HADOOP_HOME

export HBASE_HOME=/opt/soft/hbase-0.98.8-hadoop2

export HADOOP_OPTS="-Djava.library.path=$HADOOP_HOME/lib/native"

export PATH=$PATH:$JAVA_HOME/bin
export CLASSPATH=$CLASSPATH:$JAVA_HOME/lib
```

#### 6. 配置并运行hadoop
配置文件在$HADOOP_HOME/etc/hadoop这个目录下，其中core-site.xml为
```xml
<configuration>
  <property>
    <name>fs.default.name</name>
    <value>hdfs://localhost:9000</value>
  </property>
</configuration>
```
hdfs-site.xml为
```xml
<configuration>
  <property>
    <name>dfs.replication</name >
    <value>1</value>
  </property>
  
  <property>
    <name>dfs.name.dir</name>
    <value>file:///home/your_user_name/hadoopinfra/hdfs/namenode</value>
  </property>
  
  <property>
    <name>dfs.data.dir</name>
    <value>file:///home/your_user_name/hadoopinfra/hdfs/datanode</value>
  </property>
</configuration>
```
mapred-site.xml为
```xml
<configuration>
  <property>
    <name>mapreduce.framework.name</name>
    <value>yarn</value>
  </property>
</configuration>
```
yarn-site.xml为
```xml
<configuration>
  <property>
    <name>yarn.nodemanager.aux-services</name>
    <value>mapreduce_shuffle</value>
  </property>
</configuration>
```
然后运行```$HADOOP_HOME/bin/hdfs namenode -format```初始化namenode。

再分别运行```$HADOOP_HOME/sbin/start-dfs.sh```和```$HADOOP_HOME/sbin/start-yarn.sh```来启动hadoop，启动完毕后可以去http://localhost:50070 看看有没有正常跑起来。

#### 7. 配置并运行hbase
修改$HBASE_HOME/conf/hbase-env.sh，添加以下内容：
```bash
export JAVA_HOME=/opt/soft/jdk1.6.0_35
export HBASE_OPTS="-XX:+UseConcMarkSweepGC"
export HBASE_HOME=/opt/soft/hbase-0.98.8-hadoop2
export HBASE_CLASSPATH=$HBASE_HOME/lib
```
修改$HBASE_HOME/conf/hbase-site.xml，添加以下内容：
```xml
<configuration>
  <property>
    <name>hbase.rootdir</name>
    <value>hdfs://localhost:9000/hbase</value>
  </property>
  <property>
    <name>hbase.zookeeper.property.dataDir</name>
    <value>/home/your_user_name/zookeeper</value>
  </property>
  <property>
    <name>hbase.cluster.distributed</name>
    <value>true</value>
  </property>
</configuration>
```
值得注意的是，hbase.rootdir的值必须要和$HADOOP_HOME/etc/core-site.xml里的fs.default.name值一致。

然后启动hbase：```$HBASE_HOME/bin/start-hbase.sh```，如果能正确运行$HBASE_HOME/bin/hbase shell - list，应该就算正确安装了。

如果出现什么slf4j版本重复之类的warning，删掉$HBASE_HOME/lib/slf4j-log4j12-1.6.4.jar。