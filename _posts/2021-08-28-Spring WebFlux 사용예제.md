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

## @Controller모델 - hello(Flux 기본 API)
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

## @Controller모델 - stream(무한 Stream API)
String을 반환하는 API로 차이를 느끼기 어렵기 때문에 이번에는 Flux의 무한 Stream을 생성해서 응답 결과를 비교해 보려고 한다.
```java
@RestController
public class HelloController {

    @GetMapping("/")
    Flux<String> hello() {
        return Flux.just("Hello", "World");
    }

    @GetMapping("/stream")
    Flux<Map<String, Integer>> stream() {
        Stream<Integer> stream = Stream.iterate(0, i -> i + 1); // Java8의 무한Stream
        return Flux.fromStream(stream.limit(200))
                .map(i -> Collections.singletonMap("value", i));
    }
    
} 
```  
아래 그림은 Intellij의 Http Request에서 요청한 이미지이다. 요청 후 10분이 지나도 webflux-request#4, webflux-request#5 계속 진행중인 걸 확인했다. 현재 툴에서는 현재 진행 중인 과정 확인이 어려워 응답 결과 확인을 위해 limit 200을 주고 진행해 보려고 한다.  
<img src="https://sisipapa.github.io/assets/images/posts/webflux-project2.png" >  

### @GetMapping("/stream") API 요청테스트(Accept: application/json)  
```json
GET http://localhost:8080/stream

HTTP/1.1 200 OK
transfer-encoding: chunked
Content-Type: application/stream+json

{
  "value": 0
}
{
  "value": 1
}

...

{
"value": 198
}
{
"value": 199
}


Response code: 200 (OK); Time: 179ms; Content length: 2690 bytes
```  

### @GetMapping("/stream") API 요청테스트(Accept: text/event-stream)   
```json
GET http://localhost:8080/stream

HTTP/1.1 200 OK
transfer-encoding: chunked
Content-Type: text/event-stream;charset=UTF-8

data:{"value":0}

data:{"value":1}

...

data:{"value":198}

data:{"value":199}



Response code: 200 (OK); Time: 198ms; Content length: 3890 bytes
```  

### @GetMapping("/stream") API 요청테스트(Accept: application/stream+json)  
```json
GET http://localhost:8080/stream

HTTP/1.1 200 OK
transfer-encoding: chunked
Content-Type: application/stream+json

{
  "value": 0
}
{
  "value": 1
}

...

{
"value": 198
}
{
"value": 199
}


Response code: 200 (OK); Time: 206ms; Content length: 2690 bytes
```  

## @Controller모델 - POST  
Spring MVC Controller와 동일하게 @RequestBody로 클라이언트의 요청 본문을 전달받는다. Spring WebFlux는 요청 본문을 Mono로 받고, 비동기적으로 처리를 chain/compose으로 할 수 있다. Mono로 감싸지 않고 String으로 받은 경우는 논블러킹으로 동기화 처리된다. 이 경우 요청 본문을 대문자로 변환하는 map의 결과 Mono를 그대로 반환한다. Mono는 1 또는 0 요소의 Publisher이다.  
```java
@RestController
public class HelloController {

    @GetMapping("/")
    Flux<String> hello() {
        return Flux.just("Hello", "World");
    }

    @GetMapping("/stream")
    Flux<Map<String, Integer>> stream() {
        Stream<Integer> stream = Stream.iterate(0, i -> i + 1);
        return Flux.fromStream(stream).zipWith(Flux.interval(Duration.ofSeconds(1)))
                .map(tuple -> Collections.singletonMap("value", tuple.getT1() /* 튜플의 첫 번째 요소 = Stream<Integer> 요소 */));
    }

    @PostMapping("/echo")
    Mono<String> echo(@RequestBody Mono<String> body) {
        return body.map(String::toUpperCase);
    }
}
```  

### @PostMapping("/echo") API 요청테스트(Accept: application/json)  
```http request
POST http://localhost:8080/echo
Content-Type: application/json

"aaa"
###
```  
```json
POST http://localhost:8080/echo

HTTP/1.1 200 OK
Content-Type: text/plain;charset=UTF-8
Content-Length: 5

"AAA"

Response code: 200 (OK); Time: 214ms; Content length: 5 bytes
```  

## Router Functions 모델
Spring MVC는 @RestController와 @GetMapping,@PostMapping 등의 어노테이션을 선언해서 라우팅을 정의했었는데 Route Function은 경로와 Handler 함수를 통해 라우팅을 정의한다.  
SpringBoot 2.0에서는 Route Functions와 @Controller 모델은 동시에 사용을 할 수가 없기 때문에 동일한 경로의 라우팅이 있다면 Route Functions가 우선 된다.  

### HelloHandler  
Handler URI는 Controller 요청이 중복되어 사라지는 것을 방지하기 위해 ROOT URI에 /handler를 추가했다.  
```java
import java.util.Collections;
import java.util.Map;
import java.util.stream.Stream;

import org.springframework.core.ParameterizedTypeReference;
import org.springframework.core.ResolvableType;
import org.springframework.http.MediaType;
import org.springframework.stereotype.Component;
import org.springframework.web.reactive.function.server.RouterFunction;
import org.springframework.web.reactive.function.server.ServerRequest;
import org.springframework.web.reactive.function.server.ServerResponse;

import reactor.core.publisher.Flux;
import reactor.core.publisher.Mono;

import static org.springframework.web.reactive.function.BodyExtractors.toFlux;
import static org.springframework.web.reactive.function.BodyInserters.fromPublisher;
import static org.springframework.web.reactive.function.server.RequestPredicates.GET;
import static org.springframework.web.reactive.function.server.RequestPredicates.POST;
import static org.springframework.web.reactive.function.server.RouterFunctions.route;
import static org.springframework.web.reactive.function.server.ServerResponse.ok;

@Component
public class HelloHandler {

    public RouterFunction<ServerResponse> routes() {
        return route(GET("/handler"), this::hello)
                .andRoute(GET("/handler/stream"), this::stream)
                .andRoute(POST("/handler/echo"), this::echo)
                .andRoute(POST("/handler/stream"), this::postStream);
    }

    public Mono<ServerResponse> hello(ServerRequest req) {
        return ok().body(Flux.just("Hello", "World!"), String.class);
    }

    public Mono<ServerResponse> stream(ServerRequest req) {
        Stream<Integer> stream = Stream.iterate(0, i -> i + 1);
        Flux<Map<String, Integer>> flux = Flux.fromStream(stream)
                .map(i -> Collections.singletonMap("value", i));
        return ok().contentType(MediaType.APPLICATION_NDJSON)
                .body(fromPublisher(flux, new ParameterizedTypeReference<Map<String, Integer>>(){}));
    }
    
    public Mono<ServerResponse> echo(ServerRequest req) {
        Mono<String> body = req.bodyToMono(String.class).map(String::toUpperCase);
        return ok().body(body, String.class);
    }

    public Mono<ServerResponse> postStream(ServerRequest req) {
        Flux<Map<String, Integer>> body = req.body(toFlux( // BodyExtractors.toFlux을 static import해야 한다.
                new ParameterizedTypeReference<Map<String, Integer>>(){}));
        
        return ok().contentType(MediaType.TEXT_EVENT_STREAM)
                .body(fromPublisher(body.map(m -> Collections.singletonMap("double", m.get("value") * 2)),
                        new ParameterizedTypeReference<Map<String, Integer>>(){}));
    }
}
```  

### HelloHandler 테스트 - [GET] stream  
요청
```http request
GET http://localhost:8080/handler/stream
Accept: application/json

###
```  

결과
```json
GET http://localhost:8080/handler/stream

HTTP/1.1 200 OK
transfer-encoding: chunked
Content-Type: application/x-ndjson

{"value":0}
{"value":1}

...

{"value":198}
{"value":199}


Response code: 200 (OK); Time: 243ms; Content length: 2690 bytes
```

### HelloHandler 테스트 - [POST] echo  
요청  
```http request
POST http://localhost:8080/handler/echo
Accept: application/json

"aaa"
###
```   
결과
```json
POST http://localhost:8080/handler/echo

HTTP/1.1 200 OK
Content-Type: text/plain;charset=UTF-8
Content-Length: 5

"AAA"

Response code: 200 (OK); Time: 84ms; Content length: 5 bytes
```  

### HelloHandler 테스트 - [POST] stream
요청
```http request
POST http://localhost:8080/handler/stream

{"value":1}
{"value":2}
{"value":3}
###
```   
결과  
```json
POST http://localhost:8080/handler/stream

HTTP/1.1 200 OK
transfer-encoding: chunked
Content-Type: text/event-stream;charset=UTF-8

data:{"double":2}

data:{"double":4}

data:{"double":6}



Response code: 200 (OK); Time: 63ms; Content length: 57 bytes
```  
Handler의 RequestPredicates.GET RequestPredicates.POST는 Controller의 @GetMapping, @PostMapping에 해당한다.   
오늘은 [dekuma님의 블로그](http://www.devkuma.com/pages/1514)를 따라하면서 Controller와 Handler에서 동일한 예제를 실행해 보았고 webFlux가 이러게 쓰는 거라는 정도만 확인해 보았다. 아직은 webFlux에 대한 학습이 더 필요할 것 같다.  




## 참고  
[devKuma - Spring WebFlux의 간단한 사용법](http://www.devkuma.com/pages/1514)  
[튜나 개발일기 - [Spring] WebFlux의 개념 / Spring MVC와 간단비교](https://devuna.tistory.com/108)


## Github    
<https://github.com/sisipapa/Springboot-webFlux.git>