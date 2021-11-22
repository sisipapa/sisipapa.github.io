---
layout: post
title: Springboot-JPA-API 개발1
category: [jpa]
tags: [jap, Springboot, api]
redirect_from:

- /2021/11/22/

---

현 재직중인 직장 퇴사가 확정이 되고 오늘부터 1주일간 개인적으로 활용할 수 있는 시간이 생기게 되었다. 쉬는 기간동안 이직하기로 한 직장에서 사용하게 될 기술 도메인들에 대해 다시 한번 정리를 해보려고 한다. 오늘부터 3일동안 [인프런 강의: 실전! 스프링 부트와 JPA 활용2 - API 개발과 성능 최적화](https://www.inflearn.com/course/%EC%8A%A4%ED%94%84%EB%A7%81%EB%B6%80%ED%8A%B8-JPA-API%EA%B0%9C%EB%B0%9C-%EC%84%B1%EB%8A%A5%EC%B5%9C%EC%A0%81%ED%99%94/) 강의를 보면서 따라해보고 내용을 정리해 보려고 한다.  

## 준비사항
강의에서 진행하는 환경과 동일하게 구성한다. 라이브러리 버전만 조금 다르고 구성은 동일하다.  

Springboot version : 2.5.6  
Java JDK : 1.11  

### build.gradle  
```json
buildscript {
    ext {
      queryDslVersion = "4.4.0"
    }
}

plugins {
    id 'org.springframework.boot' version '2.5.6'
    id 'io.spring.dependency-management' version '1.0.11.RELEASE'
    id 'java'
}

group = 'com.example'
version = '0.0.1-SNAPSHOT'
sourceCompatibility = '11'

configurations {
    compileOnly {
        extendsFrom annotationProcessor
    }
}

repositories {
    mavenCentral()
}

dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    implementation 'org.springframework.boot:spring-boot-starter-web'
    compileOnly 'org.projectlombok:lombok'
    annotationProcessor 'org.projectlombok:lombok'
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
    
    runtimeOnly 'com.h2database:h2'
    implementation 'org.springframework.boot:spring-boot-starter-validation'
    implementation 'com.fasterxml.jackson.datatype:jackson-datatype-hibernate5'
    runtimeOnly('org.springframework.boot:spring-boot-devtools')

    // QueryDSL
    implementation "com.querydsl:querydsl-jpa:${queryDslVersion}"
    annotationProcessor(
        "javax.persistence:javax.persistence-api",
        "javax.annotation:javax.annotation-api",
        "com.querydsl:querydsl-apt:${queryDslVersion}:jpa")
}

// QueryDSL
sourceSets {
    main {
        java {
            srcDirs = ["$projectDir/src/main/java", "$projectDir/build/generated"]
        }
    }
}

test {
    useJUnitPlatform()
}
```  

### application.yaml  
```yaml
spring:
  h2:
    console:
      enabled: true
      path: /h2-console
  datasource:
    url: jdbc:h2:~/jpashop
    driver-class-name: org.h2.Driver
    username: sa
    password:

  jpa:
    hibernate:
      ddl-auto: create
    properties:
      hibernate:
#        show_sql: true
        format_sql: true
#        default_batch_fetch_size: 1000 #최적화 옵션

logging.level:
  org.hibernate.SQL: debug
#  org.hibernate.type: trace
```

## 참고  
[실전! 스프링 부트와 JPA 활용2 - API 개발과 성능 최적화](https://www.inflearn.com/course/%EC%8A%A4%ED%94%84%EB%A7%81%EB%B6%80%ED%8A%B8-JPA-API%EA%B0%9C%EB%B0%9C-%EC%84%B1%EB%8A%A5%EC%B5%9C%EC%A0%81%ED%99%94/)  

## Github  
<https://github.com/sisipapa/inflearn-jpa-rest-api.git>  



