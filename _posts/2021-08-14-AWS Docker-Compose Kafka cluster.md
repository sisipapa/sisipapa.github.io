---
layout: post 
title: AWS Docker-Compose Kafka Cluster
category: [kafka]
tags: [kafka, AWS]
redirect_from:

- /2021/08/14/

---

얼마 전 GCP 무료계정을 이용해서 3개의 인스턴스 생성 후 Kafka Cluster 구성을 해보려고 했으나 이런저런 이유로 시간이 지체 되서 AWS Docker-Compose를 활용해 하나의 인스턴스에서 Docker-Compose를 활용해 Kafka-Cluster를 구성해 보려고 한다.  

## PreSetting
### AWS EC2 인스턴스 생성
- Ubuntu Server 20.04 LTS (HVM), SSD Volume Type 선택  
- t2.medium(2CPU, 4GB) 선택 후 검토 및 시작 버튼 클릭

### docker 설치
root 계정으로 변경 후 apt update & apt upgrade를 먼저 진행 
```shell
$ apt update & apt upgrade
```  

필수 패키지 설치
```shell
$ sudo apt-get install apt-transport-https ca-certificates curl gnupg-agent software-properties-common
```  

GPG Key 인증
```shell
$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
```  

Docker Repository 등록
```shell
sudo add-apt-repository \
> "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
> $(lsb_release -cs) \
> stable"
Hit:1 http://ap-northeast-2.ec2.archive.ubuntu.com/ubuntu focal InRelease
Get:2 http://ap-northeast-2.ec2.archive.ubuntu.com/ubuntu focal-updates InRelease [114 kB]                                    
Get:3 http://ap-northeast-2.ec2.archive.ubuntu.com/ubuntu focal-backports InRelease [101 kB]                                            
Get:4 https://download.docker.com/linux/ubuntu focal InRelease [52.1 kB]                                                                 
Get:5 https://download.docker.com/linux/ubuntu focal/stable amd64 Packages [10.7 kB]                    
Get:6 http://security.ubuntu.com/ubuntu focal-security InRelease [114 kB]
Fetched 391 kB in 2s (225 kB/s)    
Reading package lists... Done
```  

apt Docker 설치
```shell
$ sudo apt-get update && sudo apt-get install docker-ce docker-ce-cli containerd.io
```  

시스템 부팅시 Docker 시작
```shell
$ sudo systemctl enable docker && service docker start
```  

docker-compose 설치
```shell
$ apt  install docker-compose
```  

## Docker-Compose Yaml 설치  
[Kafka Cluster 참고 Git 주소](https://github.com/conduktor/kafka-stack-docker-compose)  
```yaml
version: '2.1'

services:
  zoo1:
    image: zookeeper:3.4.9
    hostname: zoo1
    ports:
      - "2181:2181"
    environment:
        ZOO_MY_ID: 1
        ZOO_PORT: 2181
        ZOO_SERVERS: server.1=zoo1:2888:3888 server.2=zoo2:2888:3888 server.3=zoo3:2888:3888
    volumes:
      - ./zk-multiple-kafka-multiple/zoo1/data:/data
      - ./zk-multiple-kafka-multiple/zoo1/datalog:/datalog

  zoo2:
    image: zookeeper:3.4.9
    hostname: zoo2
    ports:
      - "2182:2182"
    environment:
        ZOO_MY_ID: 2
        ZOO_PORT: 2182
        ZOO_SERVERS: server.1=zoo1:2888:3888 server.2=zoo2:2888:3888 server.3=zoo3:2888:3888
    volumes:
      - ./zk-multiple-kafka-multiple/zoo2/data:/data
      - ./zk-multiple-kafka-multiple/zoo2/datalog:/datalog

  zoo3:
    image: zookeeper:3.4.9
    hostname: zoo3
    ports:
      - "2183:2183"
    environment:
        ZOO_MY_ID: 3
        ZOO_PORT: 2183
        ZOO_SERVERS: server.1=zoo1:2888:3888 server.2=zoo2:2888:3888 server.3=zoo3:2888:3888
    volumes:
      - ./zk-multiple-kafka-multiple/zoo3/data:/data
      - ./zk-multiple-kafka-multiple/zoo3/datalog:/datalog


  kafka1:
    image: confluentinc/cp-kafka:5.5.1
    hostname: kafka1
    ports:
      - "9092:9092"
    environment:
      KAFKA_ADVERTISED_LISTENERS: LISTENER_DOCKER_INTERNAL://kafka1:19092,LISTENER_DOCKER_EXTERNAL://${DOCKER_HOST_IP:-127.0.0.1}:9092
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: LISTENER_DOCKER_INTERNAL:PLAINTEXT,LISTENER_DOCKER_EXTERNAL:PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: LISTENER_DOCKER_INTERNAL
      KAFKA_ZOOKEEPER_CONNECT: "zoo1:2181,zoo2:2182,zoo3:2183"
      KAFKA_BROKER_ID: 1
      KAFKA_LOG4J_LOGGERS: "kafka.controller=INFO,kafka.producer.async.DefaultEventHandler=INFO,state.change.logger=INFO"
    volumes:
      - ./zk-multiple-kafka-multiple/kafka1/data:/var/lib/kafka/data
    depends_on:
      - zoo1
      - zoo2
      - zoo3

  kafka2:
    image: confluentinc/cp-kafka:5.5.1
    hostname: kafka2
    ports:
      - "9093:9093"
    environment:
      KAFKA_ADVERTISED_LISTENERS: LISTENER_DOCKER_INTERNAL://kafka2:19093,LISTENER_DOCKER_EXTERNAL://${DOCKER_HOST_IP:-127.0.0.1}:9093
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: LISTENER_DOCKER_INTERNAL:PLAINTEXT,LISTENER_DOCKER_EXTERNAL:PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: LISTENER_DOCKER_INTERNAL
      KAFKA_ZOOKEEPER_CONNECT: "zoo1:2181,zoo2:2182,zoo3:2183"
      KAFKA_BROKER_ID: 2
      KAFKA_LOG4J_LOGGERS: "kafka.controller=INFO,kafka.producer.async.DefaultEventHandler=INFO,state.change.logger=INFO"
    volumes:
      - ./zk-multiple-kafka-multiple/kafka2/data:/var/lib/kafka/data
    depends_on:
      - zoo1
      - zoo2
      - zoo3

  kafka3:
    image: confluentinc/cp-kafka:5.5.1
    hostname: kafka3
    ports:
      - "9094:9094"
    environment:
      KAFKA_ADVERTISED_LISTENERS: LISTENER_DOCKER_INTERNAL://kafka3:19094,LISTENER_DOCKER_EXTERNAL://${DOCKER_HOST_IP:-127.0.0.1}:9094
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: LISTENER_DOCKER_INTERNAL:PLAINTEXT,LISTENER_DOCKER_EXTERNAL:PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: LISTENER_DOCKER_INTERNAL
      KAFKA_ZOOKEEPER_CONNECT: "zoo1:2181,zoo2:2182,zoo3:2183"
      KAFKA_BROKER_ID: 3
      KAFKA_LOG4J_LOGGERS: "kafka.controller=INFO,kafka.producer.async.DefaultEventHandler=INFO,state.change.logger=INFO"
    volumes:
      - ./zk-multiple-kafka-multiple/kafka3/data:/var/lib/kafka/data
    depends_on:
      - zoo1
      - zoo2
      - zoo3
  kafka-manager:
    container_name: kafka-manager
    image: hlebalbau/kafka-manager:2.0.0.2
    restart: on-failure
    depends_on:
      - kafka1
      - kafka2
      - kafka3
      - zookeeper
    environment:
      ZK_HOSTS: zookeeper:2181
      APPLICATION_SECRET: "random-secret"
      KM_ARGS: -Djava.net.preferIPv4Stack=true
    ports:
      - "9000:9000"
```  
### 설치
```shell
# 로그를 확인하면서 실행
$ docker-compose -f kafka-cluster.yaml up

# backgroud 실행
$ docker-compose -d kafka-cluster.yaml up
```  

Docker-Compose가 정상적으로 실행되고 나면 kafka, zookeeper, kafka manager가 정상적으로 동작하는 것을 로그로 확인이 가능하다. yaml 내용중 volumes 부분은 각 경로에 맞게 커스텀해서 사용이 가능하다. 나의 경우는 [Apache kafka Installation by docker - DaddyProgrammer](https://daddyprogrammer.org/post/12087/apache-kafka-install-by-docker/) yaml 파일에 [docker-compose 를 사용하여 kafka Cluster 및 Kafka Manger 세팅하기](https://akageun.github.io/2020/05/01/docker-compose-kafka-cluster-manager.html) kafka manager 설정만 가져와서 사용했다.  

### kafka manager 접속(AWS 보안그룹설정) 
AWS는 기본적으로 Instance를 생성하면 22번 port만 열려있기 때문에 로컬PC에서 Kafka Manager에 접속을 하기 위해서는 9000번 port를 열어주어야 한다.  
AWS 메뉴 - 서비스 > 네트워크 및 보안 > 보안 그룹 - 우측 상단의 작업 Selectbox > 인바운드 규칙 편집  
<img src="https://sisipapa.github.io/assets/images/posts/aws-security-group1.png" >  
<img src="https://sisipapa.github.io/assets/images/posts/aws-security-group2.png" >  
<img src="https://sisipapa.github.io/assets/images/posts/aws-security-group3.png" >  

Kafka Manager를 활용해 Topic 생성,삭제 관리를 해볼 수도 있지만 이번에는 아빠프로그래머님의 블로그에 정리되어 있는 것처럼 Kafka 프로젝트내의 Shell을 이용해 Kafka Cluster 테스트를 해보려고 한다.  

## Kafka Cluster 테스트  





## 참고
[Ubuntu 20.04 Docker 설치하기](https://blog.dalso.org/linux/ubuntu-20-04-lts/13118)
[Apache kafka Installation by docker - DaddyProgrammer](https://daddyprogrammer.org/post/12087/apache-kafka-install-by-docker/)
[(docker-compose) zookeeper/kafka 클러스터 구성](https://javachoi.tistory.com/413)
[docker-compose 를 사용하여 kafka Cluster 및 Kafka Manger 세팅하기](https://akageun.github.io/2020/05/01/docker-compose-kafka-cluster-manager.html)

## Github

