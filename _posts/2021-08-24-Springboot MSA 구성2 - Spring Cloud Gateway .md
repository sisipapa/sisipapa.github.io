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
# zuul-gateway-local.yml
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

# zuul-gateway-prod.yml
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
GET http://localhost:9000/zuul-gateway/local

HTTP/1.1 200 
Content-Type: application/json
Transfer-Encoding: chunked
Date: Wed, 25 Aug 2021 03:39:41 GMT
Keep-Alive: timeout=60
Connection: keep-alive

{
  "name": "zuul-gateway",
  "profiles": [
    "local"
  ],
  "label": null,
  "version": "46e8eb3c8d007a6aef573aa6c815950d0a56fd3b",
  "state": null,
  "propertySources": [
    {
      "name": "https://github.com/sisipapa/Springboot-MSA/file:C:\\Users\\user\\AppData\\Local\\Temp\\config-repo-10700435317011410158\\config-repo\\local\\zuul-gateway-local.yml",
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
```properties

```  











## 참고
[DaddyProgrammer Spring CLoud MSA(2)](https://daddyprogrammer.org/post/4401/spring-cloud-msa-gateway-routing-by-netflix-zuul/)  

## Github
<https://github.com/sisipapa/Springboot-MSA.git>