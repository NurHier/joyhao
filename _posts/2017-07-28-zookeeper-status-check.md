---
layout: post
title: zookeeper状态查看工具
categories: articles
tags: [zookeeper]
comments: true
description: 本工具用于查看zookeeper集群的运行状态和节点的角色,脚本运行的机器需要可免密码登录到各个zk节点。
---

本工具用于查看zookeeper集群的运行状态和节点的角色，并选中leader节点，通过 `echo mntr|nc ${leaderIP} ${zkPort}`  获取集群的信息。
脚本运行的机器需要可**免密码登录**到各个zk节点。

### 1.代码实例

```shell 
#!/bin/bash
# 查看zk运行状态

nodeips=(10.10.0.102 10.10.0.103 10.10.0.104)
port=2183
zkPath="/opt/zookeeper-3.4.8/bin"
leaderIP=""
for ip in ${nodeips[@]}
do
  echo $ip
  info=`ssh root@$ip << EOF
    cd ${zkPath}
    sh ${zkPath}/zkServer.sh status
    exit 0
EOF`
  echo $info
  role=`echo $info|awk -F ":" '{print $2}'|sed s/[[:space:]]//g`
  if [ "$role"x == "leader"x ]; then
    leaderIP=$ip
  fi
  echo "-------------------------------------"
done
echo "echo mntr|nc ${leaderIP} ${port}"
ssh root@${leaderIP} "cd ${zkPath}; echo mntr|nc ${leaderIP} ${port}"
exit 0
```
### 2. 运行结果

```shell
[root@security tools]$sh zk-status.sh 
10.10.0.102
Pseudo-terminal will not be allocated because stdin is not a terminal.
ZooKeeper JMX enabled by default
Using config: /opt/zookeeper-3.4.8/bin/../conf/zoo.cfg
Mode: follower
-------------------------------------
10.10.0.103
Pseudo-terminal will not be allocated because stdin is not a terminal.
ZooKeeper JMX enabled by default
Using config: /opt/zookeeper-3.4.8/bin/../conf/zoo.cfg
Mode: leader
-------------------------------------
10.10.0.104
Pseudo-terminal will not be allocated because stdin is not a terminal.
ZooKeeper JMX enabled by default
Using config: /opt/zookeeper-3.4.8/bin/../conf/zoo.cfg
Mode: follower
-------------------------------------
echo mntr|nc 10.10.0.103 2183
zk_version	3.4.8--1, built on 02/06/2016 03:18 GMT
zk_avg_latency	0
zk_max_latency	11
zk_min_latency	0
zk_packets_received	237879
zk_packets_sent	237879
zk_num_alive_connections	3
zk_outstanding_requests	0
zk_server_state	leader
zk_znode_count	40
zk_watch_count	2
zk_ephemerals_count	0
zk_approximate_data_size	11090
zk_open_file_descriptor_count	33
zk_max_file_descriptor_count	65535
zk_followers	2
zk_synced_followers	2
zk_pending_syncs	0

```