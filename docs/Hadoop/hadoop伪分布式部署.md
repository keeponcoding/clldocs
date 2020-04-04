# 1.环境准备

* centos7
* 安装JDK1.8+ ：可参考 [Hadoop集群搭建(三台)](https://blog.csdn.net/qq_34651991/article/details/100572349) 中JDK安装步骤 

* 下载hadoop安装包，本文使用版本为 hadoop-2.6.0-cdh.5.15.1

# 2.部署

* 配置无密码认证

```shell
# 按步骤操作即可
1.ssh-keygen  # 三次回车

2.cd ~/.ssh
  cat id_rsa.pub >> authorized_keys
  
3.chmod 600 authorized_keys
```



* 新增 hadoop 用户，创建需要的目录，上传包

```shell
# 新增 hadoop 用户 并切到hadoop根目录
[root@hadoop000 ~]# useradd hadoop
[root@hadoop000 ~]# su - hadoop

# 创建文件夹 app 存放应用的目录  data 存放数据目录  software 存放压缩包  log 存放日志
[hadoop@hadoop000 ~]$ mkdir app data software data log

# 将hadoop压缩包上传至 software 并解压到app目录
[hadoop@hadoop000 software]$ tar -zxvf hadoop-2.6.0-cdh5.15.1.tar.gz -C ~/app/

# 软连接 配置环境变量
[hadoop@hadoop000 app]$ ln -s hadoop-2.6.0-cdh5.15.1 hadoop
[hadoop@hadoop000 app]$ vi ~/.bash_profile
HADOOP_HOME=/home/hadoop/app/hadoop
PATH=${HADOOP_HOME}/bin:${HADOOP_HOME}/sbin:$PATH
# 使环境变量起作用
[hadoop@hadoop000 app]$ source ~/.bash_profile

# 验证
[hadoop@hadoop000 app]$ which hadoop
~/app/hadoop/bin/hadoop
```

* 修改配置

```shell
[hadoop@hadoop000 hadoop]$ pwd
/home/hadoop/app/hadoop/etc/hadoop


# 修改 hadoop-env.sh
[hadoop@hadoop000 hadoop]$ vi hadoop-env.sh
# 修改其中的JAVA_HOME
export JAVA_HOME=/usr/java/jdk1.8.0_40
# 修改dn nn snn pid文件保存目录 默认时报存在 /tmp 下 但是会定时请里 所以需要指向一个新路径
export HADOOP_PID_DIR=/data/tmp


# 修改 core-site.xml
[hadoop@hadoop000 hadoop]$ vi core-site.xml
<!--这里配置的是hostname-->
<property>
    <name>fs.defaultFS</name>
    <value>hdfs://hadoop000:9000</value>
</property>


# 修改 slaves
[hadoop@hadoop000 hadoop]$ vi slaves
hadoop000


# 修改 hdfs-site.xml
[hadoop@hadoop000 hadoop]$ vi hdfs-site.xml
# 备份数
<property>
    <name>dfs.replication</name>
    <value>1</value>
</property>
# 指定 snn 的地址为 hostname:port
<property>
        <name>dfs.namenode.secondary.http-address</name>
        <value>hadoop000:50090</value>
</property>
<property>
        <name>dfs.namenode.secondary.https-address</name>
        <value>hadoop000:50091</value>
</property>


# 新增 mapred-site.xml  复制一份模板配置
[hadoop@hadoop000 hadoop]$ cp mapred-site.xml.template mapred-site.xml
[hadoop@hadoop000 hadoop]$ vi mapred-site.xml
# 添加配置
<property>
        <name>mapreduce.framework.name</name>
        <value>yarn</value>
</property>


# 修改 yarn-site.xml
[hadoop@hadoop000 hadoop]$ vi yarn-site.xml
# 添加配置
<property>
    <name>yarn.nodemanager.aux-services</name>
    <value>mapreduce_shuffle</value>
</property>
```

* 格式化hdfs文件系统

```shell
# 格式化
[hadoop@hadoop000 hadoop]$ hdfs namenode -format

# 启动 全部启动
[hadoop@hadoop000 hadoop]$ start-all.sh
[hadoop@hadoop000 hadoop]$ jps
1060 ResourceManager
80853 Jps
616 NameNode
747 DataNode
1164 NodeManager
911 SecondaryNameNode
```

* web页面访问

HDFS --> http://ip:50070  

YARN --> http://ip:8088/cluster

* 停止服务

```shell
[hadoop@hadoop000 hadoop]$ stop-all.sh
```

