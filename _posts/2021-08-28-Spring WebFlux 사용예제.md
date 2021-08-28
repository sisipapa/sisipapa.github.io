---
layout: post
title: Spring WebFlux 사용예제
category: [webFlux]
tags: [webFlux, springboot]
redirect_from:

- /2021/08/28/

---

## WebFlux란?  
WebFlux는 Spring Framework5에서 새롭게 추가된 모듈이다. WebFlux는 클라이언트, 서버에서 reactive 스타일의 어플리케이션 개발을 도와주는 모듈이며, reactive-stack 웹 프레임워크로 non-blocking에 reactive stream을 지원한다.  

개발자는 Reactive Stack 를 사용할지, Servlet Stack 를 사용할지 선택을 해야 한다. 두개의 stack 을 동시에 사용할 수 없다.   
<img src="https://sisipapa.github.io/assets/images/posts/webFlux.png" >  

Spring MVC와 WebFlux의 공통점은 @Controller, Reactive 클라이언트이다. 둘 다 Tomcat, Jetty, Undertow와 같은 서버에서 실행할 수 있고 Spring MVC에서는 명령형 논리, JDBC, JPA를 가질 수 있다. Spring WebFlux에서는 기능적 엔드 포인트, 이벤트 루프, 동시성 모델을 가질 수 있다. Spring WebFlux는 Netty 서버에서 실행할 수 있다는 장점이 있다.  

### Mono와 Fulx  
Spring Webflux에서 사용하는 reactive library가 Reactor이고 Reactor가 Reactive Streams의 구현체.  
Flux와 Mono는 Reactor 객체이며, 차이점은 발행하는 데이터 갯수.  
```text
Flux : 0 ~ N 개의 데이터 전달
Mono : 0 ~ 1 개의 데이터 전달
```    

## Intellij Project 생성
[devKuma님의 Spring WebFlux의 간단한 사용법](https://devuna.tistory.com/108) 정리가 잘되어 있어 예제들을 따라해 보면서 webFlux가 무엇인지 이해를 해보려고 한다.  

### Spring Initializr 프로젝트 생성  
<img src="https://sisipapa.github.io/assets/images/posts/webflux-project1.png" >  

### Spring reactive web 선택  
<img src="https://sisipapa.github.io/assets/images/posts/webflux-project2.png" >  

## HelloController 

### @GetMapping("/") API 작성  
```java
@RestController
public class HelloController {

    @GetMapping("/")
    Flux<String> hello() {
        return Flux.just("Hello", "World");
    }
} 
```  

### @GetMapping("/") API 요청테스트(Accept: text/plain)  
```json
GET http://localhost:8080

HTTP/1.1 200 OK
transfer-encoding: chunked
Content-Type: application/json;charset=UTF-8

HelloWorld

Response code: 200 (OK); Time: 173ms; Content length: 10 bytes
```  

### @GetMapping("/") API 요청테스트(Accept: text/event-stream)
```json
GET http://localhost:8080

HTTP/1.1 200 OK
transfer-encoding: chunked
Content-Type: text/event-stream;charset=UTF-8

data:Hello

data:World



Response code: 200 (OK); Time: 194ms; Content length: 24 bytes
```  

### @GetMapping("/") API 요청테스트(Accept: application/stream+json)  
```json
GET http://localhost:8080

HTTP/1.1 200 OK
transfer-encoding: chunked
Content-Type: application/stream+json;charset=UTF-8

HelloWorld

Response code: 200 (OK); Time: 203ms; Content length: 10 bytes
```  

String을 반환하는 API로 차이를 느끼기 어렵기 때문에 이번에는 Flux의 Stream을 생성해서 응답 결과를 비교해 보려고 한다.  



## 참고  
[devKuma - Spring WebFlux의 간단한 사용법](http://www.devkuma.com/pages/1514)  
[튜나 개발일기 - [Spring] WebFlux의 개념 / Spring MVC와 간단비교](https://devuna.tistory.com/108)


## Github    
