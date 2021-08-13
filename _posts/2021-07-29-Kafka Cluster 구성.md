---
layout: post 
title: kafka cluster 구성
category: [kafka]
tags: [kafka,cluster]
redirect_from:

- /2021/07/29/

---  

GCP 무료계정을 이용해서 Kafka Cluster를 구성해보려고 한다. 현재 운영중인 플랫폼에 Kafka Cluster가 구성이 되어있고 운영중에 있지만 내가 직접 Kafka Cluster 환경을 구축해 본 경험이 없어 해보려고 한다.  

## PreSetting  
Kafka Cluster 구성을 위해 GCP에 3개의 Instance를 생성한다.  
<img src="https://sisipapa.github.io/assets/images/posts/gcp-kafka-cluster.PNG" >  

kafka-cluster01, kafka-cluster02, kafka-cluster03 인스턴스에 들어가서 jdk를 설치하고 kafka를 다운로드 후 압축을 해제한다.
- jdk설치  
```shell
$ sudo yum install -y java-1.8.0-openjdk-devel.x86_64
```  

- kafka download
  <img src="https://sisipapa.github.io/assets/images/posts/kafka-download1.PNG" >  
  <img src="https://sisipapa.github.io/assets/images/posts/kafka-download2.PNG" >  
  
- 압축해제  
```shell
$ tar -xzf kafka-2.8.0-src.tgz
```  

## Zookeeper 설정
경로 - {카프카설치디렉토리}/config/zookeeper.properties
```properties  
$ vi zookeeper.properties

# 여러 zookeeper에 의해 저장된 znode의 사본과 zookeeper id를 저장하는 dir
dataDir=/home/kafka/zookeeper

# zookeeper가 coordinate하고 있는 kafka에 접속하기 위한 클라이언트 port
clientPort=2181

# 클라이언트 connection 접속 제한 개수
maxClientCnxns=10

# Follower가 leader에 접속하고 싱크를 맞추기 위해 허용된 시간. 값이 증가할수록 zookeeper가 책임지는 데이터의 양이 증가한다.
initLimit=5

####to allow followers to sync with ZooKeeper. If followers fall too far behind a leader, they will be dropped.
# Follower가 Zookeeper에 싱크를 맞추기 위해 허용된 시간. Follower가 leader에 대해 싱크를 맞추지 못하면 drop 된다고 햔다.
syncLimit=2

# Zookeeper VM의 정보
# 형식 : server.{id}={VM HostName or IP address}:{Cluster Sync Port}:{Leader Election Port}
# 입력시, localhost는 기입하지 않으며, 무조건 hostname, ip address를 이용하여 기입한더ㅏ.
server.1=10.174.0.2:2888:3888
server.2=10.174.0.3:2888:3888
server.3=10.174.0.4:2888:3888
```  

## kafka 설정  
경로 - {카프카설치디렉토리}/config/server.properties  
```properties
############################# Server Basics ############################# 
broker.id=1 

############################# Socket Server Settings ############################# 
listeners=PLAINTEXT://10.174.0.2:9092
num.network.threads=3 

############################# Log Basics ############################# 
num.partitions=3 

############################# Zookeeper ############################# 
zookeeper.connect=10.174.0.2:2181,10.174.0.3:2181,10.174.0.3:2181
```  

## myid 파일 생성(Instance 별로 위 kafka설정에 broker.id 값과 같은 값으로 세팅)  
- kafka-cluster01 접속
```shell
$ echo 1 > /home/kafka/zookeeper/myid
```  

- kafka-cluster02 접속
```shell
$ echo 2 > /home/kafka/zookeeper/myid
```  

- kafka-cluster03 접속
```shell
$ echo 3 > /home/kafka/zookeeper/myid
```  

## zookeeper 서버 기동  
```shell
$ sh bin/zookeeper-server-start.sh config/zookeeper.properties
```  

## kafka 서버 기동  
```shell
$ sh bin/kafka-server-start.sh config/server.properties
```

GCP에서 Kafka Cluster를 구성해 보려고 했으나 GCP 무료계정의 경우 한국 리전으로 최대 4개의 서버만 구성이 가능했다. 그래서 제일 가까운 리전을 선택해서 Kafka 클러스터를 구성해 보려고 했는데 접속이 불안정해서 구성을 진행하기가 힘들었다. 그래서 내린 결론은 AWS에서 2CPU,4G MEMORY정도로 docker-compose를 활용해 Kafka cluster를 구성해보려고 한다. Kafka Cluster 구성을 빨리하고 Springboot와의 연동을 정리해 보려고 했는데 시간이 많이 늦어졌다....

## 참고  
[Message Broker/Kafka # Kafka - 2 # Kafka Multi Cluster 구성](https://skysoo1111.tistory.com/75)
[플랫폼 개발팀 기술 블로그](https://team-platform.tistory.com/13)