---
layout: post
title: Spring Cloud로 개발하는 마이크로서비스 애플리케이션(MSA)-3.Microservices
category: [msa]
tags: [springboot, msa]
redirect_from:

- /2022/02/06/

---

인프런 강의 - [Spring Cloud로 개발하는 마이크로서비스 애플리케이션(MSA)](https://www.inflearn.com/course/%EC%8A%A4%ED%94%84%EB%A7%81-%ED%81%B4%EB%9D%BC%EC%9A%B0%EB%93%9C-%EB%A7%88%EC%9D%B4%ED%81%AC%EB%A1%9C%EC%84%9C%EB%B9%84%EC%8A%A4/dashboard) 강의를 보면서 정리하는 노트입니다.  

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
|제목 셀1|제목 셀2|제목 셀3|제목 셀4|
|---|---|---|---|
|내용 1|내용 2|내용 3|내용 4|
|내용 5|내용 6|내용 7|내용 8|
|내용 9|내용 10|내용 11|내용 12|


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




