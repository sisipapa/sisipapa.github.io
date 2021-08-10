---
layout: post 
title: Springboot Querydsl정리
category: [querydsl]
tags: [querydsl, springboot]
redirect_from:

- /2021/08/10/

---

현재 운영중인 시스템에서는 JPA, Querydsl을 사용하지 않고 있다. 언제 사용하게 될 지 모르지만 언제든 사용할 수 있게 Querydsl 사용과 관련된 정리를 해보려고 한다. [lelecoder님 블로그](https://lelecoder.com/145)의 내용이 정리가 잘 되어 있어 참고해서 진행할 예정이다. 

## PreSetting  
### docker mysql 설치
```shell
docker run -d --name test_mysql -p 3306:3306 -e MYSQL_ROOT_PASSWORD=rootpwd mysql:5.7 --character-set-server=utf8mb4 --collation-server=utf8mb4_unicode_ci
```  


### docker mysql 접속 및 database 생성
```shell
C:\Users\user>docker ps -a
CONTAINER ID   IMAGE       COMMAND                  CREATED          STATUS          PORTS                                                  NAMES
5b591d0d3972   mysql:5.7   "docker-entrypoint.s…"   15 seconds ago   Up 13 seconds   0.0.0.0:3306->3306/tcp, :::3306->3306/tcp, 33060/tcp   test_mysql

C:\Users\user>docker exec -ti 5b591d0d3972 bash
root@5b591d0d3972:/# mysql -u root -p
Enter password:
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 2
Server version: 5.7.35 MySQL Community Server (GPL)

Copyright (c) 2000, 2021, Oracle and/or its affiliates.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
4 rows in set (0.00 sec)

mysql> create database testdb;
Query OK, 1 row affected (0.00 sec)
```  



## 참고
[스프링 데이터 JPA와 Querydsl 인프런 강의 정리 - lelecoder](https://lelecoder.com/145)

## Github
<https://github.com/sisipapa/study-Springboot-querydsl.git>   
