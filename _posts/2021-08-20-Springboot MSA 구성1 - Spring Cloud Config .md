---
layout: post
title: Springboot MSA 구성1 - Spring Cloud Config
category: [tdd]
tags: [msa, springboot, config]
redirect_from:

- /2021/08/20/

---

Spring Cloud Config는 MSA 시스템의 환경 설정들을 중앙화해서 한 곳에서 관리를 할 수 있게 해주고 설정파일의 변경되어도 어플리케이션의 재배포 없이 적용이 가능하다.  
진행할 프로젝트 구성은 아래와 같이 진행할 예정이다.  
1. Spring Cloud Config 서버  
2. Spring Cloud Zuul(API Gateway) 서버  
3. Spring Cloud Eureka 서버  

1~3번까지 완료 후 시간이 된다면 인증서버까지 적용해 볼 예정이다. 오늘은 MSA 프로젝트의 첫번째 Spring Cloud Config 서버를 구성해 보려고 한다. 그리고 Config 서버에서 데이터를 설정파일을 잘 읽어오는지 확인을 위한 Resource 서버 2대를 같이 구성할 예정이다.  

## PreSetting  
프로젝트는 Intellij의 이전에 정리한 [노트](https://sisipapa.github.io/blog/2021/08/16/Intellij-Springboot-Multiple-Module(ver.-Gradle)/)를 참고해서 Multi Module 프로젝트를 구성했다.  
<img src="https://sisipapa.github.io/assets/images/posts/msa-config0.PNG" >  
- Config : Spring Cloud Config 서버  
- config-repo : Spring Cloud Config 서버가 바라보는 설정파일  
- Eureka : Spring Cloud Eureka 서버  
- Gateway : Spring Cloud Zuul(API Gateway) 서버  
- Resource : Application 서버
- Resource2 : Application 서버  

## Config 서버  
### build.gradle
```properties
# 공통 Dependency
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter'
    
    compileOnly 'org.projectlombok:lombok'
    annotationProcessor 'org.projectlombok:lombok'
    testImplementation('org.springframework.boot:spring-boot-starter-test') {
        exclude group: 'org.junit.vintage', module: 'junit-vintage-engine'
    }
}

# Config Dependency
project(':Config') {
    dependencies {
        implementation 'org.springframework.cloud:spring-cloud-config-server'
    }
    
    dependencyManagement {
        imports {
            mavenBom "org.springframework.cloud:spring-cloud-dependencies:${springCloudVersion}"
        }
    }
}
```  

### Application.java  
EnableConfigServer 어노테이션 추가
```java
@EnableConfigServer
@SpringBootApplication
public class ConfigApplication {

    public static void main(String[] args) {
        SpringApplication.run(ConfigApplication.class, args);
    }

}
```  

### application-{env}.yml  
#### local
```yaml
server:
  port: 9000
spring:
  application:
    name: configServer
  cloud:
    config:
      server:
        git:
          uri: https://github.com/sisipapa/Springboot-MSA
          search-paths: config-repo/local
```
#### prod  
```yaml
server:
  port: 9000
spring:
  application:
    name: configServer
  cloud:
    config:
      server:
        git:
          uri: https://github.com/sisipapa/Springboot-MSA
          search-paths: config-repo/prod
```  

### config-repo 
config-repo 모듈에 application 별 active.profiles 별로 설정파일을 만든다. 여기서는 message의 내용만 조금 다르게 설정했다.
#### Resource/local 
```yaml
spring:
  profiles: local
  message: Resource Service Local Server!!!!!
```  

#### Resource2/local  
```yaml
spring:
  profiles: local
  message: Resource2 Service Local Server!!!!!
```

여기까지만 하면 Spring Cloud Config 서버의 설정은 끝이다. 로컬서버 구동시 -Dspring.profiles.active=local로 설정 후 서버를 기동한다.  

### Config 서버 테스트
```http request
GET http://localhost:9000/resource/local

HTTP/1.1 200 
Content-Type: application/json
Transfer-Encoding: chunked
Date: Tue, 24 Aug 2021 09:31:28 GMT
Keep-Alive: timeout=60
Connection: keep-alive

{
  "name": "resource",
  "profiles": [
    "local"
  ],
  "label": null,
  "version": "5eb3f2ee05568a9878370171ccfc4a88af79a71c",
  "state": null,
  "propertySources": [
    {
      "name": "https://github.com/sisipapa/Springboot-MSA/file:C:\\Users\\user\\AppData\\Local\\Temp\\config-repo-17387822938375408351\\config-repo\\local\\resource-local.yml",
      "source": {
        "spring.profiles": "local",
        "spring.message": "Resource Service Local Server!!!!!"
      }
    }
  ]
}

Response code: 200; Time: 3933ms; Content length: 401 bytes  


GET http://localhost:9000/resource2/local

HTTP/1.1 200 
Content-Type: application/json
Transfer-Encoding: chunked
Date: Tue, 24 Aug 2021 09:32:15 GMT
Keep-Alive: timeout=60
Connection: keep-alive

{
  "name": "resource2",
  "profiles": [
    "local"
  ],
  "label": null,
  "version": "5eb3f2ee05568a9878370171ccfc4a88af79a71c",
  "state": null,
  "propertySources": [
    {
      "name": "https://github.com/sisipapa/Springboot-MSA/file:C:\\Users\\user\\AppData\\Local\\Temp\\config-repo-17387822938375408351\\config-repo\\local\\resource2-local.yml",
      "source": {
        "spring.profiles": "local",
        "spring.message": "Resource Service Local Server!!!!!"
      }
    }
  ]
}

Response code: 200; Time: 822ms; Content length: 403 bytes
```  

## Resource 서버  
### application-{env}.yml
```yaml
server:
  port: 8080
spring:
  application:
    name: resource
  config:
    import: "optional:configserver:http://localhost:9000"
management:
  endpoints:
    web:
      exposure:
        include: info, refresh
```  

### Controller  
GIT 에 저장된 property 값이 변경 된 경우, client에서 최신 파일을 다시 받아 값을 refresh 하기 위해서 RefreshScope 어노테이션을 추가해준다.  
```java
@RestController
@RefreshScope
public class ResourceController {

    @Value("${spring.message}")
    private String message;

    @GetMapping("/message")
    public String message() {
        return "ResourceController message : " + message;
    }
}
```  

### Resource 서버에서 Config 조회  
```http request
GET http://localhost:8080/message

HTTP/1.1 200 
Content-Type: application/json
Content-Length: 63
Date: Tue, 24 Aug 2021 09:42:19 GMT
Keep-Alive: timeout=60
Connection: keep-alive

ResourceController message : Resource Service Local Server!!!!!

Response code: 200; Time: 327ms; Content length: 63 bytes
```  

Resource 서버의 [GET]/message API 호출 결과로 Config property를 정상적으로 읽어 온 것을 확인했다. 이제 config-repo의 설정파일의 내용을 수정하고 동일한 API를 호출하면 변경된 결과가 나오지 않는다. 변경된 설정을 Resource 서버에서도 적용을 하기 위해서는 [POST]/actuator/refresh API를 호출해야 변경된 설정이 적용된다. API 결과로 변경된 property 목록이 리턴된다.     
```http request
POST http://localhost:8080/actuator/refresh

HTTP/1.1 200 
Content-Type: application/json
Transfer-Encoding: chunked
Date: Tue, 24 Aug 2021 09:47:06 GMT
Keep-Alive: timeout=60
Connection: keep-alive

[
  "config.client.version",
  "spring.message"
]

Response code: 200; Time: 1540ms; Content length: 42 bytes
```  



## 참고
[DaddyProgrammer Spring CLoud MSA](https://daddyprogrammer.org/post/4347/spring-cloud-msa-configuration-server/)  
[Jorten [Spring] Cloud Config 구축하기](https://goateedev.tistory.com/167)  
[Spring Cloud Config 에서 변경된 정보를 마이크로서비스 인스턴스에서 Spring Boot Actuator 를 이용하여 반영하기](https://wonit.tistory.com/505)  

## Github
<https://github.com/sisipapa/Springboot-MSA.git>