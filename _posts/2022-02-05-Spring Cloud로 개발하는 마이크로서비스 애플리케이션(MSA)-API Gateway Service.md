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
1. @Configuration Bean을 등록해서 적용 - Java 레벨에서 RequestHeader, ResponseHeader에 값을 설정한 예시이다.  
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

2. yaml 파일로 적용 - Yaml 설정파일로 RequestHeader, ResponseHeader에 값을 설정한 예시이다.  
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
1. Custom Filter 클래스를 만든다.  
- AbstractGatewayFilterFactory 상속받고 apply 메소드를 재정의 한다.  
- Config inner class 를 만든다.  
- 기본생성자를 만든다.  
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

2. application.yml 파일에 CustomFilter 등록  
- AddRequestHeader, AddResponseHeader를 주석하고 CustomFilter를 등록한다.  
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
### Spring Cloud Gateway-Logging Filter 적용  
### Spring Cloud Gateway-Load Balancer  


## Github
<https://github.com/sisipapa/spring-cloud-inflearn.git>  




