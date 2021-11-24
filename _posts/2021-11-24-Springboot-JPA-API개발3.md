---
layout: post
title: Springboot-JPA-API 개발3
category: [jpa]
tags: [jap, Springboot, api]
redirect_from:

- /2021/11/24/

---

오늘은 API 개발 고급 - 컬렉션 조회 최적화 강의를 보면서 내용을 정리해 보려고 한다.   
주문내역에서 추가로 주문한 상품 정보를 Order 기준으로 컬렉션인 OrderItem 와 Item 이 필요하다. 이전 정리에서는 xToOne(OneToOne, ManyToOne) 관계만 있었다. 이번에는 컬렉션인 일대다 관계(OneToMany)를 조회하고, 최적화하는 방법을 단계별로 정리해 볼 예정이다.  


## 참고  
[실전! 스프링 부트와 JPA 활용2 - API 개발과 성능 최적화](https://www.inflearn.com/course/%EC%8A%A4%ED%94%84%EB%A7%81%EB%B6%80%ED%8A%B8-JPA-API%EA%B0%9C%EB%B0%9C-%EC%84%B1%EB%8A%A5%EC%B5%9C%EC%A0%81%ED%99%94/)  

## Github  
<https://github.com/sisipapa/inflearn-jpa-rest-api.git>  



