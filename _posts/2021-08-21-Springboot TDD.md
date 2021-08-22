---
layout: post 
title: Springboot TDD 구성
category: [tdd]
tags: [tdd, springboot]
redirect_from

- /2021/08/21/

---  


## TDD란?  
TDD란 Test Driven Development의 약자로 '테스트 주도 개발'이라고 한다. 반복 테스트를 이용한 소프트웨어 방법론으로, 작은 단위의 테스트 케이스를 작성하고 이를 통과하는 코드를 추가하는 단계를 반복하여 구현한다.    
  

## 나의 TDD    
나는 사실 항상 TDD에 맞게 개발을 하고 있지는 못하다. 하지만 시간과 여건이 허락된다면 TDD 방법대로 개발을 해보려고 한다. 내가 TDD을 활용해 개발하는 방식을 정리해 보려고 한다.    
정리는 Member의 등록,조회,삭제 기능 개발을 한다는 전제로 진행할 예정이다. Springboot2.5.4, JPA, QueryDSL, H2, Gradle, Junit5 환경에서 Rest API를 Member의 등록,조회,삭제 API를 만들어 보려고 한다.  

### PreSetting
build.grale  
build.gradle의 dependency 내용이다.  
```text
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    implementation 'org.springframework.boot:spring-boot-starter-web'
    compileOnly 'org.projectlombok:lombok'
    annotationProcessor 'org.projectlombok:lombok'
    testImplementation 'org.springframework.boot:spring-boot-starter-test'

    runtimeOnly  'com.h2database:h2'                                        // H2 DB설정
}
```  

application.properties  
DB 접속정보 및 hibernate.ddl-auto 설정 추가  
```properties
spring.h2.console.enabled=true
spring.h2.console.path=/h2

spring.jpa.show_sql = true
spring.datasource.url=jdbc:h2:tcp://localhost/~/test;
spring.datasource.driverClassName=org.h2.Driver
spring.datasource.username=sa
spring.datasource.password=

spring.jpa.hibernate.ddl-auto=create
```



나의 경우 Repository,Service,Controller 순서로 단위테스트를 진행한다. 실제 업무에서 DB 또는 Mongo에 비지니스 비중이 크다면 Repository 단위테스트만 만들어서 진행하고 Service 단계에서 업무 비지니스가 많다면 Service 단위테스트만 만들어서 진행을 했다. 지금처럼 Repository,Service,Controller 모든 구간별로 단위테스트를 진행해 본적은 없는 것 같아 정리를 한번 해보았다. TDD 정말 좋은 개발 방법론인데 내가 내 업무에 제대로 녹여서 사용하고 있지 못한것 같다. 더 나은 내일을 위해 정리를 한다. 이상........ 

## 참고  
[Taes-k DevLog - TDD Spring 실무에서 적용하기](https://taes-k.github.io/2021/03/19/spring-tdd-practice/)  
[찰나의 개발흔적 - [springboot] 연락처 관리 프로젝트(TDD)](https://daddyprogrammer.org/post/4347/spring-cloud-msa-configuration-server/)  

## Github  
<https://github.com/sisipapa/Springboot-TDD.git>