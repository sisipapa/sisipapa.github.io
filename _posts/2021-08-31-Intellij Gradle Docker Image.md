---
layout: post
title: Intellij Gradle Docker Image
category: [docker]
tags: [docker, dockerHub, gradle]
redirect_from:

- /2021/08/31/

---

간단한 SpringBoot 프로젝트를 만들고, Dockerfile을 통해서 애플리케이션을 docker image로 만들고, 실행해보고, docker hub에 remote로 푸시하는 방법에 대해서 알아보려고 한다.  

## PreSetting    
### 가상화(Virtualization) 지원 및 활성화  
작업 관리자 성능 탭에서 가상화 지원 및 활성화 확인이 가능하다.  
<img src="https://sisipapa.github.io/assets/images/posts/window-process.PNG" >   

### Docker for Window 다운로드  
<https://hub.docker.com/editions/community/docker-ce-desktop-windows/>  
<img src="https://sisipapa.github.io/assets/images/posts/docker-download.PNG" >  

### Docker for Window 설치
install required Windows components for WSL 2 옵션 선택 확인  
<img src="https://sisipapa.github.io/assets/images/posts/docker-install1.PNG" >  
설치중  
<img src="https://sisipapa.github.io/assets/images/posts/docker-install2.PNG" >  
설치완료 후 재부팅 시 아래 이미지가 뜨면 <https://aka.ms/wsl2kernel> 클릭  
<img src="https://sisipapa.github.io/assets/images/posts/docker-install3.PNG" >  
x64 머신용 최신 WSL2 Linux 커널 업데이트 패키지'를 클릭을 하거나 아래 링크 클릭  
<https://wslstorestorage.blob.core.windows.net/wslblob/wsl_update_x64.msi>  
Dos창 설치 확인  
```shell
$ docker -v
Docker version 20.10.8, build 3967b7d
```  

## Springboot 프로젝트 만들기
Intellij 기준 Spring Initializr 프로젝트를 생성한다.  

### build.gradle  
com.palantir.docker plugin을 추가하고 docker{} 빌드 추가  
```properties
plugins {
    id 'org.springframework.boot' version '2.5.4'
    id 'io.spring.dependency-management' version '1.0.11.RELEASE'
    id 'java'
    id 'com.palantir.docker' version '0.25.0'
}

group = 'com.sisipapa.study'
version = '1.2'
sourceCompatibility = '11'

repositories {
    mavenCentral()
}

dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
}

docker {
    name "p8000:${project.version}"
    tag 'DockerHub', "coolguy239/p8000:${project.version}"
    dockerfile file('Dockerfile')
    files tasks.bootJar.outputs.files
    buildArgs(['JAR_FILE': tasks.bootJar.outputs.files.singleFile.name])
}

test {
    useJUnitPlatform()
}
```   
### Dockerfile 생성   
```dockerfile
FROM lpicanco/java11-alpine
EXPOSE 8000
ARG JAR_FILE
COPY ${JAR_FILE} app.jar
ENTRYPOINT ["java", "-jar", "-Dserver.port=8000", "-Djava.security.egd=file:/dev/./urandom", "/app.jar"]
```  

### Controller 추가  
server.port를 리턴하는 Controller 생성.  
```java
@RestController
public class CommonController {

    @Autowired
    private Environment env;

    @GetMapping("/")
    public String port(){
        return env.getProperty("server.port");
    }

}
```

## Docker hub push  
### gradle > docker > dockerPushDockerHub 더블클릭 
<img src="https://sisipapa.github.io/assets/images/posts/docker-push1.PNG" >   
<img src="https://sisipapa.github.io/assets/images/posts/docker-push-log.PNG" >    


### Docker hub Image 확인  
<img src="https://sisipapa.github.io/assets/images/posts/docker-push2.PNG" >  

## Docker Image run  
### Dos 창에서 실행  
<img src="https://sisipapa.github.io/assets/images/posts/docker-run1.PNG" >  

### Docker Apllicaiton내에서 실행  
<img src="https://sisipapa.github.io/assets/images/posts/docker-run2.PNG" >   
<img src="https://sisipapa.github.io/assets/images/posts/docker-run3.PNG" >      
  
Optional Settings(Container Name, Ports 입력)  
<img src="https://sisipapa.github.io/assets/images/posts/docker-run4.PNG" >   
LOG 확인 및 브라우저 확인 가능
<img src="https://sisipapa.github.io/assets/images/posts/docker-run5.PNG" >    

## 참고  
[gradle docker image 생성하기: spring boot](https://psawesome.tistory.com/78)  
[[4] Spring Boot 에서 gradle task 설정하기](https://velog.io/@guswns3371/4-Spring-Boot-%EC%97%90%EC%84%9C-gradle-task-%EC%84%A4%EC%A0%95%ED%95%98%EA%B8%B0)    
[Spring Boot, Dockerfile로 이미지 생성, 배포하기](https://umanking.github.io/2021/07/11/spring-boot-docker-starter/)  

## Github  
<https://github.com/sisipapa/Springboot-docker>  