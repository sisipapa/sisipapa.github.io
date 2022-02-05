---
layout: post
title: Spring Cloud로 개발하는 마이크로서비스 애플리케이션(MSA)-API Gateway Service
category: [msa]
tags: [springboot, msa, Spring Cloud Gateway]
redirect_from:

- /2022/02/05/

---

Spring Cloud Netflix Zuul 은 Spring Boot 2.4에서 Maintenance 상태이다. 오늘은 Spring Cloud Gateway에 대한 정리만 할 예정이다.  

## Spring Cloud Gateway  

### pom.xml  
프로젝트 생성 시 Spring Cloud Routing > gateway(spring-cloud-starter-gateway)를 선택한다.   
spring-cloud-starter-netflix-eureka-client, lombok 은 나중 작업을 위해 미리 추가한 라이브러리이다.  
```xml
 <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-gateway</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        </dependency>

        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>
```


## Github
<https://github.com/sisipapa/spring-cloud-inflearn.git>  




