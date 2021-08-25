---
layout: post
title: Springboot MSA 구성3 - Spring Cloud Eureka
category: [msa]
tags: [msa, springboot, eureka]
redirect_from:

- /2021/08/25/

---

Spring Cloud Eureka는 MSA 시스템에서 서비스의 로드밸런싱과 실패처리 등을 유연하게 하기 위해 각 서비스들의 IP|PORT|InstanceId를 가지고 있는 REST 기반의 미들웨어 서버이다. Eureka는 마이크로 서비스 기반의 아키텍처의 핵심 원칙 중 하나인 Service Discovery의 역할을 수행한다. MSA에서는 Service의 IP와 Port가 일정하지 않고 지속적을 변화한다. Eureka는 Client-Server의 방식으로 Eureka Server는 모든 Client 서버들이 본인의 IP와 Port, InstanceId를 Eureka-Server로 전달한다. 그리고 Eureka에 있는 정보를 Fetch하여 Eureka-Client간 통신에 사용한다.  

## Eureka 서버

### build.gradle
공통으로 사용하는 Dependency는 전체소스를 내려받아서 확인하면 되고 여기서는 Eureka 모듈에서 사용하는 Dependency만을 정리한다.  
```properties
project(':Eureka') {
    dependencies {
        implementation 'org.springframework.cloud:spring-cloud-starter-netflix-eureka-server'
    }

    dependencyManagement {
        imports {
            mavenBom "org.springframework.cloud:spring-cloud-dependencies:${springCloudVersion}"
        }
    }
}
```  

### Application.java  
EnableEurekaServer 어노테이션을 설정한다.  
```java
@EnableEurekaServer
@SpringBootApplication
public class EurekaApplication {

    public static void main(String[] args) {
        SpringApplication.run(EurekaApplication.class, args);
    }

}
```  

### bootstrap.yml  
```yaml
server:
  port: 8761
spring:
  application:
    name: eureka
eureka:
  client:
    registerWithEureka: true
    fetchRegistry: false
```  

Eureka의 설정정보를 Config 서버에서 가져오도록 Config서버에 EurekaClient 설정을 하게 되면 Config서버와 Eureka 서버간의 Dependency가 생기게 되기 떄문에 설정을 하지는 않는다.  




## 참고
[DaddyProgrammer Spring CLoud MSA(3) - Service Discovery by Eureka](https://daddyprogrammer.org/post/4446/spring-cloud-msa-service-discovery-by-eureka/)  

## Github
<https://github.com/sisipapa/Springboot-MSA.git>