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
GCP 클라우드에서 mongo P-S-A 구성을 위해서 Primary 노드 1대, Secondaray 노드 1대, Arbiter 노드 1대로 구성  
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
Primary hostname: mongodbp  
Secondary hostname: mongodbs  
Arbiter hostname: monogarb  
  
- Replicaset 구성  
rs0 – mongodbp:27018, mongodbs:27018, mongoarb:27021    
rs1 – mongodbp:27028, mongodbs:27028, mongoarb:27022   
rs1 – mongodbp:27038, mongodbs:27038, mongoarb:27023  
  
### Replicaset mongod.conf
Primary - mongodbp0.conf, mongodbp1.conf, mongodbp2.conf  
Secondary - mongodbs0.conf, mongodbs1.conf, mongodbs2.conf  
```shell
$ vi /home/mongo/mongodbp0.conf  

# for documentation of all options, see:
systemLog:
  destination: file
  logAppend: true
  path: /var/log/mongodb/mongod_p0.log
# Where and how to store data.
storage:
  dbPath: /var/lib/mongo/p0
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
  pidFilePath: /var/run/mongodb/mongod_p0.pid  # location of pidfile
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
systemLog:
...
  path: /var/log/mongodb/mongod_p0.log
...
storage:
  dbPath: /var/lib/mongo/p0
...
processManagement:
  ...
  pidFilePath: /var/run/mongodb/mongod_p0.pid  # location of pidfile
...
net:
  port: 27018
...
...
replication:
  replSetName: "rs0"
...
```  

실행
```shell
$ mongod -f conf/mongodp0.conf
$ mongod -f conf/mongodp1.conf
$ mongod -f conf/mongodp2.conf

$ mongod -f conf/mongods0.conf
$ mongod -f conf/mongods1.conf
$ mongod -f conf/mongods2.conf
```  

### Arbiter mongod.conf
mongodba0.conf, mongodbp1.conf, mongodbp2.conf 세개의 Arbiter 설정파일을 생성.  
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
$ mongod -f conf/mongoda0.conf
$ mongod -f conf/mongoda1.conf
$ mongod -f conf/mongoda2.conf
```  

### Replicaset 구성  
```shell
$ mongo --port 27018
> rs.initiate( {
   _id : "rs0",
   members: [
      { _id: 0, host: "mongodbp:27018" },
      { _id: 1, host: "mongodbs:27018" },
      { _id: 2, host: "monogarb:27021", arbiterOnly:true }
   ]
})
{ "ok" : 1 }


$ mongo --port 27028
> rs.initiate( {
   _id : "rs1",
   members: [
      { _id: 0, host: "mongodbp:27028" },
      { _id: 1, host: "mongodbs:27028" },
      { _id: 2, host: "monogarb:27022", arbiterOnly:true }
   ]
})
{ "ok" : 1 }

$ mongo --port 27038
> rs.initiate( {
   _id : "rs2",
   members: [
      { _id: 0, host: "mongodbp:27038" },
      { _id: 1, host: "mongodbs:27038" },
      { _id: 2, host: "monogarb:27023", arbiterOnly:true }
   ]
})
{ "ok" : 1 }
```  

## 참고  
[NCloud MongoDB Cluster 구성하기](https://guide.ncloud-docs.com/docs/database-database-10-3)  
[MongoDB Shard Cluster 구성하기](https://rastalion.me/mongodb-shard-cluster-%EA%B5%AC%EC%84%B1%ED%95%98%EA%B8%B0/)

