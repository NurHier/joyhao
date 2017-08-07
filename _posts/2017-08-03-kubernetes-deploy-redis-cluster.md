---
layout: post
title: 使用kubernetes部署redis cluster
categories: articles
tags: [kubernetes,redis,docker]
comments: true
description: 使用kubernetes部署3主3从的redis cluster，为保证redis 的master和slave实例分布到不同的node上，至少需要2个node节点。将redis cluster部署到docker中，使用kubernetes管理docker集群，可以依靠kubernetes的一些特性实现redis cluster的node节点选取，宕机恢复等工作。
---

### 介绍
使用kubernetes部署3主3从的redis cluster，为保证redis 的master和slave实例分布到不同的node上，至少需要2个node节点。

### 部署方案
将redis cluster部署到docker中，使用kubernetes管理docker集群，可以依靠kubernetes的一些特性实现redis cluster的node节点选取，宕机恢复等工作。

kubernetes：
1. 使用kubernetes 的ReplicaSet类型，此类型为ReplicationController的升级版，更好的支持selector，且当redis cluster 扩容缩容时修改RS配置后相应的pod不会重启导致redis实例异常
2. 创建2个pod副本，2个副本分布作为master和slave。使用kubernetes的pod反亲和性将2个副本部署到不同node上。
3. 使用环境变量配置redis实例需要的port、maxmemory、requirepass等信息，并可以写入到docker容器中，使redis.conf生效
5. kubernetes中限制docker容器的cpu和内存
6. redis cluster规范中建议使用docker host网络模式，网络损耗小，且易于管理
7. 由于redis可持久化，所以实例的持久化文件需要通过hostPath方式挂载到docker的宿主机上，以便用于恢复

docker：
1. docker镜像中打包 redis server
2. 在docker容器中使用[dockerize](https://github.com/jwilder/dockerize)生成redis.conf文件


### 部署流程

#### 1. 部署环境和版本

| node ip            | 用途                  |          
| :-------:     | :--------: |       
| 10.10.0.1   | kubernetes master | 
| 10.10.0.2   | kubernetes proxy  |

 | server| version|
| :-----| :-------: |
| docker | 1.12.3 |
| kubernetes | 1.6.4 |
| redis | 3.2.9 |

#### 2. 制作docker镜像

编写dockerfile，其中使用 dockerize插件通过redis.tmpl模板生成redis.conf配置文件
```yaml
FROM centos:7
MAINTAINER The centos project for redis
ENV REDIS_VERSION 3.2.9
COPY dockerize /usr/local/bin/
COPY redis.tmpl /opt/
COPY redis-3.2.9 /opt/redis-3.2.9
RUN mkdir -p /opt/data
```
redis.tmpl

执行docker命令构建镜像
```
docker build -t centos/redis-cluster .
```
#### 3. 创建redis cluster pod

需要创建3组RS，端口分别为7000,7001,7002
创建命令
```
kubectl create -f redis-cluster.yaml 
```
redis-cluster.yaml文件
```yaml
apiVersion: extensions/v1beta1
kind: ReplicaSet 
metadata:
  name: redis-7000
  annotations:
    description: "redis cluster service"
    clusterId: "7000"
spec: 
  replicas: 2
  template:
    metadata:
      labels: 
        app: redis
        name: redis-7000
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: name
                operator: In
                values: 
                - redis-7000
            topologyKey: kubernetes.io/hostname
      hostNetwork: true
      containers: 
      - image: centos/redis-cluster:latest
        name: redis-cluster
        imagePullPolicy: Never
        env:
        - name: CLUSTER_ID
          value: "7000"
        - name: REDIS_PORT
          value: "7000"
        - name: REDIS_MAXMEM
          #value: "3221225472"
          valueFrom:
            secretKeyRef:
              name: redissecret
              key: redismaxmem
        - name: REDIS_PASS
          #value: "RedisPass"
          valueFrom:
            secretKeyRef:
              name: redissecret
              key: redispass
        command: ["/bin/bash", "-c"]
        args: ["cd /opt/data/dump/; for i in `ls|grep -P '^$(CLUSTER_ID)'`;do name=`date +%T`;mv $i $name$i;done; dockerize -template /opt/redis.tmpl:/opt/redis-3.2.9/redis-cluster.conf /opt/redis-3.2.9/src/red
is-server /opt/redis-3.2.9/redis-cluster.conf"]
        resources:
          requests:
            cpu: 1
            memory: 6144Mi
        volumeMounts:
        - mountPath: /opt/data
          name: redis-volume
      volumes:
      - name: redis-volume
        hostPath:
          path: /opt/redisData

```

4. 组建redis cluster集群、分槽，并添加slave节点

组建redis cluster集群

```
/home/redis-3.2.9/src/redis-cli -h 10.10.0.1 -p 7000 -a RedisPass cluster meet 10.10.0.1 7001
/home/redis-3.2.9/src/redis-cli -h 10.10.0.1 -p 7000 -a RedisPass cluster meet 10.10.0.1 7002
/home/redis-3.2.9/src/redis-cli -h 10.10.0.1 -p 7000 -a RedisPass cluster meet 10.10.0.2 7000
/home/redis-3.2.9/src/redis-cli -h 10.10.0.1 -p 7000 -a RedisPass cluster meet 10.10.0.2 7001
/home/redis-3.2.9/src/redis-cli -h 10.10.0.1 -p 7000 -a RedisPass cluster meet 10.10.0.2 7002

```

分槽脚本

```shell

#!/bin/bash
for i in `seq 0 5461`
do
slot1=${slot1}" "${i}
done
echo ${slot1}
/home/redis-3.2.9/src/redis-cli -h 10.10.0.1 -p 7000 -a RedisPass cluster addslots ${slot1}

for a in `seq 5462 10922`
do
slot2=${slot2}" "${a}
done
echo ${slot2}
/home/redis-3.2.9/src/redis-cli -h 10.10.0.1 -p 7001 -a RedisPass CLUSTER ADDSLOTS ${slot2}

for k in `seq 10923 16383`
do
slot3=${slot3}" "${k}
done

/home/redis-3.2.9/src/redis-cli -h 10.10.0.1 -p 7002 -a RedisPass CLUSTER ADDSLOTS ${slot3}

```

### 部署中的问题

1. redis cluster master故障后slave节点不能自动提升为master
2. redis集群缩容时需要更新RS的maxmemory环境变量，更新后若容器故障重启，不会使用更新后的RS配置（delete pod 操作新配置会生效） 。解决方法：使用secret配置环境变量





