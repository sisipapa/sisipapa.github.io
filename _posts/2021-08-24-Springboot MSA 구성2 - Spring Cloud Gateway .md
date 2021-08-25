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






## 참고
[DaddyProgrammer Spring CLoud MSA(2)](https://daddyprogrammer.org/post/4401/spring-cloud-msa-gateway-routing-by-netflix-zuul/)  

## Github
<https://github.com/sisipapa/Springboot-MSA.git>