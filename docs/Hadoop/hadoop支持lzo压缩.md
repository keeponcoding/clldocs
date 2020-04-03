# 1.环境准备

* jdk1.8/maven3.6.2/hadoop伪分布式
* 检查当前支持的压缩

```shell
# 不存在lzo压缩
[hadoop@hadoop000 ~]$ hadoop checknative
19/09/30 05:00:45 INFO bzip2.Bzip2Factory: Successfully loaded & initialized native-bzip2 library system-native
19/09/30 05:00:45 INFO zlib.ZlibFactory: Successfully loaded & initialized native-zlib library
Native library checking:
hadoop:  true /home/hadoop/app/hadoop-2.6.0-cdh5.15.1/lib/native/libhadoop.so.1.0.0
zlib:    true /lib64/libz.so.1
snappy:  true /lib64/libsnappy.so.1
lz4:     true revision:10301
bzip2:   true /lib64/libbz2.so.1
openssl: true /lib64/libcrypto.so
```

* 安装需要的库

```shell
yum -y install lzo-devel zlib-devel gcc autoconf automake libtool
```

# 2.安装 lzo

## 2.1.下载并解压lzo

```shell
# 下载lzo压缩包
[hadoop@hadoop000 software]$ wget www.oberhumer.com/opensource/lzo/download/lzo-2.06.tar.gz

# 解压缩
[hadoop@hadoop000 software]$ tar -zxvf lzo-2.06.tar.gz -C ~/app/
```

## 2.2.编译并安装

```shell
[hadoop@hadoop000 lzo-2.06]$ pwd
/home/hadoop/app/lzo-2.06

[hadoop@hadoop000 lzo-2.06]$ export CFLAGS=-m64

# 创建存放编译后的目录
[hadoop@hadoop000 lzo-2.06]$ mkdir compile

# 编译安装
[hadoop@hadoop000 lzo-2.06]$ ./configure -enable-shared -prefix=/home/hadoop/app/lzo-2.06/complie/
[hadoop@hadoop000 lzo-2.06]$ make && make install

# 查看编译结果
[hadoop@hadoop000 compile]$ ll
总用量 0
drwxrwxr-x. 3 hadoop hadoop  17 9月  30 05:31 include
drwxrwxr-x. 2 hadoop hadoop 103 9月  30 05:31 lib
drwxrwxr-x. 3 hadoop hadoop  17 9月  30 05:31 share
```

# 3.安装hadoop-lzo

## 3.1.下载并解压

```shell
# 下载
[hadoop@hadoop000 software]$ wget https://github.com/twitter/hadoop-lzo/archive/master.zip

#解压   解压出来的是 hadoop-lzo-master
[hadoop@hadoop000 software]$ unzip master

[hadoop@hadoop000 software]$ mv hadoop-lzo-master ~/app/
```

## 3.2.修改pom.xml并增加配置

```shell
# 修改pom.xml
[hadoop@hadoop000 hadoop-lzo-master]$ vi pom.xml
<properties>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <!--该处修改成对应的hadoop版本即可-->
    <hadoop.current.version>2.6.0</hadoop.current.version>
    <hadoop.old.version>1.0.4</hadoop.old.version>
  </properties>
  
# 增加配置
[hadoop@hadoop000 hadoop-lzo-master]$ export CFLAGS=-m64
[hadoop@hadoop000 hadoop-lzo-master]$ export CXXFLAGS=-m64
[hadoop@hadoop000 hadoop-lzo-master]$ export C_INCLUDE_PATH=/home/hadoop/app/lzo-2.06/compile/include/
[hadoop@hadoop000 hadoop-lzo-master]$ export LIBRARY_PATH=/home/hadoop/app/lzo-2.06/compile/lib/
```

## 3.3.编译

```shell
[hadoop@hadoop000 hadoop-lzo-master]$ mvn clean package -Dmaven.test.skip=true
......
BUILD SUCCESS
......
# 等待编译至成功即可

[hadoop@hadoop000 Linux-amd64-64]$ pwd
/home/hadoop/app/hadoop-lzo-master/target/native/Linux-amd64-64
[hadoop@hadoop000 Linux-amd64-64]$ tar -cBf - -C lib . | tar -xBvf - -C ~
./
./libgplcompression.so.0.0.0
./libgplcompression.so.0
./libgplcompression.so
./libgplcompression.la
./libgplcompression.a
[hadoop@hadoop000 Linux-amd64-64]$ cp ~/libgplcompression* $HADOOP_HOME/lib/native/

# 拷贝jar包
[hadoop@hadoop000 target]$ pwd
/home/hadoop/app/hadoop-lzo-master/target
[hadoop@hadoop000 target]$ cp hadoop-lzo-0.4.21-SNAPSHOT.jar $HADOOP_HOME/share/hadoop/common/
[hadoop@hadoop000 target]$ cp hadoop-lzo-0.4.21-SNAPSHOT.jar $HADOOP_HOME/share/hadoop/mapreduce/lib/
```

# 4.修改hadoop配置并重启

## 4.1.修改配置文件

```shell
[hadoop@hadoop000 hadoop]$ pwd
/home/hadoop/app/hadoop/etc/hadoop

# hadoop-env.sh
[hadoop@hadoop000 hadoop]$ vi hadoop-env.sh 
# 增加编译好的lzo下的lib
export LD_LIBRARY_PATH=/home/hadoop/app/lzo-2.06/compile/lib

# core-site.xml 
[hadoop@hadoop000 hadoop]$ vi core-site.xml 
<!--压缩-->
<property>
    <name>io.compression.codecs</name>
    <value>org.apache.hadoop.io.compress.GzipCodec,
            org.apache.hadoop.io.compress.DefaultCodec,
            org.apache.hadoop.io.compress.BZip2Codec,
            com.hadoop.compression.lzo.LzoCodec,
            com.hadoop.compression.lzo.LzopCodec
    </value>
</property>
<property>
    <name>io.compression.codec.lzo.class</name>
    <value>com.hadoop.compression.lzo.LzoCodec</value>
</property>

# mapred-site.xml
[hadoop@hadoop000 hadoop]$ vi mapred-site.xml
<!--lzo压缩   start-->
<property>
    <name>mapred.child.env </name>
    <value>LD_LIBRARY_PATH=/home/hadoop/app/lzo-2.06/compile/lib</value>
</property>
<property>
    <name>mapreduce.map.output.compress</name>
    <value>true</value>
</property>
<property>
    <name>mapreduce.map.output.compress.codec</name>
    <value>com.hadoop.compression.lzo.LzoCodec</value>
</property>
<!--end-->
```

## 4.2.重启hadoop

```shell
[hadoop@hadoop000 data]$ stop-dfs.sh
[hadoop@hadoop000 data]$ start-dfs.sh
```



测试

压缩测试数据

```shell
# 压缩前  root用户下安装 lzop
yum install lzop

[hadoop@hadoop000 data]$ split -b 350m access.log

[hadoop@hadoop000 data]$ lzop access_testlzo.log

[hadoop@hadoop000 data]$ ll -lh
总用量 740M
-rw-rw-r--. 1 hadoop hadoop 565M 9月  30 12:17 access.log
-rw-rw-r--. 1 hadoop hadoop   72 9月  18 11:03 access_log.txt
-rw-rw-r--. 1 hadoop hadoop 160M 9月  30 13:11 access_testlzo.log.lzo
```





```shell
[hadoop@hadoop000 mapreduce]$ pwd
/home/hadoop/app/hadoop/share/hadoop/mapreduce

# 运行wc 查看日志发现 hadoop并没有给lzo切片
[hadoop@hadoop000 mapreduce]$ hadoop jar \
> hadoop-mapreduce-examples-2.6.0-cdh5.15.1.jar wordcount \
> /data/access_testlzo.log.lzo \
> /out
19/09/30 13:25:16 INFO client.RMProxy: Connecting to ResourceManager at /0.0.0.0:8032
19/09/30 13:25:18 INFO input.FileInputFormat: Total input paths to process : 1
19/09/30 13:25:18 INFO lzo.GPLNativeCodeLoader: Loaded native gpl library from the embedded binaries
19/09/30 13:25:18 INFO lzo.LzoCodec: Successfully loaded & initialized native-lzo library [hadoop-lzo rev 5dbdddb8cfb544e58b4e0b9664b9d1b66657faf5]
19/09/30 13:25:18 INFO mapreduce.JobSubmitter: number of splits:1
19/09/30 13:25:18 INFO mapreduce.JobSubmitter: Submitting tokens for job: job_1569814006008_0001
19/09/30 13:25:19 INFO impl.YarnClientImpl: Submitted application application_1569814006008_0001
19/09/30 13:25:19 INFO mapreduce.Job: The url to track the job: http://hadoop000:8088/proxy/application_1569814006008_0001/
```

加索引并且设置inputformat

```shell
[hadoop@hadoop000 lib]$ pwd
/home/hadoop/app/hadoop/share/hadoop/mapreduce/lib

[hadoop@hadoop000 lib]$ hadoop jar \
> hadoop-lzo-0.4.21-SNAPSHOT.jar \
> com.hadoop.compression.lzo.DistributedLzoIndexer \
> /data/access_testlzo.log.lzo

[hadoop@hadoop000 lib]$ hdfs dfs -ls /data
Found 3 items
-rw-r--r--   1 hadoop supergroup  167664788 2019-09-30 13:17 /data/access_testlzo.log.lzo
-rw-r--r--   1 hadoop supergroup      11200 2019-09-30 13:31 /data/access_testlzo.log.lzo.index


[hadoop@hadoop000 mapreduce]$ hadoop jar \
> hadoop-mapreduce-examples-2.6.0-cdh5.15.1.jar wordcount \
> -Dmapreduce.job.inputformat.class=com.hadoop.mapreduce.LzoTextInputFormat \
> /data/access_testlzo.log.lzo \
> /out
19/09/30 13:34:51 INFO client.RMProxy: Connecting to ResourceManager at /0.0.0.0:8032
19/09/30 13:34:52 INFO input.FileInputFormat: Total input paths to process : 1
19/09/30 13:34:52 INFO mapreduce.JobSubmitter: number of splits:2
19/09/30 13:34:53 INFO mapreduce.JobSubmitter: Submitting tokens for job: job_1569814006008_0003
19/09/30 13:34:53 INFO impl.YarnClientImpl: Submitted application application_1569814006008_0003
19/09/30 13:34:53 INFO mapreduce.Job: The url to track the job: http://hadoop000:8088/proxy/application_1569814006008_0003/
```

























