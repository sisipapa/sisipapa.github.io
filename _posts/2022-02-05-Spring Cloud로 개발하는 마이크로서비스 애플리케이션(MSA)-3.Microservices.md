---
layout: post
title: Spring Cloud로 개발하는 마이크로서비스 애플리케이션(MSA)-3.Microservices
category: [msa]
tags: [springboot, msa]
redirect_from:

- /2022/02/06/

---

Spring Cloud로 개발하는 마이크로서비스 애플리케이션(MSA) 강의를 보면서 정리하는 노트.  

## 강의에서 진행 할 애플리케이션 구성요소  
|구성요소|설명|
|---|---|
|Git Repository|마이크로서비스 소스 관리 및 프로파일 관리|
|Config Server|Git 저장소에 등록된 프로파일 정보 및 서비스 라우팅|
|Eureka Server|마이크로서비스 등록 및 검색|
|API Gateway Server|마이크로서비스 부하 분산 및 서비스 라우팅|
|Microservices|회원 MS, 주문 MS, 상품(카테고리) MS|
|Queuing System|마이크로서비스 간 메시지 발행 및 구독|  

## 강의에서 진행 할 애플리케이션 API 목록
|마이크로서비스|Restful API|HTTP Method|
|---|---|---|
|Catalog Service|/catalog-service/catalogs : 상품 목록 제공|GET|
|User Service|/user-service/users : 사용자 정보 등록|POST|
|User Service|/user-service/users : 전체 사용자 조회|GET|
|User Service|/user-service/users/{user_id} : 사용자 정보, 주문 내역 조회|GET|
|Order Service|/order-service/users/{user_id}/orders : 주문 등록|POST|
|Order Service|/order-service/users/{user_id}/orders : 주문 확인|GET|




## Github
<https://github.com/sisipapa/spring-cloud-inflearn.git>  




