---
layout: post 
title: Mongo Cluster 구성
category: [mongo]
tags: [mongo, cluster, psa]
redirect_from:

- /2021/07/19/

---

## MongoDB Shard Cluster 구성  
현재 운영중인 시스템의 몽고버전은 3.4.x인데 올해내로 4.x 메이저 버전 업데이트를 해야한다. 업데이트를 위해서는 현재 시스템 구성에 대한 이해 및 사전준비가 필요할 것 같아 GCP 클라우드서버에 현재 운영중인 시스템과 비슷한 Mongo Cluster 구성을 해보고 버전 업데이트를 해보려고 한다. 서버 구성은 P-S-A 레플리카 세트 2개와 Config,mongos 2대, 아비터 서버 1대로 구성할 예정이다.

## Pre Setting  
### GCP 노드생성  
GCP 클라우드에서 mongo P-S-A 구성을 위한 노드 5대, config,mongos를 위한 노드 2대 서버를 생성한다.   
<img src="https://sisipapa.github.io/assets/images/posts/gcp-mongo.PNG" >  

### Mongo 설치
YUM 레포지토리에 mongodb 4.2 repo를 추가  
```shell
$ vi /etc/yum.repos.d/mongodb-org-4.2.repo

[mongodb-org-4.2]
name=MongoDB Repository
baseurl=https://repo.mongodb.org/yum/redhat/$releasever/mongodb-org/4.2/x86_64/
gpgcheck=1
enabled=1
gpgkey=https://www.mongodb.org/static/pgp/server-4.2.asc
```  

yum install  
```shell
$ yum install -y mongodb-org  
Loaded plugins: fastestmirror
Determining fastest mirrors
epel/x86_64/metalink                                                                                                                                                                                                                                   | 4.9 kB  00:00:00     
 * base: mirror.kakao.com
 * epel: ftp.iij.ad.jp
 * extras: mirror.kakao.com
 * updates: mirror.kakao.com
base                                                                                                                                                                                                                                                   | 3.6 kB  00:00:00     
epel                                                                                                                                                                                                                                                   | 4.7 kB  00:00:00     
extras                                                                                                                                                                                                                                                 | 2.9 kB  00:00:00     
google-cloud-sdk                                                                                                                                                                                                                                       | 1.4 kB  00:00:00     
google-compute-engine                                                                                                                                                                                                                                  | 1.4 kB  00:00:00     
mongodb-org-4.2                                                                                                                                                                                                                                        | 2.5 kB  00:00:00     
updates                                                                                                                                                                                                                                                | 2.9 kB  00:00:00     
(1/10): base/7/x86_64/group_gz                                                                                                                                                                                                                         | 153 kB  00:00:00     
(2/10): extras/7/x86_64/primary_db                                                                                                                                                                                                                     | 242 kB  00:00:00     
(3/10): base/7/x86_64/primary_db                                                                                                                                                                                                                       | 6.1 MB  00:00:00     
(4/10): epel/x86_64/group_gz                                                                                                                                                                                                                           |  96 kB  00:00:00     
(5/10): epel/x86_64/updateinfo                                                                                                                                                                                                                         | 1.0 MB  00:00:00     
(6/10): updates/7/x86_64/primary_db                                                                                                                                                                                                                    | 8.8 MB  00:00:00     
(7/10): google-cloud-sdk/primary                                                                                                                                                                                                                       | 209 kB  00:00:00     
(8/10): mongodb-org-4.2/7/primary_db                                                                                                                                                                                                                   |  69 kB  00:00:00     
(9/10): google-compute-engine/primary                                                                                                                                                                                                                  | 3.8 kB  00:00:00     
(10/10): epel/x86_64/primary_db                                  
...
```  

## Replicaset 노드 설정
- hostname  
Primary hostname: mongodbp01, mongodbp02  
Secondary hostname: mongodbs01, mongodbs02  
Arbiter hostname: monogarb  
  
- Replicaset 구성  
rs0 – mongodbp01:27018, mongodbs01:27018, mongoarb:27021  
rs1 – mongodbp02:27018, mongodbs02:27018, mongoarb:27022  
  
- port 구성  
27017 : MongoDB의 기본 포트 번호이자 샤드를 구성할 때 Mongos가 이용하는 포트 번호  
27018 : 샤드를 구성할 때 샤드 서버들이 사용하는 포트번호  
27019 : Config 서버들이 사용하는 포트 번호  

### Replicaset mongod.conf
home 디렉토리 하위에 conf 디렉토리를 생성하고 mongod.conf 파일을 생성.  
```shell
$ vi /etc/mongod.conf  

# for documentation of all options, see:
#   http://docs.mongodb.org/manual/reference/configuration-options/
# where to write logging data.
systemLog:
  destination: file
  logAppend: true
  path: /var/log/mongodb/mongod.log
# Where and how to store data.
storage:
  dbPath: /var/lib/mongo
  journal:
    enabled: true
    commitIntervalMs: 200
#  engine:
  wiredTiger:
    engineConfig:
      cacheSizeGB: 4
      journalCompressor: snappy
      directoryForIndexes: false
    collectionConfig:
      blockCompressor: snappy
    indexConfig:
      prefixCompression: true
# how the process runs
processManagement:
  fork: true  # fork and run in background
  pidFilePath: /var/run/mongodb/mongod.pid  # location of pidfile
  timeZoneInfo: /usr/share/zoneinfo
# network interfaces
net:
  port: 27018
  bindIp: 0.0.0.0  # Enter 0.0.0.0,:: to bind to all IPv4 and IPv6 addresses or, alternatively, use the net.bindIpAll setting.
setParameter:
  enableLocalhostAuthBypass: false
#security:
#operationProfiling:
replication:
  replSetName: "rs0"
sharding:
  clusterRole: shardsvr
## Enterprise-Only Options
#auditLog:
#snmp:
```  
노드별로 수정해야 할 부분(replication.replSetName)  
```shell
...
replication:
  replSetName: "rs0"
...  
```  

### Arbiter mongod.conf(rs0,rs1)
mongod0.conf, mongod1.conf 두개의 Arbiter 설정파일을 생성.  
```shell
$ vi /etc/mongod0.conf
# for documentation of all options, see:
#   http://docs.mongodb.org/manual/reference/configuration-options/
# where to write logging data.
systemLog:
  destination: file
  logAppend: true
  path: /var/log/mongodb/mongod_arb0.log
# Where and how to store data.
storage:
  dbPath: /var/lib/mongo/arb0
  journal:
    enabled: true
    commitIntervalMs: 200
#  engine:
  wiredTiger:
    engineConfig:
      cacheSizeGB: 1
      journalCompressor: snappy
      directoryForIndexes: false
    collectionConfig:
      blockCompressor: snappy
    indexConfig:
      prefixCompression: true
# how the process runs
processManagement:
  fork: true  # fork and run in background
  pidFilePath: /var/run/mongodb/mongod_arb0.pid  # location of pidfile
  timeZoneInfo: /usr/share/zoneinfo
# network interfaces
net:
  port: 27021
  bindIp: 0.0.0.0  # Enter 0.0.0.0,:: to bind to all IPv4 and IPv6 addresses or, alternatively, use the net.bindIpAll setting.
setParameter:
  enableLocalhostAuthBypass: false
#security:
#operationProfiling:
replication:
  replSetName: "rs0"
sharding:
  clusterRole: shardsvr
## Enterprise-Only Options
#auditLog:
#snmp:

```  
수정해야 할 부분(systemLog.path, storage.dbPath, processManagement.pidFilePath, net.port, replication.replSetName)  
Arbiter의 경우 하나의 노드에 port로 구분해서 두개의 mongo를 띄우는 거라 수정할 부분이 많다.  
```shell
systemLog:
...
  path: /var/log/mongodb/mongod_arb0.log
...
storage:
  dbPath: /var/lib/mongo/arb0
...
processManagement:
  ...
  pidFilePath: /var/run/mongodb/mongod_arb0.pid  # location of pidfile
...
net:
  port: 27021
...
...
replication:
  replSetName: "rs0"
...
```  
실행  
```shell
$ mongod -f conf/mongod0.conf
$ mongod -f conf/mongod1.conf
```  

### Replicaset 구성  
```shell
> rs.initiate( {
   _id : "rs0",
   members: [
      { _id: 0, host: "mongodbp01:27018" },
      { _id: 1, host: "mongodbs01:27018" },
      { _id: 2, host: "mongoarb:27021", arbiterOnly:true }
   ]
})

```







## 참고  
[NCloud MongoDB Cluster 구성하기](https://guide.ncloud-docs.com/docs/database-database-10-3)  
[MongoDB Shard Cluster 구성하기](https://rastalion.me/mongodb-shard-cluster-%EA%B5%AC%EC%84%B1%ED%95%98%EA%B8%B0/)

