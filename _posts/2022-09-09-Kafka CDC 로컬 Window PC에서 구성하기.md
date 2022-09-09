---
layout: post
title: Kafka CDC 로컬 Window PC에서 구성하기
category: [kafka]
tags: [kafka, cdc, docker]
redirect_from:

- /2022/09/09/

---

최근 Kafka CDC 라는걸 알게 되었고 어떻게 구성을 하고 동작을 하는지 알아보기 위해 개인 로컬 PC에 환경구성을 하면서 정리한 내용입니다. 아래 참고 블로그의 내용을 그대로 실행하면서 정리한 내용입니다.   

## 1. docker-compose.yml 준비
아래 yml 파일을 실행하면 아래 첨부한 이미지와 동일한 2개의 Source, Sink용 Mysql DB와 Zookeeper, Kafka 컨테이너가 생성된다.  
```yaml
version: '3'
services:
  mysql-source:
    image: mysql:latest
    container_name: mysql-source
    ports:
      - 3306:3306
    environment:
      MYSQL_ROOT_PASSWORD: admin
      MYSQL_USER: mysqluser
      MYSQL_PASSWORD: mysqlpw 
    command:
      - --character-set-server=utf8mb4
      - --collation-server=utf8mb4_unicode_ci
    volumes:
      - C:/mysql-source/data:/var/lib/mysql
      
  mysql-sink:
    image: mysql:latest
    container_name: mysql-sink
    ports:
      - 3307:3306
    environment:
      MYSQL_ROOT_PASSWORD: admin
      MYSQL_USER: mysqluser
      MYSQL_PASSWORD: mysqlpw 
    command:
      - --character-set-server=utf8mb4
      - --collation-server=utf8mb4_unicode_ci
    volumes:
      - C:/mysql-sink/data:/var/lib/mysql

  zookeeper:
    container_name: zookeeper
    image: wurstmeister/zookeeper
    ports:
      - "2181:2181"

  kafka:
    container_name: kafka
    image: wurstmeister/kafka
    depends_on:
      - zookeeper
    ports:
      - "9092:9092"
    environment:
      KAFKA_ADVERTISED_HOST_NAME: 127.0.0.1
      KAFKA_ADVERTISED_PORT: 9092
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock  
```  

<img src="https://sisipapa.github.io/assets/images/posts/2022-09-09-kafka-connect.PNG" >  
[이미지출처](https://www.confluent.io/blog/no-more-silos-how-to-integrate-your-databases-with-apache-kafka-and-cdc/)  

## 2. docker-compose.yml 실행  
윈도우 CMD창에서 아래 명령을 실행하면 컨테이너가 생성된다.
```shell
docker-compose -f docker-compose.yml up -d 
```  

### 실행 후 컨테이너 확인  
DOS 명령어가 익숙하지 않다면 로컬에 설치된 git bash와 같은 shell 명령어가 되는 COMMAND 창에서 실행해도 동일한 결과를 확인할 수 있다.
```shell
$ docker ps
CONTAINER ID   IMAGE                    COMMAND                  CREATED        STATUS        PORTS                                                NAMES
d6759801dd79   wurstmeister/kafka       "start-kafka.sh"         14 hours ago   Up 14 hours   0.0.0.0:9092->9092/tcp                               kafka
850204814ae6   mysql:latest             "docker-entrypoint.s…"   14 hours ago   Up 14 hours   33060/tcp, 0.0.0.0:3307->3306/tcp                    mysql-sink
28690f7d4fca   wurstmeister/zookeeper   "/bin/sh -c '/usr/sb…"   14 hours ago   Up 14 hours   22/tcp, 2888/tcp, 3888/tcp, 0.0.0.0:2181->2181/tcp   zookeeper
9eed78105b3b   mysql:latest             "docker-entrypoint.s…"   14 hours ago   Up 14 hours   0.0.0.0:3306->3306/tcp, 33060/tcp                    mysql-source
```  





## 참고
[MySQL 에서 Kafka 로 Source Connector 구축하기](https://wecandev.tistory.com/m/109)  
[Kafka 에서 Mysql 로 Sink Connector 구축하기](https://wecandev.tistory.com/110)  
[JDBC connector 생성 시 No suitable driver found 에러 발생](https://wecandev.tistory.com/111)


## Github
<https://github.com/sisipapa/kafka-cdc>  




