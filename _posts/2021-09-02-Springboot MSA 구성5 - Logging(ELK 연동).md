---
layout: post
title: Springboot MSA 구성5 - Logging(ELK 연동)
category: [msa]
tags: [msa, springboot, elk]
redirect_from:

- /2021/09/02/

---

Docker-Compose를 이용해서 로컬PC에 ELK Stack 환경구성이 완료되었다. 이번에는 기존에 진행했던 Springboot MSA 프로젝트에 ELK Stack을 붙여 중앙 집중식 로깅 환경을 구성해 보려고 한다. 

## PreSetting  
기존 Springboot-MSA 프로젝트에는 logging 관련 설정이 안되어 있기 때문에 logback 설정과 적재된 로그를 logstash로 보낼때 필요한 인코딩 라이브러리 logstash-logback-encoder Dependency 추가가 필요하다.  
### logback.xml  
FileAppender 설정없이 Console과 Logstash 관련 설정만 추가.
```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <!--콘솔에 로그를 남깁니다.-->
    <appender name="console" class="ch.qos.logback.core.ConsoleAppender">
        <encoder class="ch.qos.logback.classic.encoder.PatternLayoutEncoder">
            <pattern>%-5level %d{HH:mm:ss.SSS} [%thread] %logger{36} - %msg%n</pattern>
        </encoder>
    </appender>

    <appender name="stash" class="net.logstash.logback.appender.LogstashTcpSocketAppender">
        <destination>127.0.0.1:5000</destination>

        <!-- encoder is required -->
        <encoder class="net.logstash.logback.encoder.LogstashEncoder" />
    </appender>

    <root level="INFO">
        <appender-ref ref="console"/>
        <appender-ref ref="stash"/>
    </root>

</configuration>
```  

### build.gradle
공통 Dependency에 logstash-logback-encoder Dependency 추가
```properties
dependencies {
    implementation group: 'net.logstash.logback', name: 'logstash-logback-encoder', version: '6.6' 
}
```  

### Controller info레벨 로그추가
```java
// Resource 모듈 info 로그 추가
@GetMapping("/db")
public String db() {
    log.info("ResourceController db : " + ip + ":" + port + " [" + id +" : " + password + "]");
    return "ResourceController db : " + ip + ":" + port + " [" + id +" : " + password + "]";
}

// Resource2 모듈 info 로그 추가
@GetMapping("/health")
public String health() {
    log.info("PayController running");
    return "PayController running";
}
```  

### 로그 추가한 API 호출
Reource 모듈 API - http://localhost:9100/v1/resource/db  
```shell
INFO  08:51:08.607 [http-nio-8080-exec-9] c.s.s.m.r.c.ResourceController - ResourceController db : 10.10.10.10:3333 [dbid : dbpass]
INFO  08:51:09.411 [http-nio-8080-exec-10] c.s.s.m.r.c.ResourceController - ResourceController db : 10.10.10.10:3333 [dbid : dbpass]
INFO  08:51:10.227 [http-nio-8080-exec-5] c.s.s.m.r.c.ResourceController - ResourceController db : 10.10.10.10:3333 [dbid : dbpass]
INFO  08:51:11.856 [http-nio-8080-exec-4] c.s.s.m.r.c.ResourceController - ResourceController db : 10.10.10.10:3333 [dbid : dbpass]
INFO  08:51:12.638 [http-nio-8080-exec-2] c.s.s.m.r.c.ResourceController - ResourceController db : 10.10.10.10:3333 [dbid : dbpass]
```  

Reource2 모듈 API - http://localhost:9100/v1/pay/health
```shell
INFO  08:51:14.138 [http-nio-8090-exec-3] c.s.s.m.r.controller.PayController - PayController running
INFO  08:51:14.945 [http-nio-8090-exec-2] c.s.s.m.r.controller.PayController - PayController running
INFO  08:51:15.720 [http-nio-8090-exec-1] c.s.s.m.r.controller.PayController - PayController running
INFO  08:51:16.516 [http-nio-8090-exec-10] c.s.s.m.r.controller.PayController - PayController running
INFO  08:51:17.260 [http-nio-8090-exec-5] c.s.s.m.r.controller.PayController - PayController running
```  
logback 설정에 console,stash에 로그가 적재되도록 설정을 했고 위에서 Console 로그는 확인되었다. 이제 Kibana에서 로그가 정상적으로 출력되었는지 확인해 보자.



## Kibana 설정/확인  
### Index 생성을 위해 Manage 클릭  
<img src="https://sisipapa.github.io/assets/images/posts/kibana-index1.PNG" >   


## 참고  
[MSA 와 Log - 중앙 집중식 로깅 ELK stack 편](https://bravenamme.github.io/2021/01/28/elk-stack/)  
[Elasticsearch + Logstash + Kibana 구축하기 (2) - Spring boot와 연동](https://investment-engineer.tistory.com/m/5)  

## Github    
<https://github.com/sisipapa/Springboot-MSA.git>  
