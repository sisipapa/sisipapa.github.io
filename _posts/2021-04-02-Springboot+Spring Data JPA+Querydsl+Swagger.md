---
layout: post
title: Springboot+Spring Data JPA+Querydsl+Swagger
category: [Springboot]
tags: [JPA,QueryDSL,Swagger,RestAPI]
redirect_from:
  - /2021/04/02/
---

- **Springboot+Spring Data JPA+Querydsl+Swagger Rest API **  

개발 진행순서는 아래와 같다.
1. H2 DB설치
2. Springboot 프로젝트 생성(Sprinboot + Spring Data JPA 연동)
3. QueryDSL 연동
4. Swagger(Springdoc-openapi 연동)

# 1. H2 DB설치  
- H2 database 다운로드 후 설치  
  http://www.h2database.com/html/main.html 페이지에서 OS맞는 설치파일을 다운로드 후 설치한다.
- 접속 확인
  http://localhost:8082/login.jsp?jsessionid=3ca85e5c51fb926f441f097c1a6d0434  
- JDBC URL을 TCP 모드로 세팅후 연결
  jdbc:h2:tcp://localhost/~/DB명

<img src="https://sisipapa.github.io/assets/images/posts/2021-04-02-h2.PNG" width="300" height="200"> 


  
