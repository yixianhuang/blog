## 整体集群规划

### 软件版本


| 软件 | 版本 |
| --- | --- |
| jdk | 1.8.0_221 |
| hadoop | 2.9.2 |
### 集群规划
|节点名称 | node01 | node02| node03 |
| --- | --- | --- | --- |
|IP | 192.168.33.11 | 192.168.33.12 |192.168.33.13  |
|HDFS | NameNode DataNode | DataNode |SecondaryNameNode DataNode  |
|YARN | NodeManager |  ResourceManager NodeManager| NodeManager |


## 环境准备
### 配置3个节点的主机名：
```bash
hostnamectl set-hostname node01
hostnamectl set-hostname node02
hostnamectl set-hostname node03
```

### 每台node配置host：
```bash
cat <<EOF >> /etc/hosts
192.168.33.11 node01
192.168.33.12 node02
192.168.33.13 node03
EOF
```

### ssh 3台node互相免密码登录
```bash
## 3台node上执行
ssh-keygen -t rsa
ssh-copy-id node01
ssh-copy-id node02
ssh-copy-id node03
```

### 集群时间同步
```bash
## 所有node上安装ntp
yum install ntp

## 选择一台node 作为时间同步的机器
## 在node01 上进行配置
#server 0.centos.pool.ntp.org iburst
#server 1.centos.pool.ntp.org iburst
#server 2.centos.pool.ntp.org iburst
#server 3.centos.pool.ntp.org iburst
server 127.127.1.0
fudge 127.127.1.0 stratum 10

## 查看状态
service ntpd status
## 检查是否启动
service ntpd start
## 配置自动运行
chkconfig ntpd on

## 在ndoe02 node03 以root用户进行时间同步配置
crontab -e
## 每10分钟 进行时间同步
*/2 * * * * /usr/sbin/ntpdate node01

## 查看定时任务
crontab -l

## 检验时间同步是否可行
## 修改日期
date -s "2018-8-8 11:11:11"
## 查看时间是否有变化
date
```

### 安装java： 
```bash
#创建java目录：
mkdir /usr/local/java/

#解压：
tar -zxvf jdk1.8.0_221-linux-x64.tar.gz -C /usr/local/java/

#添加环境变量：
vi  /etc/profile
export JAVA_HOME=/usr/local/java/jdk1.8.0_221
export JRE_HOME=${JAVA_HOME}/jre
export CLASSPATH=.:${JAVA_HOME}/lib:${JRE_HOME}/lib
export PATH=${JAVA_HOME}/bin:$PATH

#使环境变量生效
source /etc/profile

#检查
java -version
```

### 安装Hadoop：
```bash
#创建hadoop目录：
mkdir /usr/local/hadoop/

#解压：
tar -zxvf hadoop-2.9.2.tar.gz -C /usr/local/hadoop/

#添加环境变量：
vi  /etc/profile
export HADOOP_HOME=/usr/local/hadoop/hadoop-2.9.2
export PATH=${HADOOP_HOME}/bin:$PATH
export PATH=${HADOOP_HOME}/sbin:$PATH

#使环境变量生效
source /etc/profile

#检查
hadoop version
```
## Hadoop 配置(3 node进行相同配置)
```xml
vi etc/hadoop/core-site.xml
<!-- 指定HDFS中NameNode的地址 -->
<property>
    <name>fs.defaultFS</name>
    <value>hdfs://node01:9000</value>
</property>

<!-- 指定hadoop运行时产生文件的存储目录
此值会影响 etc/hadoop/hdfs-site.xml 中的
dfs.namenode.name.dir
dfs.datanode.data.dir
-->
<property>
    <name>hadoop.tmp.dir</name>
    <value>/usr/local/hadoop/hadoop-2.9.2/data/tmp</value>
</property>
```

```xml
vi etc/hadoop/hdfs-site.xml
<!-- 复本数量-->
<property>
    <name>dfs.replication</name>
    <value>3</value>
</property>

<!-- secondaryNamenode 地址-->
<property>
    <name>dfs.namenode.secondary.http-address</name>
    <value>node03:50090</value>
</property>
```

```xml
vi etc/hadoop/yarn-site.xml
<!-- reducer获取数据的方式 -->
<property>
    <name>yarn.nodemanager.aux-services</name>
    <value>mapreduce_shuffle</value>
</property>

<!-- 指定YARN的ResourceManager的地址 -->
<property>
    <name>yarn.resourcemanager.hostname</name>
    <value>node02</value>
</property>

<!-- 日志聚集功能使能 -->
<property>
    <name>yarn.log-aggregation-enable</name>
    <value>true</value>
</property>

<!-- 日志保留时间设置7天 -->
<property>
    <name>yarn.log-aggregation.retain-seconds</name>
    <value>604800</value>
</property>

```


```xml
sudo cp mapred-site.xml.template mapred-site.xml
vi etc/hadoop/mapred-site.xml

<!-- 指定mr运行在yarn上 -->
<property>
    <name>mapreduce.framework.name</name>
    <value>yarn</value>
</property>
```

```bash
vi etc/hadoop/slaves

node01
node02
node03
```

```bash
## 修改环境文件 添加java home
vi hadoop-env.sh
export JAVA_HOME=/usr/local/java/jdk1.8.0_221

vi yarn-env.sh
export JAVA_HOME=/usr/local/java/jdk1.8.0_221

vi mapred-env.sh
export JAVA_HOME=/usr/local/java/jdk1.8.0_221
```



## 集群启动
```bash
## 如果集群是第一次启动，需要格式化NameNode
## 在 node01 初始化Namenode
hadoop namenode -format

## 在 node01 启动HDFS
sbin/start-dfs.sh

## 在 node02 启动YARN
sbin/start-yarn.sh

## 在 node02 启动jobhistory
 sbin/mr-jobhistory-daemon.sh start historyserver

## 在每个节点上检查是否已经启动
jps

## 浏览器检查集群
## HDFS
http://192.168.33.11:50070/dfshealth.html#tab-overview
## YARN
http://192.168.33.12:8088/cluster
## jobhistory
http://192.168.33.12:19888/jobhistory
## 查看 SecondaryNameNode 状态
http://192.168.33.13:50090/status.html
```

## 集群测试
```bash
## 创建目录
hadoop fs -mkdir -p /user/vagrant/wordcount/input/

## 上次文件到 HDFS
hadoop fs -put README.txt /user/vagrant/wordcount/input

## 在web上可以查看到此文件的信息：
#Block ID: 1073741825
#Block Pool ID: BP-1137935038-192.168.33.11-1567258714316
#Generation Stamp: 1001
#Size: 1366
#Availability:
#node03
#node02
#node01
## 在本地文件系统中可以查找到对应的文件
/usr/local/hadoop/hadoop-2.9.2/data/tmp/dfs/data/current/BP-1137935038-192.168.33.11-1567258714316/current/finalized/subdir0/subdir0
## 查看里面的内容与 上传的 README.txt 是一样的。1073741825对应 Block ID
cat blk_1073741825

## 运行 demo程序看出结果
hadoop jar share/hadoop/mapreduce/hadoop-mapreduce-examples-2.9.2.jar wordcount /user/vagrant/wordcount/input  /user/vagrant/wordcount/output

##查看结果
bin/hdfs dfs -cat /user/vagrant/wordcount/output/*
```
