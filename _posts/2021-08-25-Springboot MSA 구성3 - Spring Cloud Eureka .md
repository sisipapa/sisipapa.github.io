---
layout: post
title: Springboot MSA 구성3 - Spring Cloud Eureka
category: [msa]
tags: [msa, springboot, eureka]
redirect_from:

- /2021/08/25/

---

Spring Cloud Eureka는 MSA 시스템에서 서비스의 로드밸런싱과 실패처리 등을 유연하게 하기 위해 각 서비스들의 IP,PORT,InstanceId를 가지고 있는 REST 기반의 미들웨어 서버이다. Eureka는 마이크로 서비스 기반의 아키텍처의 핵심 원칙 중 하나인 Service Discovery의 역할을 수행한다. MSA에서는 Service의 IP와 Port가 일정하지 않고 지속적을 변화한다. Eureka는 Client-Server의 방식으로 Eureka Server는 모든 Client 서버들이 본인의 IP와 Port, InstanceId를 Eureka-Server로 전달한다. 그리고 Eureka에 있는 정보를 Fetch하여 Eureka-Client간 통신에 사용한다.    

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

### Eureka 대시보드 확인   
Eureka 서버의 기본설정은 여기까지이고 -Dspring.profiles.active=local 파라미터를 주고 서버를 구동하고 대시보드에 접속한다.     
Eureka 대시보드 접속 - [http://localhost:8761]   
<img src="https://sisipapa.github.io/assets/images/posts/eureka-board.PNG" >  

## Eureka Discovery에 클라이언트 서비스 등록  
gateway, resource, resource2 모듈에 eureka client 설정을 추가한다.  

### build.gradle  
gateway, resource, reource2 모듈에 spring-cloud-starter-netflix-eureka-client dependency를 추가한다.
```properties
project(':Gateway') {
    dependencies {
        implementation 'org.springframework.cloud:spring-cloud-starter-netflix-zuul'
        implementation 'org.springframework.cloud:spring-cloud-starter-config'
        implementation 'org.springframework.cloud:spring-cloud-starter-netflix-eureka-client'
    }

    dependencyManagement {
        imports {
            mavenBom "org.springframework.cloud:spring-cloud-dependencies:${springCloudVersion}"
        }
    }
}

project(':Resource') {
    dependencies {
        implementation 'org.springframework.boot:spring-boot-starter-web'
        implementation 'org.springframework.cloud:spring-cloud-starter-config'
        implementation 'org.springframework.boot:spring-boot-starter-actuator'
        implementation 'org.springframework.cloud:spring-cloud-starter-netflix-eureka-client'
    }
    
    dependencyManagement {
        imports {
            mavenBom "org.springframework.cloud:spring-cloud-dependencies:${springCloudVersion}"
        }
    }
}

project(':Resource2') {
    dependencies {
        implementation 'org.springframework.boot:spring-boot-starter-web'
        implementation 'org.springframework.cloud:spring-cloud-starter-config'
        implementation 'org.springframework.boot:spring-boot-starter-actuator'
        implementation 'org.springframework.cloud:spring-cloud-starter-netflix-eureka-client'
    }

    dependencyManagement {
        imports {
            mavenBom "org.springframework.cloud:spring-cloud-dependencies:${springCloudVersion}"
        }
    }
}
```  

### Application.jva 설정
#### resource    
```java
@EnableDiscoveryClient
@SpringBootApplication
public class ResourceApplication {
    public static void main(String[] args) {
        SpringApplication.run(ResourceApplication.class, args);
    }
}
```  

#### resource2  
```java
@EnableDiscoveryClient
@SpringBootApplication
public class Resource2Application {
    public static void main(String[] args) {
        SpringApplication.run(Resource2Application.class, args);
    }
}
```  

#### gateway  
```java
@EnableDiscoveryClient
@EnableZuulProxy
@SpringBootApplication
public class GatewayApplication {
    public static void main(String[] args) {
        SpringApplication.run(GatewayApplication.class, args);
    }
}
```  

### Eureka 대시보드 확인  
gateway, resource, resource2 모듈을 재기동하고 대시보드를 확인한다.  
Eureka 대시보드 접속 - [http://localhost:8761]
<img src="https://sisipapa.github.io/assets/images/posts/eureka-board2.PNG" >  

## Eureka Discovery 활용1  
config-repo 모듈의 gateway 관련 설정 중 zuul관련설정을 변경한다.  
```yaml
# 변경전
zuul:
  routes:
    member:
      stripPrefix: false
      path: /v1/member/**
      url: http://localhost:8080
      serviceId: resource
    pay:
      stripPrefix: false
      path: /v1/pay/**
      url: http://localhost:8081
      serviceId: resource2
    else:
      stripPrefix: false
      path: /v1/**
      url: http://localhost:8081
      serviceId: resource2  

---

#변경후  
zuul:
  routes:
    member:
      stripPrefix: false
      path: /v1/member/**
      serviceId: resource
    pay:
      stripPrefix: false
      path: /v1/pay/**
      serviceId: resource2
    else:
      stripPrefix: false
      path: /v1/**
      serviceId: resource2
```  
재배포 없이 실시간 반영을 위한 gateway 서버의 /actuator/refresh API를 호출한다.  
```json
POST http://localhost:9100/actuator/refresh

HTTP/1.1 200 
Content-Type: application/json
Transfer-Encoding: chunked
Date: Thu, 26 Aug 2021 06:07:53 GMT
Keep-Alive: timeout=60
Connection: keep-alive

[
  "config.client.version",
  "zuul.routes.member.url",
  "zuul.routes.else.url",
  "zuul.routes.pay.url"
]

Response code: 200; Time: 7755ms; Content length: 95 bytes
```  
IP,PORT 접속정보 제거후 serviceId 접속테스트  
```json
GET http://localhost:9100/v1/member/health

HTTP/1.1 200 
Date: Thu, 26 Aug 2021 06:08:43 GMT
Keep-Alive: timeout=60
Content-Type: application/json
Transfer-Encoding: chunked
Connection: keep-alive

MemberController running

Response code: 200; Time: 1087ms; Content length: 24 bytes
```  

## Eureka Discovery 활용2
Config 서버의 정보도 Eureka에 등록하고 활용하는 테스트를 진행해 보겠다.  
### build.gradle   
Config 모듈에도 spring-cloud-starter-netflix-eureka-client Dependency를 추가한다.  
```properties
project(':Config') {
    dependencies {
        implementation 'org.springframework.cloud:spring-cloud-config-server'
        implementation 'org.springframework.cloud:spring-cloud-starter-netflix-eureka-client'
    }

    dependencyManagement {
        imports {
            mavenBom "org.springframework.cloud:spring-cloud-dependencies:${springCloudVersion}"
        }
    }
}
```  

### Applicaiton.java 설정  
```java
@EnableDiscoveryClient
@EnableConfigServer
@SpringBootApplication
public class ConfigApplication {

    public static void main(String[] args) {
        SpringApplication.run(ConfigApplication.class, args);
    }

}
```  

### 모듈의 config 서버정보 config.uri > config.discovery.service.id로 변경
```yaml
#변경전 
spring:
  application:
    name: resource
  cloud:
    config:
      uri: http://localhost:9000

---

#변경후
spring:
  application:
    name: resource
  cloud:
    config:
      fail-fast: true
      discovery:
        service-id: config
        enabled: true
    # uri: http://localhost:9000
```  

### Eureka 대시보드 확인  
Eureka 대시보드에서 Config 서버 등록 확인.  
<img src="https://sisipapa.github.io/assets/images/posts/beureka-oard3.PNG" >  

### config 서버 service명으로 접속 확인
resource, reource2 서버의 config서버의 설정 파일에 설정 값을 읽어오는 API 호출 시 정상 응답 확인.
```json
GET http://localhost:9100/v1/resource2/message

HTTP/1.1 200 
Date: Thu, 26 Aug 2021 07:30:23 GMT
Keep-Alive: timeout=60
Content-Type: application/json
Transfer-Encoding: chunked
Connection: keep-alive

Resource2Controller message : Resource2 Service Local Server!!!!!

Response code: 200; Time: 470ms; Content length: 65 bytes
```    

## Eureka Discovery 활용3  
Resource2 서버의 요청이 많아서 서버의 증설이 필요한 상황을 가정해서 Service 모듈을 추가하는 과정을 정리해 보려고 한다.

### 실행 가능한 jar파일 생성  
IntelliJ의 Gradle bootJar를 실행해서 실행가능한 jar파일을 생성한다.  
<img src="https://sisipapa.github.io/assets/images/posts/eureka-bootjar.PNG" >  

### jar 파일 실행
```shell
$ java -jar -Dserver.port=8082 -Dspring.profiles.active=local C:\projects\Springboot-MSA\Resource2\build\libs\Resource2-0.01-SNAPSHOT.jar
```  

### Eureka 대시보드 확인  
8082 포트로 실행한 Resource2 서비스 추가확인
<img src="https://sisipapa.github.io/assets/images/posts/eureka-board4.PNG" >  

### 테스트  
2번의 요청을 날렸는데 8081, 8082로 한번씩 로드밸런싱 되어 요청을 한다.  
```json
GET http://localhost:9100/v1/resource2/message

HTTP/1.1 200 
Date: Thu, 26 Aug 2021 09:18:16 GMT
Keep-Alive: timeout=60
Content-Type: application/json
Transfer-Encoding: chunked
Connection: keep-alive

Resource2Controller message : Resource2 Service Local Server!!!!! | port: 8082

Response code: 200; Time: 317ms; Content length: 79 bytes
        
        
GET http://localhost:9100/v1/resource2/message

HTTP/1.1 200
Date: Thu, 26 Aug 2021 09:19:27 GMT
Keep-Alive: timeout=60
Content-Type: application/json
Transfer-Encoding: chunked
Connection: keep-alive

Resource2Controller message : Resource2 Service Local Server!!!!! | port: 8081

Response code: 200; Time: 411ms; Content length: 79 bytes
```  

## 참고
[DaddyProgrammer Spring CLoud MSA(3) - Service Discovery by Eureka](https://daddyprogrammer.org/post/4446/spring-cloud-msa-service-discovery-by-eureka/)  

## Github
<https://github.com/sisipapa/Springboot-MSA.git>