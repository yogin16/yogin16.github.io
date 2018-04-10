---
layout:     post
title:      "Giraph Hadoop Setup: Big Graph Analysis with Apache Giraph"
date:       2018-04-10 01:02:58 +0530
comments:   true
---

We wanted to use [http://giraph.apache.org/](http://giraph.apache.org/) for big graph analysis. As Giraph depends on Hadoop and HDFS, We need to setup hadoop cluster. and deploy Giraph.

### Introduction
The setup [http://giraph.apache.org/quick_start.html](http://giraph.apache.org/quick_start.html) explains for hadoop version 1. We are going to use version hadoop 2 - which doesnt have map reduce and it is replaced by yarn framework.

### Setup Hadoop
For local hadoop yarn setup I followed (not followed with exact steps - there are some config props changed due to version):

[https://www.alexjf.net/blog/distributed-systems/hadoop-yarn-installation-definitive-guide/](https://www.alexjf.net/blog/distributed-systems/hadoop-yarn-installation-definitive-guide/)

These are the steps for single node cluster setup:

#### Install
```bash
Yogin-Patel:~ yoginpatel$ pwd
/Users/yoginpatel
```
```bash
Yogin-Patel:hadoop yoginpatel$ sudo wget https://archive.apache.org/dist/hadoop/core/hadoop-2.6.5/hadoop-2.6.5.tar.gz

Yogin-Patel:hadoop yoginpatel$ tar xvf hadoop-2.6.5.tar.gz --gzip

Yogin-Patel:hadoop yoginpatel$ mv hadoop-2.6.5 hadoop

Yogin-Patel:hadoop yoginpatel$ cd hadoop

Yogin-Patel:hadoop yoginpatel$ pwd
/Users/yoginpatel/hadoop

Yogin-Patel:hadoop yoginpatel$ ls
LICENSE.txt NOTICE.txt  README.txt  bin         etc         include     lib         libexec     logs        sbin        share
Yogin-Patel:hadoop yoginpatel$
```

Since I added that in my home with my personal user on mac. this is what I added for paths in `~/.bash_profile`:

```bash
export HADOOP_HOME=/Users/yoginpatel/hadoop
PATH=$HADOOP_HOME/bin:$HADOOP_HOME/sbin:$PATH
```

#### Configure
Need to create data directory for hdfs.

In my system's case I created `/mnt1/hadoop3` - also `/mnt1` is not owned by `yoginpatel` local user. so I had to give 777 permission for hadoop3.

```bash
Yogin-Patel:hadoop3 yoginpatel$ cd ~/hadoop/etc/hadoop/
Yogin-Patel:hadoop yoginpatel$ ls
capacity-scheduler.xml     hadoop-env.sh              httpfs-env.sh              kms-env.sh                 mapred-env.sh              ssl-server.xml.example
configuration.xsl          hadoop-metrics.properties  httpfs-log4j.properties    kms-log4j.properties       mapred-queues.xml.template yarn-env.cmd
container-executor.cfg     hadoop-metrics2.properties httpfs-signature.secret    kms-site.xml               mapred-site.xml            yarn-env.sh
core-site.xml              hadoop-policy.xml          httpfs-site.xml            log4j.properties           slaves                     yarn-site.xml
hadoop-env.cmd             hdfs-site.xml              kms-acls.xml               mapred-env.cmd             ssl-client.xml.example
Yogin-Patel:hadoop yoginpatel$
```

##### Configure HDFS:
Need to change file at: `hadoop/etc/hadoop/hdfs-site.xml` (we basically need to configure data directory)

content from local for reference:
```bash
Yogin-Patel:hadoop yoginpatel$ cat hdfs-site.xml
```
```xml
<?xml version="1.0" encoding="UTF-8"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>

<!-- Put site-specific property overrides in this file. -->

<configuration>
	<property>
            <name>dfs.namenode.name.dir</name>
            <value>/mnt1/hadoop3/namenode</value>
    </property>

    <property>
            <name>dfs.datanode.data.dir</name>
            <value>/mnt1/hadoop3/datanode</value>
    </property>

    <property>
            <name>dfs.replication</name>
            <value>1</value>
    </property>
</configuration>
```

##### Configure Yarn:
Need to change file at: `hadoop/etc/hadoop/yarn-site.xml`

content from my local setup: the limits are due to my local machine limitation - we need to allocate more memories in actual setup.

```bash
Yogin-Patel:hadoop yoginpatel$ cat yarn-site.xml
```
```xml
<?xml version="1.0"?>
<configuration>

<!-- Site specific YARN configuration properties -->
	<property>
            <name>yarn.acl.enable</name>
            <value>0</value>
    </property>

    <property>
            <name>yarn.resourcemanager.hostname</name>
            <value>localhost</value>
    </property>

    <property>
            <name>yarn.nodemanager.aux-services</name>
            <value>mapreduce_shuffle</value>
    </property>

	<property>
        <name>yarn.nodemanager.resource.memory-mb</name>
        <value>4096</value>
</property>

<property>
        <name>yarn.scheduler.maximum-allocation-mb</name>
        <value>4096</value>
</property>

<property>
        <name>yarn.scheduler.minimum-allocation-mb</name>
        <value>128</value>
</property>

<property>
        <name>yarn.nodemanager.vmem-check-enabled</name>
        <value>false</value>
</property>
</configuration>
```

#### Start hadoop:
```bash
$ hdfs namenode -format

(below files are part of: $HADOOP_HOME/sbin)
$ start-dfs.sh
$ start-yarn.sh
```

This started hadoop cluster. this was for localhost one node setup. the config would change in multinode setup.
http://localhost:8082 would show the resource manager ui.

============

### Setup Girpah:

For yarn there is no distribution available we need to build from source:

#### Checkout Giraph Source:

```bash
Yogin-Patel:~ yoginpatel$ pwd
/Users/yoginpatel

$ mkdir giraph
$ cd giraph

$ git clone https://github.com/apache/giraph.git

Yogin-Patel:giraph yoginpatel$ ls
CHANGELOG                README                   checkstyle.xml           giraph-accumulo          giraph-debugger          giraph-hbase             license-header.txt
CODE_CONVENTIONS         bin                      conf                     giraph-block-app         giraph-dist              giraph-hcatalog          pom.xml
LICENSE.txt              checkstyle-relaxed-8.xml dev-support              giraph-block-app-8       giraph-examples          giraph-parent.iml        src
NOTICE                   checkstyle-relaxed.xml   findbugs-exclude.xml     giraph-core              giraph-gora              giraph-rexster           target
Yogin-Patel:giraph yoginpatel$
```

#### Build Giraph

There is a bug in source where building for Hadoop Yarn profile the build fails. To fix the build we need to change the `pom.xml` file:

-> remove `STATIC_SASL_SYMBOL` from line no: 1270 of pom.xml ( The sample diff looks like this: https://github.com/yogin16/giraph/commit/acd536a0de748510a849392df18089ece69988c3 )

```bash
$mvn -Phadoop_yarn -Dhadoop.version=2.6.5 clean package -DskipTests
```

On build success there should be a target jar created:
```bash
$ ls giraph/giraph-core/target/giraph-1.3.0-SNAPSHOT-for-hadoop-2.6.5-jar-with-dependencies.jar
```

Add Giraph jars which created from above build to hadoop classpath:

```bash
cp giraph-1.3.0-SNAPSHOT-for-hadoop-2.6.5-jar-with-dependencies.jar ~/hadoop/share/hadoop/yarn/lib/
cp giraph-examples-1.3.0-SNAPSHOT-for-hadoop-2.6.5-jar-with-dependencies.jar ~/hadoop/share/hadoop/yarn/lib/
```

#### Restart hadoop

This works for single node. For clustered setup this might be different steps.

```bash
$ stop-all.sh

$ start-dfs.sh
$ start-yarn.sh
```

### References:
1. [http://giraph.apache.org/quick_start.html](http://giraph.apache.org/quick_start.html)
1. [https://www.alexjf.net/blog/distributed-systems/hadoop-yarn-installation-definitive-guide/](https://www.alexjf.net/blog/distributed-systems/hadoop-yarn-installation-definitive-guide/)
1. [https://hadoop.apache.org/docs/r2.7.1/hadoop-project-dist/hadoop-hdfs/hdfs-default.xml](https://hadoop.apache.org/docs/r2.7.1/hadoop-project-dist/hadoop-hdfs/hdfs-default.xml)
1. [https://hadoop.apache.org/docs/current/hadoop-yarn/hadoop-yarn-common/yarn-default.xml](https://hadoop.apache.org/docs/current/hadoop-yarn/hadoop-yarn-common/yarn-default.xml)
1. [https://stackoverflow.com/questions/48447198/calling-a-giraph-job-from-a-simple-java-program](https://stackoverflow.com/questions/48447198/calling-a-giraph-job-from-a-simple-java-program)
1. [https://stackoverflow.com/questions/48515640/fatal-main-org-apache-hadoop-mapreduce-v2-app-mrappmaster-error-starting-mrap](https://stackoverflow.com/questions/48515640/fatal-main-org-apache-hadoop-mapreduce-v2-app-mrappmaster-error-starting-mrap)
1. [https://www.slideshare.net/rhatr/introduction-into-scalable-graph-analysis-with-apache-giraph-and-spark-graphx](https://www.slideshare.net/rhatr/introduction-into-scalable-graph-analysis-with-apache-giraph-and-spark-graphx)