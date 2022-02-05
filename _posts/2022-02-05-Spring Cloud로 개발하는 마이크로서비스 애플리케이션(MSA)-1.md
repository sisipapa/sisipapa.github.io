---
layout: post
title: Spring Cloud로 개발하는 마이크로서비스 애플리케이션(MSA)-1
category: [msa]
tags: [springboot, msa]
redirect_from:

- /2022/02/05/

---

며칠전 MSA 관련 인프런 강의를 보기 시작했다. 강의를 보고 따라해 보면서 진행했던 내용들 중 중요하거나 다시 확인이 필요한 부분을 정리할 예정이다.  

## Service Discovery
Springboot version - 2.6.3  
JDK Version - 11  
Spring cloud version - 2021.0.0  

### pom.xml
프로젝트 생성시 Spring Cloud Discovery > Eureka Server 만 선택 후 프로젝트를 생성한다.
```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```  

### ~Application.java  
@EnableEurekaServer 어노테이션을 추가한다.
```java
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.netflix.eureka.server.EnableEurekaServer;


@SpringBootApplication
@EnableEurekaServer
public class DiscoveryserviceApplication {

    public static void main(String[] args) {
        SpringApplication.run(DiscoveryserviceApplication.class, args);
    }

}
```  

### application.yml
```yaml
server:
  port: 8761

spring:
  application:
    name: discoveryservice

eureka:
  client:
    register-with-eureka: false
    fetch-registry: false
```
### Eureka 실행화면  
<img src="https://sisipapa.github.io/assets/images/posts/eureka-dashboard.png" >     

## Github
<https://github.com/sisipapa/spring-cloud-inflearn.git>  




