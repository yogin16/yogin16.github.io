---
layout:     post
title:      "Configuring SolrCloud with Solr 6.4.1"
date:       2017-02-28 10:26:58 +0530
comments:   true
---
This article follows configuring SolrCloud on [Solr](http://lucene.apache.org/solr/) 6.4.1 and [ZooKeeper](https://zookeeper.apache.org/) 3.4.6.

In order to configure SolrCloud we need

1. ZooKeeper ensemble setup
2. `solr.xml`: the solr node config file. this file has to be present in `solr.home.home`
3. `schema.xml`: this defines the schema of the core. (needed for every core) Solr uses this from `managed-schema`
4. `solrconfig.xml`: config for core. (needed for every core)

In SolrCloud the config would be uploaded as Zookeeper znode. and all the node starting as could would get the config from there.

This is how you can add one collection having 2 shards and 2 replicas on 3 solr node cluster with a zookeeper ensemble of 3 nodes.

### Setup Zookeeper Ensemble
From the ZK admin [doc](http://zookeeper.apache.org/doc/r3.4.6/zookeeperAdmin.html):
> For the ZooKeeper service to be active, there must be a majority of non-failing machines that can communicate with each other. To create a deployment that can tolerate the failure of F machines, you should count on deploying 2xF+1 machines. Thus, a deployment that consists of three machines can handle one failure, and a deployment of five machines can handle two failures. Note that a deployment of six machines can only handle two failures since three machines is not a majority. For this reason, ZooKeeper deployments are usually made up of an odd number of machines.

##### Install Zookeeper
0. Three servers ready to launch zookeeper.
1. Download latest Zookeeper from [their release page.](http://zookeeper.apache.org/releases.html#download). (We have used 3.4.6)
2. Untar and put in a directory on all three servers. (e.g., `/mnt1/zookeeper-3.4.6`)
3. We have to setup `zoo.cfg` in `zookeeper-3.4.6/conf` directory on all three servers. There should be one `zoo_sample.cfg` shipped in the default release distribution.

     - The `zoo.cfg` looks like this:
     ```bash
     [zookeeper-3.4.6/conf]$ cat zoo.cfg
     dataDir=/mnt1/data/zookeeper
     dataLogDir=/zklog/datalog
     clientPort=2181
     initLimit=5
     syncLimit=2
     server.1=10.0.1.167:2888:3888
     server.2=10.0.10.205:2888:3888
     server.3=10.0.2.66:2888:3888
     maxClientCnxns=0
     autopurge.snapRetainCount=3
     autopurge.purgeInterval=1
     ```

     - `clientPort` is where the zk process will start.
     - `server.{myid}` is the list of servers which will be part of the ensemble.

     - The `myid` of servers has to be put in data directory as a file. (we need to create file in the `dataDir`)
     ```bash
     [zookeeper-3.4.6/conf]$ cat /mnt1/data/zookeeper/myid
     1
     ```

     - Value for myid is `1` in the server with ip `10.0.1.167` as per the config. and other two has their own ids respectively.
4. Start Zookeeper on all three servers.
    ```bash
    [zookeeper-3.4.6/bin] ./zkServer.sh start
    ```

If config is okay that should setup the zookeeper ensemble. We can check the status on each Zookeeper server by the status command.
Exactly One of the servers in the ensemble will have the mode as `leader`:
```bash
[/mnt1/zookeeper-3.4.6/bin] ./zkServer.sh status
JMX enabled by default
Using config: /mnt1/zookeeper-3.4.6/bin/../conf/zoo.cfg
Mode: leader
```

Rest both servers will be in `follower` mode:
```bash
[mnt1/zookeeper-3.4.6/bin] ./zkServer.sh status
JMX enabled by default
Using config: /mnt1/zookeeper-3.4.6/bin/../conf/zoo.cfg
Mode: follower
```

##### Setup Solr Cluster
0. Three servers ready to launch solr.
1. Download latest Zookeeper from [their release page.](http://lucene.apache.org/solr/downloads.html) (We have used 6.4.1)
2. Untar and put in a directory on all three servers. (e.g., `/mnt1/solr-6.4.1`)
3. The default SOLR_HOME is `/solr-6.4.1/server/solr` which will be picked up if there is no external solr home passed as argument while start command. This directory expects `solr.xml` file to be present. The default location already has one present which looks like following:
    ```xml
    <?xml version="1.0" encoding="UTF-8" ?>
    <!--
     Licensed to the Apache Software Foundation (ASF) under one or more
     contributor license agreements.  See the NOTICE file distributed with
     this work for additional information regarding copyright ownership.
     The ASF licenses this file to You under the Apache License, Version 2.0
     (the "License"); you may not use this file except in compliance with
     the License.  You may obtain a copy of the License at

         http://www.apache.org/licenses/LICENSE-2.0

     Unless required by applicable law or agreed to in writing, software
     distributed under the License is distributed on an "AS IS" BASIS,
     WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
     See the License for the specific language governing permissions and
     limitations under the License.
    -->

    <!--
       This is an example of a simple "solr.xml" file for configuring one or
       more Solr Cores, as well as allowing Cores to be added, removed, and
       reloaded via HTTP requests.

       More information about options available in this configuration file,
       and Solr Core administration can be found online:
       http://wiki.apache.org/solr/CoreAdmin
    -->

    <solr>

      <solrcloud>

        <str name="host">${host:}</str>
        <int name="hostPort">${jetty.port:8983}</int>
        <str name="hostContext">${hostContext:solr}</str>

        <bool name="genericCoreNodeNames">${genericCoreNodeNames:true}</bool>

        <int name="zkClientTimeout">${zkClientTimeout:30000}</int>
        <int name="distribUpdateSoTimeout">${distribUpdateSoTimeout:600000}</int>
        <int name="distribUpdateConnTimeout">${distribUpdateConnTimeout:60000}</int>
        <str name="zkCredentialsProvider">${zkCredentialsProvider:org.apache.solr.common.cloud.DefaultZkCredentialsProvider}</str>
        <str name="zkACLProvider">${zkACLProvider:org.apache.solr.common.cloud.DefaultZkACLProvider}</str>

      </solrcloud>

      <shardHandlerFactory name="shardHandlerFactory"
        class="HttpShardHandlerFactory">
        <int name="socketTimeout">${socketTimeout:600000}</int>
        <int name="connTimeout">${connTimeout:60000}</int>
      </shardHandlerFactory>

    </solr>
    ```

4. SolrCloud has one benefit of using the centralized configuration. Where the config for the collection stays in the Zookeeper. We can upload config for one time to ZK, using the scripts provided in the solr distribution. (this is needed only from one server, one time for collection)
We can skip this step if we have the config already uploaded to zookeeper.

    - You need to have `managed-schema` (`schema.xml` has become `managed-schema` in latest solr upgrade) and `solrconfig.xml` for your collection. Let say it is in one location on a server.
    ```bash
    [/mnt1/solr-6.4.1/server/solr/configsets/my_collection/conf] ls
    managed-schema  solrconfig.xml
    ```
    - to upload config to Zookeeper:
    ```bash
    [/mnt1/solr-6.4.1/server/scripts/cloud-scripts] ./solr zk upconfig -z 10.0.1.167:2181,10.0.10.205:2181,10.0.2.66:2181 -n my_collection_config -d /mnt1/solr-6.4.1/server/solr/configsets/my_collection/conf
    ```
    - here `10.0.1.167:2181,10.0.10.205:2181,10.0.2.66:2181` is the "zookeeper connection string" - this has to be IPs of the ensemble we setup.

5. Starting solr as SolrCloud node (needs to be done on all nodes of the cluster):
    ```bash
    ./mnt1/solr-6.4.1/bin/solr start -cloud -p 8983 -z 10.0.1.167:2181,10.0.10.205:2181,10.0.2.66:2181 -m 2g
    ```
    - `-z` is for zookeeper collection string. When the node starts up it will connect to this Zookeeper ensemble. The Zookeeper needs to be properly setup.
    - `-cloud` starts the solr on cloud. When this option is specified it is must to provide "zookeeper connection string" with option `-z`.
    - `-p` is for the port number for the HTTP listen of SolrCloud. defaul is `8983`.
    - `-m` is for memory.
    - There can be many other options passed as startup. Can be found [here](https://cwiki.apache.org/confluence/display/solr/Solr+Control+Script+Reference).

6. This should get the SolrCloud up and running. Check the status of solr server:
    ```bash
    /mnt1/solr-6.4.1/bin] ./solr status

    Found 1 Solr nodes:

    Solr process 22352 running on port 8983
    {
      "solr_home":"/mnt1/solr-6.4.1/server/solr",
      "version":"6.4.1 72f75b2503fa0aa4f0aff76d439874feb923bb0e - jpountz - 2017-02-01 14:49:06",
      "startTime":"2017-02-23T12:39:34.899Z",
      "uptime":"5 days, 22 hours, 6 minutes, 43 seconds",
      "memory":"384.9 MB (%19.6) of 1.9 GB",
      "cloud":{
        "ZooKeeper":"10.0.1.167:2181,10.0.10.205:2181,10.0.2.66:2181",
        "liveNodes":"3",
        "collections":"1"}}

    ```
    - P.S.: If your SolrCloud doesn't have any collection yet there can be "0" in the `collections`.

##### Setup Collection
There is an API now in latest Solr to create collections via REST. Following curl from anyone of the solr server:
```
curl "http://localhost:8983/solr/admin/collections?action=CREATE&name=myindex&numShards=2&replicationFactor=2&maxShardsPerNode=2&collection.configName=my_collection_config"

<?xml version="1.0" encoding="UTF-8"?>
<response>
<lst name="responseHeader"><int name="status">0</int><int name="QTime">4083</int></lst><lst name="success"><lst name="10.0.10.205:8983_solr"><lst name="responseHeader"><int name="status">0</int><int name="QTime">2677</int></lst><str name="core">myindex_shard2_replica1</str></lst><lst name="10.0.1.167:8983_solr"><lst name="responseHeader"><int name="status">0</int><int name="QTime">2681</int></lst><str name="core">myindex_shard1_replica2</str></lst><lst name="10.0.2.66:8983_solr"><lst name="responseHeader"><int name="status">0</int><int name="QTime">2941</int></lst><str name="core">myindex_shard1_replica1</str></lst><lst name="10.0.2.66:8983_solr"><lst name="responseHeader"><int name="status">0</int><int name="QTime">2946</int></lst><str name="core">myindex_shard2_replica2</str></lst></lst>
</response>
```
All the params are important, and plays role in the load and QPS of the collection.

The above request will create a collection named `myindex` with 2 shards and 2 replicas. And it assumes that the `collection.configName` is already uploaded on Zookeeper.

We can very this by checking the `clusterstatus`.
```
curl "http://localhost:8983/solr/admin/collections?action=clusterstatus&wt=json"


{
  "responseHeader": {
    "status": 0,
    "QTime": 5
  },
  "cluster": {
    "collections": {
      "myindex": {
        "replicationFactor": "2",
        "shards": {
          "shard1": {
            "range": "80000000-ffffffff",
            "state": "active",
            "replicas": {
              "core_node3": {
                "core": "myindex_shard1_replica2",
                "base_url": "http://10.0.1.167:8983/solr",
                "node_name": "10.0.1.167:8983_solr",
                "state": "active",
                "leader": "true"
              },
              "core_node4": {
                "core": "myindex_shard1_replica1",
                "base_url": "http://10.0.2.66:8983/solr",
                "node_name": "10.0.2.66:8983_solr",
                "state": "active"
              }
            }
          },
          "shard2": {
            "range": "0-7fffffff",
            "state": "active",
            "replicas": {
              "core_node1": {
                "core": "myindex_shard2_replica1",
                "base_url": "http://10.0.10.205:8983/solr",
                "node_name": "10.0.10.205:8983_solr",
                "state": "active",
                "leader": "true"
              },
              "core_node2": {
                "core": "myindex_shard2_replica2",
                "base_url": "http://10.0.2.66:8983/solr",
                "node_name": "10.0.2.66:8983_solr",
                "state": "active"
              }
            }
          }
        },
        "router": {
          "name": "compositeId"
        },
        "maxShardsPerNode": "2",
        "autoAddReplicas": "false",
        "znodeVersion": 6,
        "configName": "my_collection_conf"
      }
    },
    "live_nodes": [
      "10.0.1.167:8983_solr",
      "10.0.10.205:8983_solr",
      "10.0.2.66:8983_solr"
    ]
  }
}
```

This should setup the SolrCloud.

References:
1. [https://cwiki.apache.org/confluence/display/solr/Using+ZooKeeper+to+Manage+Configuration+Files](https://cwiki.apache.org/confluence/display/solr/Using+ZooKeeper+to+Manage+Configuration+Files)
2. [https://cwiki.apache.org/confluence/display/solr/Distributed+Requests](https://cwiki.apache.org/confluence/display/solr/Distributed+Requests)
3. [https://cwiki.apache.org/confluence/display/solr/Solr+Control+Script+Reference](https://cwiki.apache.org/confluence/display/solr/Solr+Control+Script+Reference)
4. [http://stackoverflow.com/questions/27857450/how-to-use-upconfig-linkconfig-scripts-on-external-zookeeper](http://stackoverflow.com/questions/27857450/how-to-use-upconfig-linkconfig-scripts-on-external-zookeeper)
5. [https://myjeeva.com/solrcloud-cluster-single-collection-deployment.html](https://myjeeva.com/solrcloud-cluster-single-collection-deployment.html)
6. [https://sematext.com/blog/2016/10/19/handling-shards-in-solrcloud/](https://sematext.com/blog/2016/10/19/handling-shards-in-solrcloud/)
7. [http://community.cloudera.com/t5/Cloudera-Search-Apache-SolrCloud/quot-No-live-SolrServers-available-to-handle-this-request-quot/m-p/26152](http://community.cloudera.com/t5/Cloudera-Search-Apache-SolrCloud/quot-No-live-SolrServers-available-to-handle-this-request-quot/m-p/26152)
8. [http://stackoverflow.com/questions/22143018/query-solr-cluster-for-state-of-nodes](http://stackoverflow.com/questions/22143018/query-solr-cluster-for-state-of-nodes)
9. [https://cwiki.apache.org/confluence/display/solr/Command+Line+Utilities](https://cwiki.apache.org/confluence/display/solr/Command+Line+Utilities)