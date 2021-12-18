---
layout: post
title: MySQL Replication(feat.docker)
category: [docker]
tags: [docker, mysql, replication]
redirect_from:

- /2021/12/18/

---

## Docker-Compose 로 Mysql Replication(Master/Slave) 환경 구축
회사의 업무를 위해 DB Master,Slave 환경을 구축해서 JPA Connection 테스트를 진행해 보려고 한다. 이번 정리에서는 Docker Compose를 활용해서 Mysql Master,Slave 환경을 구축해 보려고 한다.  

## 디렉토리 구성  
<img src="https://sisipapa.github.io/assets/images/posts/mysql-master-slave-project.PNG" >  

### Dockerfile(master,slave)  
docker-compose에서 build할 Dockerfile이다.
```dockerfile
FROM mysql:5.7
ADD ./master/my.cnf /etc/mysql/my.cnf
```  

### my.cnf(master)  
```properties
[mysqld]
log_bin = mysql-bin
server_id = 10
default_authentication_plugin=mysql_native_password
```  

### my.cnf(slave)  
```properties
[mysqld]
log_bin = mysql-bin
server_id = 11
relay_log = /var/lib/mysql/mysql-relay-bin
log_slave_updates = 1
read_only = 1
default_authentication_plugin=mysql_native_password
```  

### docker-compose.yml   
master port - 3306  
slave port - 3307  

```yaml
version: "3"
services:
  db-master:
    build: 
      context: ./
      dockerfile: master/Dockerfile
    restart: always
    environment:
      MYSQL_DATABASE: 'db'
      MYSQL_USER: 'user'
      MYSQL_PASSWORD: 'password'
      MYSQL_ROOT_PASSWORD: 'password'
    ports:
      - '3306:3306'
    # Where our data will be persisted
    volumes:
      - my-db-master:/var/lib/mysql
      - my-db-master:/var/lib/mysql-files
    networks:
      - net-mysql
  
  db-slave:
    build: 
      context: ./
      dockerfile: slave/Dockerfile
    restart: always
    environment:
      MYSQL_DATABASE: 'db'
      MYSQL_USER: 'user'
      MYSQL_PASSWORD: 'password'
      MYSQL_ROOT_PASSWORD: 'password'
    ports:
      - '3307:3306'
    # Where our data will be persisted
    volumes:
      - my-db-slave:/var/lib/mysql
      - my-db-slave:/var/lib/mysql-files
    networks:
      - net-mysql
  
# Names our volume
volumes:
  my-db-master:
  my-db-slave: 

networks: 
  net-mysql:
    driver: bridge
```  

## 컨테이너 생성

### docker-compose up -d 명령어 실행
```shell
$ docker-compose up -d
Building db-master
[+] Building 2.8s (8/8) FINISHED
 => [internal] load build definition from Dockerfile                                                                                                                                                                               0.0s
 => => transferring dockerfile: 92B                                                                                                                                                                                                0.0s
 => [internal] load .dockerignore                                                                                                                                                                                                  0.0s
 => => transferring context: 2B                                                                                                                                                                                                    0.0s
 => [internal] load metadata for docker.io/library/mysql:5.7                                                                                                                                                                       2.7s
 => [auth] library/mysql:pull token for registry-1.docker.io                                                                                                                                                                       0.0s
 => [internal] load build context                                                                                                                                                                                                  0.0s
 => => transferring context: 170B                                                                                                                                                                                                  0.0s
 => [1/2] FROM docker.io/library/mysql:5.7@sha256:d1cc87a3bd5dc07defc837bc9084f748a130606ff41923f46dec1986e0dc828d                                                                                                                 0.0s
 => CACHED [2/2] ADD ./master/my.cnf /etc/mysql/my.cnf                                                                                                                                                                             0.0s
 => exporting to image                                                                                                                                                                                                             0.0s
 => => exporting layers                                                                                                                                                                                                            0.0s
 => => writing image sha256:62b37d9c32c6d996e435511f44c8694aff3919c710d4b8555cec178389ec4e1c                                                                                                                                       0.0s
 => => naming to docker.io/library/mysql-master-slave_db-master                                                                                                                                                                    0.0s

Use 'docker scan' to run Snyk tests against images to find vulnerabilities and learn how to fix them
WARNING: Image for service db-master was built because it did not already exist. To rebuild this image you must use `docker-compose build` or `docker-compose up --build`.
Building db-slave
[+] Building 0.7s (7/7) FINISHED
 => [internal] load build definition from Dockerfile                                                                                                                                                                               0.0s
 => => transferring dockerfile: 91B                                                                                                                                                                                                0.0s
 => [internal] load .dockerignore                                                                                                                                                                                                  0.0s
 => => transferring context: 2B                                                                                                                                                                                                    0.0s
 => [internal] load metadata for docker.io/library/mysql:5.7                                                                                                                                                                       0.5s
 => [internal] load build context                                                                                                                                                                                                  0.0s
 => => transferring context: 252B                                                                                                                                                                                                  0.0s
 => [1/2] FROM docker.io/library/mysql:5.7@sha256:d1cc87a3bd5dc07defc837bc9084f748a130606ff41923f46dec1986e0dc828d                                                                                                                 0.0s
 => CACHED [2/2] ADD ./slave/my.cnf /etc/mysql/my.cnf                                                                                                                                                                              0.0s
 => exporting to image                                                                                                                                                                                                             0.0s
 => => exporting layers                                                                                                                                                                                                            0.0s
 => => writing image sha256:56662e16e2451931b3061c0220e0dc9198448ab2868be53ead73fa9c217945b8                                                                                                                                       0.0s
 => => naming to docker.io/library/mysql-master-slave_db-slave 
```  

### master slave 통신을 위한 내부 IP 주소를 확인
```shell
$ docker network ls
NETWORK ID     NAME                           DRIVER    SCOPE
f5bef6daac9a   bridge                         bridge    local
88a870f218f7   docker-elk_elk                 bridge    local
38a49601eb10   host                           host      local
8512717ba3fb   mysql-master-slave_net-mysql   bridge    local
09e62c8e1a63   none                           null      local
```  

docker inspect를 통해 Containers 하위에 master IPv4Address 프로퍼티의 값을 확인한다.   
```shell
$ docker inspect 8512717ba3fb
[
    {
        "Name": "mysql-master-slave_net-mysql",
        "Id": "8512717ba3fb50e8df8dc6621d18d07902c5e1ff951175e12674055412bd68ea",
        "Created": "2021-12-17T15:43:05.7419187Z",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": null,
            "Config": [
                {
                    "Subnet": "172.18.0.0/16",
                    "Gateway": "172.18.0.1"
                }
            ]
        },
        "Internal": false,
        "Attachable": true,
        "Ingress": false,
        "ConfigFrom": {
            "Network": ""
        },
        "ConfigOnly": false,
        "Containers": {
            "0a19ee35212e53b5408e9443cc87dc5e95060deb7f904ec54178724dc0b3cd9b": {
                "Name": "mysql-master-slave_db-master_1",
                "EndpointID": "57f634d0bcef77ad5983a4bfa78ce9526ddd9fe2e66df1b1c879d7dce5fae49d",
                "MacAddress": "02:42:ac:12:00:02",
                "IPv4Address": "172.18.0.2/16",
                "IPv6Address": ""
            },
            "2ef34d9e20746bfe78cc09eabf5edb8367afed9a6ee2be91cd5066cf64ab02ed": {
                "Name": "mysql-master-slave_db-slave_1",
                "EndpointID": "bcbea40cb4374db791f6d9f9bdd1a1112589376663e656a198a0a26d5c53a720",
                "MacAddress": "02:42:ac:12:00:03",
                "IPv4Address": "172.18.0.3/16",
                "IPv6Address": ""
            }
        },
        "Options": {},
        "Labels": {
            "com.docker.compose.network": "net-mysql",
            "com.docker.compose.project": "mysql-master-slave",
            "com.docker.compose.version": "1.29.2"
        }
    }
]
```  

### slave에서 master 연결정보 설정  
위 docker inspect 명령어로 확인한 master의 host정보를 MASTER_HOST 속성에 설정하고 아래 명령어 실행을 했을 때 아래와 같은 오류가 나왔다.  
```mysql
mysql> CHANGE MASTER TO MASTER_HOST='172.18.0.2', MASTER_USER='root', MASTER_PASSWORD='password', MASTER_LOG_FILE='mysql-bin.000013', MASTER_LOG_POS=0;

mysql> start slave;

mysql> show slave status\G;
*************************** 1. row ***************************
               Slave_IO_State:
                  Master_Host: 172.18.0.2
                  Master_User: root
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000013
          Read_Master_Log_Pos: 4
               Relay_Log_File: mysql-relay-bin.000008
                Relay_Log_Pos: 4
        Relay_Master_Log_File: mysql-bin.000013
             Slave_IO_Running: No
            Slave_SQL_Running: Yes
              Replicate_Do_DB:
          Replicate_Ignore_DB:
           Replicate_Do_Table:
       Replicate_Ignore_Table:
      Replicate_Wild_Do_Table:
  Replicate_Wild_Ignore_Table:
                   Last_Errno: 0
                   Last_Error:
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 4
              Relay_Log_Space: 154
              Until_Condition: None
               Until_Log_File:
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File:
           Master_SSL_CA_Path:
              Master_SSL_Cert:
            Master_SSL_Cipher:
               Master_SSL_Key:
        Seconds_Behind_Master: NULL
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 1236
                Last_IO_Error: Got fatal error 1236 from master when reading data from binary log: 'Could not find first log file name in binary log index file'
               Last_SQL_Errno: 0
               Last_SQL_Error:
  Replicate_Ignore_Server_Ids:
             Master_Server_Id: 10
                  Master_UUID: 4f5b2ae7-5f40-11ec-8372-0242ac170003
             Master_Info_File: /var/lib/mysql/master.info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: Slave has read all relay log; waiting for more updates
           Master_Retry_Count: 86400
                  Master_Bind:
      Last_IO_Error_Timestamp: 211218 00:05:47
     Last_SQL_Error_Timestamp:
               Master_SSL_Crl:
           Master_SSL_Crlpath:
           Retrieved_Gtid_Set:
            Executed_Gtid_Set:
                Auto_Position: 0
         Replicate_Rewrite_DB:
                 Channel_Name:
           Master_TLS_Version:
1 row in set (0.00 sec)

ERROR:
No query specified
```  

### master에 접속해서 status 정보 확인  
```mysql
mysql> SHOW MASTER STATUS;
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000001 |      154 |              |                  |                   |
+------------------+----------+--------------+------------------+-------------------+
1 row in set (0.00 sec)

```  

### master에서 확인한 file, position 정보로 slave 연결 정보 변경  
아래 두옵션이 Yes인지 확인  
Slave_IO_Running: Yes  
Slave_SQL_Running: Yes  


```mysql
mysql> STOP SLAVE;
Query OK, 0 rows affected (0.00 sec)

mysql> RESET SLAVE;
Query OK, 0 rows affected (0.02 sec)

mysql> CHANGE MASTER TO MASTER_HOST='172.18.0.2', MASTER_USER='root', MASTER_PASSWORD='password', MASTER_LOG_FILE='mysql-bin.000001', MASTER_LOG_POS=154;
Query OK, 0 rows affected, 2 warnings (0.03 sec)

mysql> START SLAVE;
Query OK, 0 rows affected (0.01 sec)

mysql> show slave status\G
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 172.18.0.2
                  Master_User: root
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000001
          Read_Master_Log_Pos: 154
               Relay_Log_File: mysql-relay-bin.000002
                Relay_Log_Pos: 320
        Relay_Master_Log_File: mysql-bin.000001
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
              Replicate_Do_DB:
          Replicate_Ignore_DB:
           Replicate_Do_Table:
       Replicate_Ignore_Table:
      Replicate_Wild_Do_Table:
  Replicate_Wild_Ignore_Table:
                   Last_Errno: 0
                   Last_Error:
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 154
              Relay_Log_Space: 527
              Until_Condition: None
               Until_Log_File:
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File:
           Master_SSL_CA_Path:
              Master_SSL_Cert:
            Master_SSL_Cipher:
               Master_SSL_Key:
        Seconds_Behind_Master: 0
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error:
               Last_SQL_Errno: 0
               Last_SQL_Error:
  Replicate_Ignore_Server_Ids:
             Master_Server_Id: 10
                  Master_UUID: 4f5b2ae7-5f40-11ec-8372-0242ac170003
             Master_Info_File: /var/lib/mysql/master.info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: Slave has read all relay log; waiting for more updates
           Master_Retry_Count: 86400
                  Master_Bind:
      Last_IO_Error_Timestamp:
     Last_SQL_Error_Timestamp:
               Master_SSL_Crl:
           Master_SSL_Crlpath:
           Retrieved_Gtid_Set:
            Executed_Gtid_Set:
                Auto_Position: 0
         Replicate_Rewrite_DB:
                 Channel_Name:
           Master_TLS_Version:
1 row in set (0.00 sec)
```  

### 오류 확인
ERROR 3021 (HY000): This operation cannot be performed with a running slave io thread; run STOP SLAVE IO_THREAD FOR CHANNEL ” first.  
```mysql
mysql> STOP SLAVE IO_THREAD;
Query OK, 0 rows affected (0.00 sec)
mysql> change master to master_host='192.168.0.99',master_port=3307,master_user='copy',master_password='copy',master_log_file='mysql-bin.000001',master_log_pos=154;
Query OK, 0 rows affected, 2 warnings (0.00 sec)
mysql> START SLAVE IO_THREAD;
Query OK, 0 rows affected (0.00 sec)
```  

### Master Slave Replication 테스트
테스트는 master DB에 접속해서 데이터베이스를 생성 후 slave접속해서 master에서 생성한 database가 보이는지 확인한다.
```shell
# master에 접속해서 test_db 데이터베이스 생성
$ mysql -u root -p
Enter password:
mysql> create database test_db;  

# slave에 접속해서 test_db 데이터베이스 조회
$ mysql -u root -p
Enter password:
mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| db                 |
| mysql              |
| performance_schema |
| sys                |
| test_db            |
+--------------------+
6 rows in set (0.00 sec)
```  

여기까지 mysql master slave 연동설정은 마치고 다음에는 jpa multi connection을 진행할 예정이다.

## 참고  
[MySql - Master Slave Replication 구조 만들어보기](https://huisam.tistory.com/entry/mysql-replication)  
[Mysql replication "Last_IO_Errno: 1236 Last_IO_Error" 에러 해결](http://115.68.22.172/index.php?mid=board_KaGH35&document_srl=292)
[DebugAH - ERROR 3021 (HY000)](https://debugah.com/error-3021-hy000-this-operation-cannot-be-performed-with-a-running-slave-io-thread-run-stop-slave-io_thread-for-channel-first-23784/)




