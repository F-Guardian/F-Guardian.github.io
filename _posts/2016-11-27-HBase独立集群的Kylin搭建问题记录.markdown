---
title:  "HBase独立集群的Kylin搭建问题记录"
date:   2016-11-27 00:17:18
categories: [Kylin]
tags: [Kylin,HBase]
---

## 1. 前言
前两天在公司现有集群上搭建一个`Kylin`服务，由于`HBase`是一个独立集群，所以搭建的过程中遇到了各种问题，这里把过程中遇到的问题记录一下，对Kylin本身和基本配置不做介绍，想了解Kylin介绍的可以去[Kylin官网][url1]。相关组件版本说明如下：

>JDK：1.7.0_67  
>HBase：1.0.1.1  
>Kylin：1.5.1  
>Hive：1.2.1  
>Hadoop：2.6.0  

<br/>

## 2. 额外配置
首先由于`HBase`是独立集群的原因，所以本身需要一些额外的配置，根据[官网的说法][url2]，有以下2点需要做额外配置。

### 2.1 HBase独立集群的HDFS指定
将`kylin.properties`中的` kylin.hbase.cluster.fs`设置为`hdfs://hbase-cluster-nn01.example.com:8020`，如果HBase集群中设置了NN-HA相关的配置则可以将这个值设为`hdfs://hbase-cluster`

### 2.2 合并2个集群中与NN-HA有关的配置
需要将2个集群中与NN-HA有关的配置写入`kylin_job_conf.xml`，例子如下：

```xml
   <property>
        <name>dfs.nameservices</name>
        <value>cluster1,cluster2</value>
    </property>
    <property>
        <name>dfs.ha.namenodes.cluster1</name>
        <value>nn1,nn2</value>
    </property>
    <property>
        <name>dfs.namenode.rpc-address.cluster1.nn1</name>
        <value>cluster1-server1:8020</value>
    </property>
    <property>
        <name>dfs.namenode.http-address.cluster1.nn1</name>
        <value>cluster1-server1:50070</value>
    </property>
    <property>
        <name>dfs.namenode.servicerpc-address.cluster1.nn1</name>
        <value>cluster1-server1:54321</value>
    </property>
    <property>
        <name>dfs.namenode.rpc-address.cluster1.nn2</name>
        <value>cluster1-server2:8020</value>
    </property>
    <property>
        <name>dfs.namenode.http-address.cluster1.nn2</name>
        <value>cluster1-server2:50070</value>
    </property>
    <property>
        <name>dfs.namenode.servicerpc-address.cluster1.nn2</name>
        <value>cluster1-server2:54321</value>
    </property>
    <property>
        <name>dfs.client.failover.proxy.provider.cluster1</name>
        <value>org.apache.hadoop.hdfs.server.namenode.ha.ConfiguredFailoverProxyProvider</value>
    </property>
    <property>
        <name>dfs.ha.namenodes.cluster2</name>
        <value>nn1,nn2</value>
    </property>
    <property>
        <name>dfs.namenode.rpc-address.cluster2.nn1</name>
        <value>cluster2-server1:8020</value>
    </property>
    <property>
         <name>dfs.namenode.http-address.cluster2.nn1</name>
         <value>cluster2-server1:50070</value>
    </property>
    <property>
         <name>dfs.namenode.servicerpc-address.cluster2.nn1</name>
         <value>cluster2-server1:54321</value>
    </property>
    <property>
         <name>dfs.namenode.rpc-address.cluster2.nn2</name>
         <value>cluster2-server2:8020</value>
     </property>
     <property>
         <name>dfs.namenode.http-address.cluster2.nn2</name>
         <value>cluster2-server2:50070</value>
     </property>
     <property>
         <name>dfs.namenode.servicerpc-address.cluster2.nn2</name>
         <value>cluster2-server2:54321</value>
     </property>
     <property>
         <name>dfs.client.failover.proxy.provider.cluster2</name>
         <value>org.apache.hadoop.hdfs.server.namenode.ha.ConfiguredFailoverProxyProvider</value>
     </property>
```
<br/>

## 3. MR任务提交到主集群
按照官方的说明配置好后启动`Kylin`构建cube的过程中第二步` Extract Fact Table Distinct `就会有问题，你会发现Kylin默认将MR任务提交到了独立的HBase集群上，这显然不是我们预期的，预想的流程应该是从Hive中读取数据后再主集群中完成各种MR任务最后写入HBase集群。所以在网上一番搜寻后发现问题出在Kylin的`kylin.sh`中，在这个shell文件中可以看到Kylin所有任务的提交都是通过`hbase`这个shell脚本来完成的，所以RM任务提交时默认是读取的`HBase`的classpath中`Hadoop`配置文件目录，所以需要在Kylin使用hbase脚本启动MR之前在HBase的classpath前加上主集群的Hadoop配置文件目录。这里我图省事没有在kylin.sh中取找地方添加，而是直接在hbase这个shell文件中添加了如下脚本：

```bash
if [ "$HBASE_CLASSPATH_PREFIX" != "" ]; then
  HBASE_CLASSPATH_PREFIX=${YOUR_HADOOP_HOME}/etc/hadoop:${HBASE_CLASSPATH_PREFIX}
else
  HBASE_CLASSPATH_PREFIX=${YOUR_HADOOP_HOME}/etc/hadoop
fi
```
<br/>

## 4. 连接不上Resourcemanager
如果`kylin.log`中报如下错误则说明连接不上集群的Resourcemanager

>yarn.resourcemanager.webapp.address:http://0.0.0.0:8088 and java.net.ConnectException: Connection refused

解决方案为在`kylin.properties`中添加如下内容

```xml
kylin.job.yarn.app.rest.check.status.url=http://YOUR_RM_NODE:8088/ws/v1/cluster/apps/${job_id}?anonymous=true
```
<br/>

## 5. tmp目录权限问题
`Kylin`默认使用`hadoop.tmp.dir`目录来存储临时文件，可以在`kylin_job_conf.xml`中将其值设置为`YOUR_PATH`并确认2个集群中都存在此目录，且启动Kylin的用户对其有写权限。

<br/>

## 6. HBase命名空间问题
目前Kylin还不支持配置HBase的命名空间，所以默认在HBase
中的default下创建表，这个问题目前只能等Kylin更新或者手动修改源码重新编译来解决。顺带一提，`kylin.metadata.url`是支持命名空间的，可以设置为带命名空间的地址。
<br/>
<br/>
<br/>
<br/>
<br/>

#### 其他参考资料

[Apache Kylin - hive and hbase use different hdfs](http://apache-kylin.74782.x6.nabble.com/hive-and-hbase-use-different-hdfs-td1922.html)  
[Apache Kylin | FAQ](http://kylin.apache.org/docs15/gettingstarted/faq.html)  
[Leverage HBase Namespace to isolate Kylin HTables](https://issues.apache.org/jira/browse/KYLIN-224)  

[url1]:http://kylin.apache.org/
[url2]:http://kylin.apache.org/blog/2016/06/10/standalone-hbase-cluster/