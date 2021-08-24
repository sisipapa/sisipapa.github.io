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



## 참고
[DaddyProgrammer Spring CLoud MSA](https://daddyprogrammer.org/post/4347/spring-cloud-msa-configuration-server/)  
[Jorten [Spring] Cloud Config 구축하기](https://goateedev.tistory.com/167)  
[Spring Cloud Config 에서 변경된 정보를 마이크로서비스 인스턴스에서 Spring Boot Actuator 를 이용하여 반영하기](https://wonit.tistory.com/505)  

## Github
<https://github.com/sisipapa/Springboot-MSA.git>