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
아래 yml 파일을 실행하면 아래 첨부한 이미지와 동일한 2개의 Source, Sink용 MySQL DB와 Zookeeper, Kafka 컨테이너가 생성된다.  
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

<img src="https://sisipapa.github.io/assets/images/posts/2022-09-09-kafka-connect.png" >  
[이미지출처](https://www.confluent.io/blog/no-more-silos-how-to-integrate-your-databases-with-apache-kafka-and-cdc/)  

## 2. docker-compose.yml 실행  
윈도우 CMD창에서 아래 명령을 실행하면 컨테이너가 생성된다.
```shell
# docker-compose -f docker-compose.yml up -d 
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

## 3. Source-mysql 설정  
### source-mysql 컨테이저 접속  
```shell
# docker exec -ti 9eed78105b3b bash  
```  
  
### Database, Table 생성
```shell
# mysql -u root -p
```  
```mysql
create database testdb;

use testdb;

CREATE TABLE accounts (
   account_id VARCHAR(255),
   role_id VARCHAR(255),
   user_name VARCHAR(255),
   user_description VARCHAR(255),
   update_date DATETIME DEFAULT CURRENT_TIMESTAMP,
   PRIMARY KEY (account_id)
);
```  

### 사용자 권한 확인  
컨테이너 생성시 docker-compose.yml 파일에서 생성한 mysqluser 파일에 권한을 부여한다.  
```mysql
use mysql;

GRANT ALL PRIVILEGES ON *.* TO 'mysqluser'@'%';

FLUSH PRIVILEGES;
```  

## 4. Source Connector 설치  - Debezium Connector
### 플러그인 다운로드    
[플러그인 다운로드 - debezium-connector-mysql-1.5.4.Final-plugin.tar.gz](https://debezium.io/releases/1.5/)  

### 카프카 컨테이너의 /opt/kafka_2.13-2.8.1/ 하위에 connectors 디렉토리를 생성
```shell
# docker ps
CONTAINER ID   IMAGE                    COMMAND                  CREATED        STATUS        PORTS                                                NAMES
d6759801dd79   wurstmeister/kafka       "start-kafka.sh"         14 hours ago   Up 14 hours   0.0.0.0:9092->9092/tcp                               kafka
850204814ae6   mysql:latest             "docker-entrypoint.s…"   14 hours ago   Up 14 hours   33060/tcp, 0.0.0.0:3307->3306/tcp                    mysql-sink
28690f7d4fca   wurstmeister/zookeeper   "/bin/sh -c '/usr/sb…"   14 hours ago   Up 14 hours   22/tcp, 2888/tcp, 3888/tcp, 0.0.0.0:2181->2181/tcp   zookeeper
9eed78105b3b   mysql:latest             "docker-entrypoint.s…"   14 hours ago   Up 14 hours   0.0.0.0:3306->3306/tcp, 33060/tcp                    mysql-source

# docker exec -ti d6759801dd79 bash
root@d6759801dd79:/# mkdir /opt/kafka_2.13-2.8.1/connectors
```  

### 카프카 컨테이너 안으로 복사
```shell
# docker cp debezium-connector-mysql-1.5.4.Final-plugin.tar.gz kafka:/opt/kafka_2.13-2.8.1/connectors/debezium-connector-mysql-1.5.4.Final-plugin.tar.gz  
```  

### 카프카 컨테이너  커넥터 플러그인 압축 해제
```shell
# # docker exec -ti d6759801dd79 bash
root@d6759801dd79:/# cd /opt/kafka_2.13-2.8.1/connectors
root@d6759801dd79:/# tar -zxvf debezium-connector-mysql-1.5.4.Final-plugin.tar.gz  

debezium-connector-mysql/CHANGELOG.md
debezium-connector-mysql/CONTRIBUTE.md
debezium-connector-mysql/COPYRIGHT.txt
debezium-connector-mysql/LICENSE-3rd-PARTIES.txt
debezium-connector-mysql/LICENSE.txt
debezium-connector-mysql/README.md
debezium-connector-mysql/README_ZH.md
debezium-connector-mysql/debezium-core-1.5.4.Final.jar
debezium-connector-mysql/debezium-api-1.5.4.Final.jar
debezium-connector-mysql/guava-30.0-jre.jar
debezium-connector-mysql/failureaccess-1.0.1.jar
debezium-connector-mysql/debezium-ddl-parser-1.5.4.Final.jar
debezium-connector-mysql/antlr4-runtime-4.7.2.jar
debezium-connector-mysql/mysql-binlog-connector-java-0.25.1.jar
debezium-connector-mysql/mysql-connector-java-8.0.21.jar
debezium-connector-mysql/debezium-connector-mysql-1.5.4.Final.jar
```  

### 카프카 컨테이너 플러그인 경로 수정
도커 컨테이너는 기본적으로 VI 에디터 설치가 안되어 있다. 플러그인 변경을 위해 VI 에디터를 설치한다.  
```shell
# apt-get update
# apt-get install vim
```  
  
/opt/kafka/config/connect-distributed.properties 파일을 수정하고 카프카 컨테이너를 재시작하면 변경한 플러그인 경로가 반영된다.  
```shell
// 변경전 
#plugin.path=

// 변경후
plugin.path=/opt/kafka_2.13-2.8.1/connectors
```  

## 5. 카프카 Connect 실행  
카프카 컨테이너 안에서 실행한다.
```shell
# connect-distributed.sh /opt/kafka/config/connect-distributed.properties
```  

## 6. Source Connector 생성  

### 카프카 Connect 클러스터 정보 조회  
```shell
# curl http://localhost:8083/
{"version":"2.8.1","commit":"839b886f9b732b15","kafka_cluster_id":"BZSpEGKFSfOHrUgI-Tl-6Q"}
```   

### MySQL 플러그인 확인  
지금은 Source 커넥터, Sink 커넥터 모두 확인이 가능하다. 
```shell
[
  {
    "class": "io.confluent.connect.jdbc.JdbcSinkConnector",
    "type": "sink",
    "version": "10.5.2"
  },
  {
    "class": "io.confluent.connect.jdbc.JdbcSourceConnector",
    "type": "source",
    "version": "10.5.2"
  },
  {
    "class": "io.debezium.connector.mysql.MySqlConnector",
    "type": "source",
    "version": "1.5.4.Final"
  },
  {
    "class": "org.apache.kafka.connect.file.FileStreamSinkConnector",
    "type": "sink",
    "version": "2.8.1"
  },
  {
    "class": "org.apache.kafka.connect.file.FileStreamSourceConnector",
    "type": "source",
    "version": "2.8.1"
  },
  {
    "class": "org.apache.kafka.connect.mirror.MirrorCheckpointConnector",
    "type": "source",
    "version": "1"
  },
  {
    "class": "org.apache.kafka.connect.mirror.MirrorHeartbeatConnector",
    "type": "source",
    "version": "1"
  },
  {
    "class": "org.apache.kafka.connect.mirror.MirrorSourceConnector",
    "type": "source",
    "version": "1"
  }
]
```  

### API Connector 생성  
```shell
# curl --location --request POST 'http://localhost:8083/connectors' --header 'Content-Type: application/json' --data-raw '{
  "name": "source-test-connector",
  "config": {
    "connector.class": "io.debezium.connector.mysql.MySqlConnector",
    "tasks.max": "1",
    "database.hostname": "mysql-source",
    "database.port": "3306",
    "database.user": "mysqluser",
    "database.password": "mysqlpw",
    "database.server.id": "184054",
    "database.server.name": "dbserver1",
    "database.allowPublicKeyRetrieval": "true",
    "database.include.list": "testdb",
    "database.history.kafka.bootstrap.servers": "kafka:9092",
    "database.history.kafka.topic": "dbhistory.testdb",
    "key.converter": "org.apache.kafka.connect.json.JsonConverter",
    "key.converter.schemas.enable": "true",
    "value.converter": "org.apache.kafka.connect.json.JsonConverter",
    "value.converter.schemas.enable": "true",
    "transforms": "unwrap,addTopicPrefix",
    "transforms.unwrap.type": "io.debezium.transforms.ExtractNewRecordState",
    "transforms.addTopicPrefix.type":"org.apache.kafka.connect.transforms.RegexRouter",
    "transforms.addTopicPrefix.regex":"(.*)",
    "transforms.addTopicPrefix.replacement":"$1"
  }
}'
```  

[설정 옵션 설명](https://aws.amazon.com/ko/blogs/korea/introducing-amazon-msk-connect-stream-data-to-and-from-your-apache-kafka-clusters-using-managed-connectors/)  

### API Connector 목록  
```shell
curl --location --request GET 'http://localhost:8083/connectors'
```  

### API Connector 상세
```shell
curl --location --request GET 'http://localhost:8083/connectors/source-test-connector/config ' --header 'Content-Type: application/json'
```  

### API Connector 삭제  
```shell
curl --location --request DELETE 'http://localhost:8083/connectors/source-test-connector'
```

## 7. Source Connect 테스트  
Source 데이터베이스의 테이블에 데이터 Insert 후 카프카 컨슈머에서 데이터 확인

### Source 테스트 데이터 Insert  
```mysql
INSERT INTO accounts VALUES ("1", "role1", "name1", "desc1", now());
INSERT INTO accounts VALUES ("2", "role2", "name2", "desc2", now());
INSERT INTO accounts VALUES ("3", "role3", "name3", "desc3", now());
```  

### 콘솔 카프카 컨슈머 확인  
```shell
# kafka-console-consumer.sh --topic dbserver1.testdb.accounts --bootstrap-server localhost:9092 --from-beginning

{"schema":{"type":"struct","fields":[{"type":"string","optional":false,"field":"account_id"},{"type":"string","optional":true,"field":"role_id"},{"type":"string","optional":true,"field":"user_name"},{"type":"string","optional":true,"field":"user_description"},{"type":"int64","optional":true,"name":"io.debezium.time.Timestamp","version":1,"default":0,"field":"update_date"}],"optional":false,"name":"dbserver1.testdb.accounts.Value"},"payload":{"account_id":"1","role_id":"role1","user_name":"name1","user_description":"desc1","update_date":1662970838000}}
{"schema":{"type":"struct","fields":[{"type":"string","optional":false,"field":"account_id"},{"type":"string","optional":true,"field":"role_id"},{"type":"string","optional":true,"field":"user_name"},{"type":"string","optional":true,"field":"user_description"},{"type":"int64","optional":true,"name":"io.debezium.time.Timestamp","version":1,"default":0,"field":"update_date"}],"optional":false,"name":"dbserver1.testdb.accounts.Value"},"payload":{"account_id":"2","role_id":"role2","user_name":"name2","user_description":"desc2","update_date":1662970839000}}
{"schema":{"type":"struct","fields":[{"type":"string","optional":false,"field":"account_id"},{"type":"string","optional":true,"field":"role_id"},{"type":"string","optional":true,"field":"user_name"},{"type":"string","optional":true,"field":"user_description"},{"type":"int64","optional":true,"name":"io.debezium.time.Timestamp","version":1,"default":0,"field":"update_date"}],"optional":false,"name":"dbserver1.testdb.accounts.Value"},"payload":{"account_id":"3","role_id":"role3","user_name":"name3","user_description":"desc3","update_date":1662970841000}}
```  
여기까지 Source Connector 환경 구성을 해보았다.  
```text
Source Connector: MySQL --> kafka Connect(Debezium Source Connector) --> Kafka  
```
8번 부터는 Sink Connector 관련 구성을 진행할 예정이다.  
```text
Sink Connector : Kafka --> kafka Connect(JDBC Sink Connector) --> MySQL
```  
  
  
  
## 8. Sink-MySQL 설정
위 1번 docker-compose.yml 파일에서 Sink MySQL 관련 컨테이너는 이미 생성이 되어있다. Sink용 데이터베이스 생성 및 테이블을 생성한다. Sink용 DB가 Source와 달라진 점은 데이터베이스 이름이 testdb에서 sinkdb로 변경된 것을 제외하고 모두 동일하다.    
### Database, Table 생성  
```shell
# mysql -u root -p
```  
```mysql
create database sinkdb;

use sinkdb;

CREATE TABLE accounts (
   account_id VARCHAR(255),
   role_id VARCHAR(255),
   user_name VARCHAR(255),
   user_description VARCHAR(255),
   update_date DATETIME DEFAULT CURRENT_TIMESTAMP,
   PRIMARY KEY (account_id)
);
```  

### 사용자 권한 확인  
컨테이너 생성시 docker-compose.yml 파일에서 생성한 mysqluser 파일에 권한을 부여한다.
```mysql
use mysql;

GRANT ALL PRIVILEGES ON *.* TO 'mysqluser'@'%';

FLUSH PRIVILEGES;
```  

## 9. Sink Connector 설치 - Kafka JDBC Connector  

### JDBC Connector 다운로드
[JDBC Connector 다운로드](https://www.confluent.io/hub/confluentinc/kafka-connect-jdbc) 경로에서 confluentinc-kafka-connect-jdbc-10.5.2.zip 파일을 다운로드 받는다.  

### 카프카 컨테이너 안으로 복사  
```shell
# docker cp confluentinc-kafka-connect-jdbc-10.5.2.zip kafka:/opt/kafka_2.13-2.8.1/connectors/
```  

### JDBC Connector 압축해제  
```shell
# cd /opt/kafka_2.13-2.8.1/connectors
# unzip confluentinc-kafka-connect-jdbc-10.5.2.zip
```  

### 카프카 Connect 재실행  
카프카 플러그인 경로 설정은 Source Connector 설정 시 했기때문에 신규로 추가된 Sink Connector 플러그인이 반영되도록 재시작
```shell
# connect-distributed.sh /opt/kafka/config/connect-distributed.properties
```  

## 10. Sink Connector 생성하기
### API Connetor 생성  
```shell
curl --location --request POST 'http://localhost:8083/connectors' \
--header 'Content-Type: application/json' \
--data-raw '{
  "name": "sink-test-connector",
  "config": {
    "connector.class": "io.confluent.connect.jdbc.JdbcSinkConnector",
    "tasks.max": "1",
    "connection.url": "jdbc:mysql://mysql-sink:3306/sinkdb?user=mysqluser&password=mysqlpw",
    "auto.create": "false",
    "auto.evolve": "false",
    "delete.enabled": "true",
    "insert.mode": "upsert",
    "pk.mode": "record_key",
    "table.name.format":"${topic}",
    "tombstones.on.delete": "true",
    "connection.user": "mysqluser",
    "connection.password": "mysqlpw",
    "topics.regex": "dbserver1.testdb.(.*)",
    "key.converter": "org.apache.kafka.connect.json.JsonConverter",
    "key.converter.schemas.enable": "true",
    "value.converter": "org.apache.kafka.connect.json.JsonConverter",
    "value.converter.schemas.enable": "true",
    "transforms": "unwrap, route, TimestampConverter",
    "transforms.unwrap.type": "io.debezium.transforms.ExtractNewRecordState",
    "transforms.unwrap.drop.tombstones": "true",
    "transforms.route.type": "org.apache.kafka.connect.transforms.RegexRouter",
    "transforms.route.regex": "([^.]+)\\.([^.]+)\\.([^.]+)",
    "transforms.route.replacement": "$3",
    "transforms.TimestampConverter.type": "org.apache.kafka.connect.transforms.TimestampConverter$Value",
    "transforms.TimestampConverter.format": "yyyy-MM-dd HH:mm:ss",
    "transforms.TimestampConverter.target.type": "Timestamp",
    "transforms.TimestampConverter.field": "update_date"
  }
}'
```   
옵션들에 대한 자세한 설명은 아래 참고 블로그에서 확인하도록 한다.  

### JDBC connector 생성 시 No suitable driver found 에러  
```shell
Caused by: org.apache.kafka.connect.errors.ConnectException: java.sql.SQLException: No suitable driver found for jdbc:mysql://localhost:3307/testdb
```  
[Connect/J JDBC driver for MySQL](https://mvnrepository.com/artifact/mysql/mysql-connector-java/8.0.27) 파일을 다운로드 받아 JDBC Connector 플러그인 하위의 lib 폴더에 넣어준다.  
```shell
# docker cp mysql-connector-java-8.0.27.jar kafka:/opt/kafka_2.13-2.8.1/connectors/confluentinc-kafka-connect-jdbc-10.5.2/lib/
```  

## 11. Sink Connect 테스트  

### Source 테스트 데이터 Insert  
```mysql
INSERT INTO accounts VALUES ("4", "role4", "name4", "desc4", now());
INSERT INTO accounts VALUES ("5", "role5", "name5", "desc5", now());
INSERT INTO accounts VALUES ("6", "role6", "name6", "desc6", now());
```  

### 콘솔 카프카 컨슈머 확인  
```shell
# kafka-console-consumer.sh --topic dbserver1.testdb.accounts --bootstrap-server localhost:9092 --from-beginning

{"schema":{"type":"struct","fields":[{"type":"string","optional":false,"field":"account_id"},{"type":"string","optional":true,"field":"role_id"},{"type":"string","optional":true,"field":"user_name"},{"type":"string","optional":true,"field":"user_description"},{"type":"int64","optional":true,"name":"io.debezium.time.Timestamp","version":1,"default":0,"field":"update_date"}],"optional":false,"name":"dbserver1.testdb.accounts.Value"},"payload":{"account_id":"4","role_id":"role4","user_name":"name4","user_description":"desc4","update_date":1662974998000}}
{"schema":{"type":"struct","fields":[{"type":"string","optional":false,"field":"account_id"},{"type":"string","optional":true,"field":"role_id"},{"type":"string","optional":true,"field":"user_name"},{"type":"string","optional":true,"field":"user_description"},{"type":"int64","optional":true,"name":"io.debezium.time.Timestamp","version":1,"default":0,"field":"update_date"}],"optional":false,"name":"dbserver1.testdb.accounts.Value"},"payload":{"account_id":"5","role_id":"role5","user_name":"name5","user_description":"desc5","update_date":1662974999000}}
{"schema":{"type":"struct","fields":[{"type":"string","optional":false,"field":"account_id"},{"type":"string","optional":true,"field":"role_id"},{"type":"string","optional":true,"field":"user_name"},{"type":"string","optional":true,"field":"user_description"},{"type":"int64","optional":true,"name":"io.debezium.time.Timestamp","version":1,"default":0,"field":"update_date"}],"optional":false,"name":"dbserver1.testdb.accounts.Value"},"payload":{"account_id":"6","role_id":"role6","user_name":"name6","user_description":"desc6","update_date":1662975000000}}
```  

### Sink 테이블 데이터 확인
```mysql
# select * from accouts   

account_id|role_id|user_name|user_description|update_date        |
----------+-------+---------+----------------+-------------------+
1         |role1  |name1    |desc1           |2022-09-12 08:20:38|
2         |role2  |name2    |desc2           |2022-09-12 08:20:39|
3         |role3  |name3    |desc3           |2022-09-12 08:20:41|
4         |role4  |name4    |desc4           |2022-09-12 09:29:58|
5         |role5  |name5    |desc5           |2022-09-12 09:29:59|
6         |role6  |name6    |desc6           |2022-09-12 09:30:00|
```  

Source에서 입력한 테스트 데이터를 Kafka Consumer와 Sink 테이블에서 조회되는 것을 확인했고 로컬 환경에서 Kafka CDC 환경구성을 마무리 하려고한다.    


## 참고  
[MySQL 에서 Kafka 로 Source Connector 구축하기](https://wecandev.tistory.com/m/109)  
[Kafka 에서 Mysql 로 Sink Connector 구축하기](https://wecandev.tistory.com/110)  
[JDBC connector 생성 시 No suitable driver found 에러 발생](https://wecandev.tistory.com/111)   
[Source Connector API 명령 참고1](https://docs.confluent.io/platform/current/connect/references/restapi.html)  
[Source Connector API 명령 참고2](https://developer.confluent.io/learn-kafka/kafka-connect/rest-api/)   
[Sink Connector API 명령 참고1](https://docs.confluent.io/kafka-connect-jdbc/current/sink-connector/sink_config_options.html)   
[Sink Connector API 토픽이름 RegrexRouter1](https://docs.confluent.io/platform/current/connect/transforms/regexrouter.html)   
[Sink Connector API 토픽이름 RegrexRouter2](https://debezium.io/blog/2017/09/25/streaming-to-another-database/)   

## Github  
<https://github.com/sisipapa/kafka-cdc>   
