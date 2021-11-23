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

## API 기본 - MemberApiController  
아래는 강의를 들으면서 정리한 소스이다. v1 API는 Entity를 파라미터로 받아서 처리하는 경우이고 v2 API는 Entity가 아닌 별도의 DTO를 생성해서 처리한 경우이다.  

### 등록(POST)  
```java
@PostMapping("/api/v1/members")
public CreateMemberResponse saveMemberV1(@RequestBody @Valid Member member){
    Long id = memberService.join(member);
    return new CreateMemberResponse(id);
}

@PostMapping("/api/v2/members")
public CreateMemberResponse saveMemberV2(@RequestBody @Valid CreateMemberRequest request){
    Member member = new Member();
    member.setName(request.getName());
    
    Long id = memberService.join(member);
    return new CreateMemberResponse(id);
}
```  
v1 API처럼 Entity를 Request Body에 직접 매핑의 문제점
- Entity에 화면 로직이 추가된다.
- Entity에 API Validation 로직을 적용하는데 제약이 생긴다.
- Entity가 변경되면 API 스펙도 변경된다.

#### 이러한 이유로 API 요청 스펙에 맞게 별도의 DTO를 만들어 사용해야 한다.  

### 조회(GET)  
```java
@GetMapping("/api/v1/members")
public List<Member> memberV1(){
    return memberService.findMembers();
}

@GetMapping("/api/v2/members")
public Result memberV2(){
    List<Member> findMembers = memberService.findMembers();
    List<MemberDto> collect = findMembers.stream()
    .map( member -> new MemberDto(member.getName()))
    .collect(Collectors.toList());
    
    return new Result(collect);
}
```  
v1 API처럼 Entity를 Request Body에 직접 외부에 노출했을 때의 문제점
- Entity에 화면 로직이 추가된다.  
- Entity의 모든 값이 노출된다.    
- 응답 스펙에 맞게 가공이 필요하다.  
- 실무에서는 같은 엔티티에 대해 API가 용도에 따라 다양하게 만들어지는데, 한 엔티티에 각각의 API를 위한 프레젠테이션 응답 로직을 담기는 어렵다.  
- Entity가 변경되면 API 스펙도 변경된다.  

#### 이러한 이유로 API 요청 스펙에 맞게 별도의 DTO를 만들어 사용해야 한다.

### 수정(PUT)  
회원 수정 API updateMemberV2 은 회원 정보를 부분 업데이트 한다. 여기서 PUT 방식을 사용했는데, PUT은 전체 업데이트를 할 때 사용하는 것이 맞다. 부분 업데이트를 하려면 PATCH를 사용하거나 POST를 사용하는 것이 REST 스타일에 맞다.    
```java
@PutMapping("/api/v2/members/{id}")
public UpdateMemberResponse updateMemberV2(@PathVariable("id") Long id,
                                           @RequestBody @Valid UpdateMemberRequest request){
    memberService.update(id, request.getName());
    Member findMember = memberService.findOne(id);
    return new UpdateMemberResponse(findMember.getId(), findMember.getName());
}
```

다음 정리는 xToOne 관계의 성능최적화에 대해서 정리를 할 예정이다.  

## 참고  
[실전! 스프링 부트와 JPA 활용2 - API 개발과 성능 최적화](https://www.inflearn.com/course/%EC%8A%A4%ED%94%84%EB%A7%81%EB%B6%80%ED%8A%B8-JPA-API%EA%B0%9C%EB%B0%9C-%EC%84%B1%EB%8A%A5%EC%B5%9C%EC%A0%81%ED%99%94/)  

## Github  
<https://github.com/sisipapa/inflearn-jpa-rest-api.git>  



