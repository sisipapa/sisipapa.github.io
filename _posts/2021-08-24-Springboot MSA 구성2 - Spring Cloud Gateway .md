---
layout: post
title: Springboot MSA 구성2 - Spring Cloud Gateway
category: [msa]
tags: [msa, springboot, gateway]
redirect_from:

- /2021/08/24/

---

Spring Cloud Gateway는 API로 라우팅할 수 있는 간단하면서도 효과적인 방법을 제공하고 보안, 모니터링/메트릭스, 복원력 등과 같은 다양한 우려 사항을 제공하는 것을 목표로 한다. 서비스 엔드포인트를 하나로 통일해서 요청의 특성별로 알맞는 서비스로 라우팅 할 수 있는 기능을 제공한다. 라우팅 설정은 무중단 적용이 가능하다. 클라이언트는 엔드포인트가 한군데로 통일되어 관리포인트가 줄어드는 장점이 있다.  

## API Gateway의 역할
프록시의 역할과 로드밸런싱 - URI에 따라 서비스 엔드포인트를 다르게 가져가는 동적 라우팅이 가능하다.  
인증 서버로서의 기능 - 모든 요청/응답을 관리할 수가 있어 앞단에 인증 및 보안을 적용하기가 용이하다.  
로깅 서버로서의 기능 - 모든 요청/응답을 관제할 수 있는 모니터링 시스템 구성이 단순해진다.  

## Config서버에서 관리하는 config-repo 모듈에 Gateway Route 설정 추가  

### Gateway Route 설정추가  
```yaml
# gateway-local.yml
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

# gateway-prod.yml
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
```   

### Config 서버확인  
```json
GET http://localhost:9000/gateway/local

HTTP/1.1 200 
Content-Type: application/json
Transfer-Encoding: chunked
Date: Wed, 25 Aug 2021 03:39:41 GMT
Keep-Alive: timeout=60
Connection: keep-alive

{
  "name": "gateway",
  "profiles": [
    "local"
  ],
  "label": null,
  "version": "46e8eb3c8d007a6aef573aa6c815950d0a56fd3b",
  "state": null,
  "propertySources": [
    {
      "name": "https://github.com/sisipapa/Springboot-MSA/file:C:\\Users\\user\\AppData\\Local\\Temp\\config-repo-10700435317011410158\\config-repo\\local\\gateway-local.yml",
      "source": {
        "spring.profiles": "local",
        "zuul.routes.member.stripPrefix": false,
        "zuul.routes.member.path": "/v1/member/**",
        "zuul.routes.member.url": "http://localhost:8080",
        "zuul.routes.member.serviceId": "resource",
        "zuul.routes.pay.stripPrefix": false,
        "zuul.routes.pay.path": "/v1/pay/**",
        "zuul.routes.pay.url": "http://localhost:8081",
        "zuul.routes.pay.serviceId": "resource2",
        "zuul.routes.else.stripPrefix": false,
        "zuul.routes.else.path": "/v1/**",
        "zuul.routes.else.url": "http://localhost:8081",
        "zuul.routes.else.serviceId": "resource2"
      }
    }
  ]
}

Response code: 200; Time: 1120ms; Content length: 843 bytes
```  
## Gateway 모듈 구성  
Gateway 서버의 경우 클라이언트들의 엔드포인트로 트래픽이 높기 때문에 단일 서비스로 운영하기 보다는 여러대로 구성하고 LoadBalancer로 묶어 HA를 확보하는 것이 좋다.  
### build.gradle  
Multi Module 프로젝트 구성의 build.gradle의 Gateway 모듈에 해당하는 부분만 정리. spring-cloud-starter-netflix-zuul dependency 추가한다. gradle에서 zuul 라이브러리를 내려받지 못해 2.2.9.RELEASE 버전을 입력해 주었다.   
```properties
project(':Gateway') {
    dependencies {
        implementation 'org.springframework.cloud:spring-cloud-starter-netflix-zuul:2.2.9.RELEASE'
        implementation 'org.springframework.cloud:spring-cloud-starter-config'
    }

    dependencyManagement {
        imports {
            mavenBom "org.springframework.cloud:spring-cloud-dependencies:${springCloudVersion}"
        }
    }
}
```  

### Gateway 모듈 application-{env}.yml
local, prod 내용은 동일하다. 
```yaml
# application-local.yml
server:
  port: 9100
spring:
  application:
    name: gateway
  config:
    import: "optional:configserver:http://localhost:9000"
management:
  endpoints:
    web:
      exposure:
        include: refresh

---

# application-local.yml
server:
  port: 9100
spring:
  application:
    name: gateway
  config:
    import: "optional:configserver:http://localhost:9000"
management:
  endpoints:
    web:
      exposure:
        include: refresh
```  

### Application.java  
EnableZuulProxy 어노테이션을 추가한다.  
```java
@EnableZuulProxy
@SpringBootApplication
public class GatewayApplication {

    public static void main(String[] args) {
        SpringApplication.run(GatewayApplication.class, args);
    }

}
```  

### Resource 서버에서 확인을 위한 Gateway Routing 확인을 위한 Controller 추가
#### Resource 서버(8080) 추가된 Controller  
```java
@RequestMapping("/v1")
@RestController
@RefreshScope
public class MemberController {

    @GetMapping("/member/health")
    public String memberHealth() {
        return "MemberController running";
    }
    
    @GetMapping("/member2/health")
    public String member2Health() {
        return "MemberController running";
    }

}
```  
#### Resource2 서버(8081) 추가된 Controller  
PayController
```java
@RequestMapping("/v1/pay")
@RestController
@RefreshScope
public class PayController {
    @GetMapping("/health")
    public String health() {
        return "PayController running";
    }
}
```  
ProductController
```java
@RequestMapping("/v1")
@RestController
@RefreshScope
public class ProductController {
    @GetMapping("/product/health")
    public String productHealth() {
        return "PayController running";
    }
    @GetMapping("/product2/health")
    public String product2Health() {
        return "PayController running";
    }
}
```  

## Gateway Route Test 테스트  
### 장애발생!!!!!
```shell
java.lang.NoSuchMethodError: org.springframework.boot.web.servlet.error.ErrorController.getErrorPath()Ljava/lang/String;
	at org.springframework.cloud.netflix.zuul.web.ZuulHandlerMapping.lookupHandler(ZuulHandlerMapping.java:87) ~[spring-cloud-netflix-zuul-2.2.9.RELEASE.jar:2.2.9.RELEASE]
	at org.springframework.web.servlet.handler.AbstractUrlHandlerMapping.getHandlerInternal(AbstractUrlHandlerMapping.java:152) ~[spring-webmvc-5.3.9.jar:5.3.9]
	at org.springframework.web.servlet.handler.AbstractHandlerMapping.getHandler(AbstractHandlerMapping.java:498) ~[spring-webmvc-5.3.9.jar:5.3.9]
	at org.springframework.web.servlet.DispatcherServlet.getHandler(DispatcherServlet.java:1258) ~[spring-webmvc-5.3.9.jar:5.3.9]
	at org.springframework.web.servlet.DispatcherServlet.doDispatch(DispatcherServlet.java:1040) ~[spring-webmvc-5.3.9.jar:5.3.9]
	at org.springframework.web.servlet.DispatcherServlet.doService(DispatcherServlet.java:963) ~[spring-webmvc-5.3.9.jar:5.3.9]
	at org.springframework.web.servlet.FrameworkServlet.processRequest(FrameworkServlet.java:1006) ~[spring-webmvc-5.3.9.jar:5.3.9]
	at org.springframework.web.servlet.FrameworkServlet.doGet(FrameworkServlet.java:898) ~[spring-webmvc-5.3.9.jar:5.3.9]
	at javax.servlet.http.HttpServlet.service(HttpServlet.java:655) ~[tomcat-embed-core-9.0.52.jar:4.0.FR]
	at org.springframework.web.servlet.FrameworkServlet.service(FrameworkServlet.java:883) ~[spring-webmvc-5.3.9.jar:5.3.9]
	at javax.servlet.http.HttpServlet.service(HttpServlet.java:764) ~[tomcat-embed-core-9.0.52.jar:4.0.FR]
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:227) ~[tomcat-embed-core-9.0.52.jar:9.0.52]
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:162) ~[tomcat-embed-core-9.0.52.jar:9.0.52]
	at org.apache.tomcat.websocket.server.WsFilter.doFilter(WsFilter.java:53) ~[tomcat-embed-websocket-9.0.52.jar:9.0.52]
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:189) ~[tomcat-embed-core-9.0.52.jar:9.0.52]
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:162) ~[tomcat-embed-core-9.0.52.jar:9.0.52]
	at org.springframework.web.filter.RequestContextFilter.doFilterInternal(RequestContextFilter.java:100) ~[spring-web-5.3.9.jar:5.3.9]
	at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:119) ~[spring-web-5.3.9.jar:5.3.9]
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:189) ~[tomcat-embed-core-9.0.52.jar:9.0.52]
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:162) ~[tomcat-embed-core-9.0.52.jar:9.0.52]
	at org.springframework.web.filter.FormContentFilter.doFilterInternal(FormContentFilter.java:93) ~[spring-web-5.3.9.jar:5.3.9]
	at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:119) ~[spring-web-5.3.9.jar:5.3.9]
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:189) ~[tomcat-embed-core-9.0.52.jar:9.0.52]
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:162) ~[tomcat-embed-core-9.0.52.jar:9.0.52]
	at org.springframework.boot.actuate.metrics.web.servlet.WebMvcMetricsFilter.doFilterInternal(WebMvcMetricsFilter.java:96) ~[spring-boot-actuator-2.5.4.jar:2.5.4]
	at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:119) ~[spring-web-5.3.9.jar:5.3.9]
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:189) ~[tomcat-embed-core-9.0.52.jar:9.0.52]
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:162) ~[tomcat-embed-core-9.0.52.jar:9.0.52]
	at org.springframework.web.filter.CharacterEncodingFilter.doFilterInternal(CharacterEncodingFilter.java:201) ~[spring-web-5.3.9.jar:5.3.9]
	at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:119) ~[spring-web-5.3.9.jar:5.3.9]
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:189) ~[tomcat-embed-core-9.0.52.jar:9.0.52]
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:162) ~[tomcat-embed-core-9.0.52.jar:9.0.52]
	at org.apache.catalina.core.StandardWrapperValve.invoke(StandardWrapperValve.java:197) ~[tomcat-embed-core-9.0.52.jar:9.0.52]
	at org.apache.catalina.core.StandardContextValve.invoke(StandardContextValve.java:97) ~[tomcat-embed-core-9.0.52.jar:9.0.52]
	at org.apache.catalina.authenticator.AuthenticatorBase.invoke(AuthenticatorBase.java:542) ~[tomcat-embed-core-9.0.52.jar:9.0.52]
	at org.apache.catalina.core.StandardHostValve.invoke(StandardHostValve.java:135) ~[tomcat-embed-core-9.0.52.jar:9.0.52]
	at org.apache.catalina.valves.ErrorReportValve.invoke(ErrorReportValve.java:92) ~[tomcat-embed-core-9.0.52.jar:9.0.52]
	at org.apache.catalina.core.StandardEngineValve.invoke(StandardEngineValve.java:78) ~[tomcat-embed-core-9.0.52.jar:9.0.52]
	at org.apache.catalina.connector.CoyoteAdapter.service(CoyoteAdapter.java:357) ~[tomcat-embed-core-9.0.52.jar:9.0.52]
	at org.apache.coyote.http11.Http11Processor.service(Http11Processor.java:382) ~[tomcat-embed-core-9.0.52.jar:9.0.52]
	at org.apache.coyote.AbstractProcessorLight.process(AbstractProcessorLight.java:65) ~[tomcat-embed-core-9.0.52.jar:9.0.52]
	at org.apache.coyote.AbstractProtocol$ConnectionHandler.process(AbstractProtocol.java:893) ~[tomcat-embed-core-9.0.52.jar:9.0.52]
	at org.apache.tomcat.util.net.NioEndpoint$SocketProcessor.doRun(NioEndpoint.java:1726) ~[tomcat-embed-core-9.0.52.jar:9.0.52]
	at org.apache.tomcat.util.net.SocketProcessorBase.run(SocketProcessorBase.java:49) ~[tomcat-embed-core-9.0.52.jar:9.0.52]
	at org.apache.tomcat.util.threads.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1191) ~[tomcat-embed-core-9.0.52.jar:9.0.52]
	at org.apache.tomcat.util.threads.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:659) ~[tomcat-embed-core-9.0.52.jar:9.0.52]
	at org.apache.tomcat.util.threads.TaskThread$WrappingRunnable.run(TaskThread.java:61) ~[tomcat-embed-core-9.0.52.jar:9.0.52]
	at java.base/java.lang.Thread.run(Thread.java:834) ~[na:na]
```  
spring-boot-starter 버전이 업그레이드 되면서 spring-cloud-starter-netflix-zuul의 ZuulHandlerMapping 클래스에서 ErrorController.getErrorPath() NoSuchMethodError 발생했고 오류 두시간 이상 오류 해결이 되지 않아 springboot, spring cloud 라이브러리 버전을 변경하게 되었다.  
|라이브러리|변경전|변경후|
|------|---|---|
|Springboot|2.5.4|2.3.12.RELEASE|
|Spring Cloud|2020.0.3|Hoxton.SR5|  

라이브러리 변경 후 config 서버 설정변경
```yaml
#AS-IS(Spring Cloud-2020.0.3)
spring:
  config:
    import: "optional:configserver:http://localhost:9000"  
    
---

#TO-BE(Spring Cloud-Hoxton.SR5)
spring:
  cloud:
    config:
      uri: http://localhost:9000
```


라이브러리 버전 변경후 Config 서버 구동시 오류  
Springboot 2.5.3 버전에서는 applicaition.yml 파일내에서 encrypt설정을 정상적으로 로드가 되지만 Springboot 버전을 2.3.12.RELEASE로 변경 후 아래 오류가 발생해서 Config 서버의 application-{env}.yml 파일의 이름을 bootstrap-{env}.yml로 변경
```shell
***************************l
APPLICATION FAILED TO START
***************************

Description:

Field rsaProperties in org.springframework.cloud.config.server.config.EncryptionAutoConfiguration$KeyStoreConfiguration required a bean of type 'org.springframework.cloud.bootstrap.encrypt.RsaProperties' that could not be found.

The injection point has the following annotations:
	- @org.springframework.beans.factory.annotation.Autowired(required=false)


Action:

Consider defining a bean of type 'org.springframework.cloud.bootstrap.encrypt.RsaProperties' in your configuration.
```

[장애와 관련된 링크](https://github.com/spring-cloud/spring-cloud-netflix/issues/4008) 해결책을 못찾아서 현재는 라이브러리 버전 다운그레이드....  

### 장애해결 후 재테스트!!!
Gateway 서버로 요청을 보내면 Resource, Resource2 서버로 Routing 되는 것을 확인할 수 있다.  

#### Gateway(9000) 서버로 요청 > Resource 서버로 Routing
```json
GET http://localhost:9100/v1/member/health

HTTP/1.1 200 
Date: Wed, 25 Aug 2021 09:03:23 GMT
Keep-Alive: timeout=60
Content-Type: application/json
Transfer-Encoding: chunked
Connection: keep-alive

MemberController running

Response code: 200; Time: 195ms; Content length: 24 bytes
```  

#### Gateway(9000) 서버로 요청 > Resource2 서버로 Routing  
```json
GET http://localhost:9100/v1/product/health

HTTP/1.1 200 
Date: Wed, 25 Aug 2021 08:56:42 GMT
Keep-Alive: timeout=60
Content-Type: application/json
Transfer-Encoding: chunked
Connection: keep-alive

PayController running

Response code: 200; Time: 56ms; Content length: 21 bytes
```  


## 참고
[DaddyProgrammer Spring CLoud MSA(2)](https://daddyprogrammer.org/post/4401/spring-cloud-msa-gateway-routing-by-netflix-zuul/)  

## Github
<https://github.com/sisipapa/Springboot-MSA.git>