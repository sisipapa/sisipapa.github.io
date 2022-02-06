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
### Spring Cloud Gateway-Filter 적용   
아래 두개의 예시의 결과는 동일하다.   
- @Configuration Bean을 등록해서 적용 - Java 레벨에서 RequestHeader, ResponseHeader에 값을 설정한 예시이다.  

```java
import org.springframework.cloud.gateway.route.RouteLocator;
import org.springframework.cloud.gateway.route.builder.RouteLocatorBuilder;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class FilterConfig {

    @Bean
    public RouteLocator gatewayRoutes(RouteLocatorBuilder builder){
        return builder.routes()
                .route(r -> r.path("/first-service/**")
                        .filters(f -> f.addRequestHeader("first-request", "first-request-header")
                                        .addResponseHeader("first-response", "first-response-header"))
                        .uri("http://localhost:8081"))
                .route(r -> r.path("/second-service/**")
                        .filters(f -> f.addRequestHeader("second-request", "second-request-header")
                                .addResponseHeader("second-response", "second-response-header"))
                        .uri("http://localhost:8082"))
                .build();

    }
}
```  

- yaml 파일로 적용 - Yaml 설정파일로 RequestHeader, ResponseHeader에 값을 설정한 예시이다.  
```yaml
server:
  port: 8000
eureka:
  client:
    register-with-eureka: false
    fetch-registry: false
spring:
  application:
    name: apigateway-service
  cloud:
    gateway:
      routes:
        - id: first-service
          uri: http://localhost:8081
          predicates:
            - Path=/first-service/**
          filters:
            - AddRequestHeader=first-request, first-request-header
            - AddResponseHeader=first-response, first-response-header
        - id: second-service
          uri: http://localhost:8082
          predicates:
            - Path=/second-service/**
          filters:
            - AddRequestHeader=second-request, second-request-header
            - AddResponseHeader=second-response, second-response-header
```  

### Spring Cloud Gateway-Custom Filter 적용  
#### Custom Filter 클래스 생성 
AbstractGatewayFilterFactory 상속받고 apply 메소드를 재정의 한다.  
Config inner class 를 만든다.  
기본생성자를 만든다.  

```java
import lombok.extern.slf4j.Slf4j;
import org.springframework.cloud.gateway.filter.GatewayFilter;
import org.springframework.cloud.gateway.filter.factory.AbstractGatewayFilterFactory;
import org.springframework.http.server.reactive.ServerHttpRequest;
import org.springframework.http.server.reactive.ServerHttpResponse;
import org.springframework.stereotype.Component;
import reactor.core.publisher.Mono;

@Component
@Slf4j
public class CustomFilter extends AbstractGatewayFilterFactory<CustomFilter.Config> {

    public CustomFilter() {
        super(Config.class);
    }

    @Override
    public GatewayFilter apply(Config config) {
        // Custom Pre Filter
        return (exchange, chain) -> {
            ServerHttpRequest request = exchange.getRequest();
            ServerHttpResponse response = exchange.getResponse();

            log.info("Custom PRE filter : request id -> {}", request.getId());

            // Custom Post Filter
            return chain.filter(exchange).then(Mono.fromRunnable(() -> {
                log.info("Custom POST filter : response code -> {}", response.getStatusCode());
            }));
        };
    }

    public static class Config {
        // Put the configuration properties
    }
}
```

#### application.yml 파일에 CustomFilter 등록  
AddRequestHeader, AddResponseHeader를 주석하고 CustomFilter를 등록한다.  

```yaml
server:
  port: 8000
eureka:
  client:
    register-with-eureka: false
    fetch-registry: false
spring:
  application:
    name: apigateway-service
  cloud:
    gateway:
      routes:
        - id: first-service
          uri: http://localhost:8081
          predicates:
            - Path=/first-service/**
          filters:
#            - AddRequestHeader=first-request, first-request-header2
#            - AddResponseHeader=first-response, first-response-header2
            - CustomFilter
        - id: second-service
          uri: http://localhost:8082
          predicates:
            - Path=/second-service/**
          filters:
#            - AddRequestHeader=second-request, second-request-header2
#            - AddResponseHeader=second-response, second-response-header2
            - CustomFilter
```  

### Spring Cloud Gateway-Global Filter 적용  
#### Global Filter 클래스 생성
Global Filter를 만드는 방법은 Custom Filter와 동일하다.    
AbstractGatewayFilterFactory 상속받고 apply 메소드를 재정의 한다.    
Config inner class 를 만든다.    
기본생성자를 만든다.    

```java
import lombok.Data;
import lombok.extern.slf4j.Slf4j;
import org.springframework.cloud.gateway.filter.GatewayFilter;
import org.springframework.cloud.gateway.filter.factory.AbstractGatewayFilterFactory;
import org.springframework.http.server.reactive.ServerHttpRequest;
import org.springframework.http.server.reactive.ServerHttpResponse;
import org.springframework.stereotype.Component;
import reactor.core.publisher.Mono;

@Component
@Slf4j
public class GlobalFilter extends AbstractGatewayFilterFactory<GlobalFilter.Config> {

    public GlobalFilter() {
        super(Config.class);
    }

    @Override
    public GatewayFilter apply(Config config) {
        return (exchange, chain) -> {
            ServerHttpRequest request = exchange.getRequest();
            ServerHttpResponse response = exchange.getResponse();

            log.info("Global filter baseMessage : {}", config.getBaseMessage());

            if(config.isPreLogger()){
                log.info("Global filter Start : request id -> {}", request.getId());
            }

            // Custom Post Filter
            return chain.filter(exchange).then(Mono.fromRunnable(() -> {
                if(config.isPostLogger()){
                    log.info("Global filter End : response code -> {}", response.getStatusCode());
                }
            }));
        };
    }

    @Data
    public static class Config {
        // Put the configuration properties
        private String baseMessage;
        private boolean preLogger;
        private boolean postLogger;
    }
}
```  

#### application.yml 파일에 GlobalFilter 등록  
spring.cloud.gateway.default-filters에 Global Filter를 등록한다.  

```yaml
server:
  port: 8000
eureka:
  client:
    register-with-eureka: false
    fetch-registry: false
spring:
  application:
    name: apigateway-service
  cloud:
    gateway:
      default-filters:
        - name: GlobalFilter
          args:
            baseMessage: Spring Cloud Gateway Global Filter
            PreLogger: true
            PostLogger: true
      routes:
        - id: first-service
          uri: http://localhost:8081
          predicates:
            - Path=/first-service/**
          filters:
#            - AddRequestHeader=first-request, first-request-header2
#            - AddResponseHeader=first-response, first-response-header2
            - CustomFilter
        - id: second-service
          uri: http://localhost:8082
          predicates:
            - Path=/second-service/**
          filters:
#            - AddRequestHeader=second-request, second-request-header2
#            - AddResponseHeader=second-response, second-response-header2
            - CustomFilter
```  


### Spring Cloud Gateway-Logging Filter 적용  
#### Logging Filter 클래스 생성
기존 작업한 Global Filter와 거의 동일하기 때문에 GlobalFilter.java 파일을 복사해서 작업한다.  
OrderedGatewayFilter 를 구현한다.  
```java
import lombok.Data;
import lombok.extern.slf4j.Slf4j;
import org.springframework.cloud.gateway.filter.GatewayFilter;
import org.springframework.cloud.gateway.filter.OrderedGatewayFilter;
import org.springframework.cloud.gateway.filter.factory.AbstractGatewayFilterFactory;
import org.springframework.core.Ordered;
import org.springframework.http.server.reactive.ServerHttpRequest;
import org.springframework.http.server.reactive.ServerHttpResponse;
import org.springframework.stereotype.Component;
import reactor.core.publisher.Mono;

@Component
@Slf4j
public class LoggingFilter extends AbstractGatewayFilterFactory<LoggingFilter.Config> {

    public LoggingFilter() {
        super(Config.class);
    }

    // Ordered.HIGHEST_PRECEDENCE
    // Ordered.LOWEST_PRECEDENCE
    @Override
    public GatewayFilter apply(Config config) {
        GatewayFilter filter = new OrderedGatewayFilter((exchange, chain) -> {
            ServerHttpRequest request = exchange.getRequest();
            ServerHttpResponse response = exchange.getResponse();

            log.info("Logging filter baseMessage : {}", config.getBaseMessage());

            if(config.isPreLogger()){
                log.info("Logging PRE filter : request id -> {}", request.getId());
            }

            // Custom Post Filter
            return chain.filter(exchange).then(Mono.fromRunnable(() -> {
                if(config.isPostLogger()){
                    log.info("Logging POST filter : response code -> {}", response.getStatusCode());
                }
            }));
        }, Ordered.LOWEST_PRECEDENCE);

        return filter;
    }

    @Data
    public static class Config {
        // Put the configuration properties
        private String baseMessage;
        private boolean preLogger;
        private boolean postLogger;
    }
}
```  
#### application.yml 파일에 LoggingFilter 등록  
second 서비스에만 Logging Filter 를 적용.
```yaml
server:
  port: 8000
eureka:
  client:
    register-with-eureka: false
    fetch-registry: false
spring:
  application:
    name: apigateway-service
  cloud:
    gateway:
      default-filters:
        - name: GlobalFilter
          args:
            baseMessage: Spring Cloud Gateway Global Filter
            PreLogger: true
            PostLogger: true
      routes:
        - id: first-service
          uri: http://localhost:8081
          predicates:
            - Path=/first-service/**
          filters:
#            - AddRequestHeader=first-request, first-request-header2
#            - AddResponseHeader=first-response, first-response-header2
            - CustomFilter
        - id: second-service
          uri: http://localhost:8082
          predicates:
            - Path=/second-service/**
          filters:
            - name: CustomFilter
            - name: LoggingFilter
              args:
                baseMessage: Hi, there
                PreLogger: true
                PostLogger: true
```  

#### Ordered.HIGHEST_PRECEDENCE 실행결과 
LoggingFilter의 우선순위가 가장 높다.  
```shell
2022-02-06 14:11:02.101  INFO 21280 --- [ctor-http-nio-2] c.s.a.filter.LoggingFilter               : Logging filter baseMessage : Hi, there
2022-02-06 14:11:02.101  INFO 21280 --- [ctor-http-nio-2] c.s.a.filter.LoggingFilter               : Logging PRE filter : request id -> 37903ef2-3
2022-02-06 14:11:02.101  INFO 21280 --- [ctor-http-nio-2] c.s.a.filter.GlobalFilter                : Global filter baseMessage : Spring Cloud Gateway Global Filter
2022-02-06 14:11:02.101  INFO 21280 --- [ctor-http-nio-2] c.s.a.filter.GlobalFilter                : Global filter Start : request id -> 37903ef2-3
2022-02-06 14:11:02.101  INFO 21280 --- [ctor-http-nio-2] c.s.a.filter.CustomFilter                : Custom PRE filter : request id -> 37903ef2-3
2022-02-06 14:11:02.115  INFO 21280 --- [ctor-http-nio-2] c.s.a.filter.CustomFilter                : Custom POST filter : response code -> 200 OK
2022-02-06 14:11:02.115  INFO 21280 --- [ctor-http-nio-2] c.s.a.filter.GlobalFilter                : Global filter End : response code -> 200 OK
2022-02-06 14:11:02.116  INFO 21280 --- [ctor-http-nio-2] c.s.a.filter.LoggingFilter               : Logging POST filter : response code -> 200 OK
```  

#### Ordered.LOWEST_PRECEDENCE 실행결과  
LoggingFilter의 우선순위가 가장 낮다.  
```shell
2022-02-06 14:15:44.650  INFO 18540 --- [ctor-http-nio-2] c.s.a.filter.GlobalFilter                : Global filter baseMessage : Spring Cloud Gateway Global Filter
2022-02-06 14:15:44.650  INFO 18540 --- [ctor-http-nio-2] c.s.a.filter.GlobalFilter                : Global filter Start : request id -> 7abe4ddd-1
2022-02-06 14:15:44.651  INFO 18540 --- [ctor-http-nio-2] c.s.a.filter.CustomFilter                : Custom PRE filter : request id -> 7abe4ddd-1
2022-02-06 14:15:46.625  INFO 18540 --- [ctor-http-nio-2] c.s.a.filter.LoggingFilter               : Logging filter baseMessage : Hi, there
2022-02-06 14:15:46.626  INFO 18540 --- [ctor-http-nio-2] c.s.a.filter.LoggingFilter               : Logging PRE filter : request id -> 7abe4ddd-1
2022-02-06 14:15:46.626  INFO 18540 --- [ctor-http-nio-2] c.s.a.filter.LoggingFilter               : Logging POST filter : response code -> 200 OK
2022-02-06 14:15:46.626  INFO 18540 --- [ctor-http-nio-2] c.s.a.filter.CustomFilter                : Custom POST filter : response code -> 200 OK
2022-02-06 14:15:46.626  INFO 18540 --- [ctor-http-nio-2] c.s.a.filter.GlobalFilter                : Global filter End : response code -> 200 OK
```  

### Spring Cloud Gateway-Load Balancer  


## Github
<https://github.com/sisipapa/spring-cloud-inflearn.git>  




