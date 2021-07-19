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

### 

## 참고  
[NCloud MongoDB Cluster 구성하기](https://guide.ncloud-docs.com/docs/database-database-10-3)  
[MongoDB Shard Cluster 구성하기](https://rastalion.me/mongodb-shard-cluster-%EA%B5%AC%EC%84%B1%ED%95%98%EA%B8%B0/)

