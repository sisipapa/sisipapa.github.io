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
```shell
> Task :compileJava
> Task :processResources
> Task :classes
> Task :bootJarMainClassName
> Task :bootJar
> Task :dockerClean UP-TO-DATE
> Task :dockerPrepare

> Task :docker
#1 [internal] load build definition from Dockerfile
#1 sha256:f5039bf16e45d59f91b57928fd75c17b30a24947015ae558c622af6282d8af31
#1 transferring dockerfile:
#1 transferring dockerfile: 225B done
#1 DONE 0.3s

#2 [internal] load .dockerignore
#2 sha256:f95ea9928809b580ab19af8c0dc931f4ae6d8002587f717c808f2f8ea42cbce1
#2 transferring context: 2B done
#2 DONE 0.3s

#3 [internal] load metadata for docker.io/lpicanco/java11-alpine:latest
#3 sha256:58d09c891944c55525a27bae13de640e2322e2d5cffc8c1cf94abf29cc7311e9
#3 ...

#4 [auth] lpicanco/java11-alpine:pull token for registry-1.docker.io
#4 sha256:1f7953651195a71b8fc780e5881466fb41c6308571b59e3fb863a48cf0bd4858
#4 DONE 0.0s

#3 [internal] load metadata for docker.io/lpicanco/java11-alpine:latest
#3 sha256:58d09c891944c55525a27bae13de640e2322e2d5cffc8c1cf94abf29cc7311e9
#3 DONE 3.2s

#5 [1/2] FROM docker.io/lpicanco/java11-alpine@sha256:2146492ec807b8b0cadcfabe382a77ee5f92dbe2de1908b753d3405203f68289
#5 sha256:e83809e535af21afb0faf3b95d32f7c4b2548b5388c9aa2d05a5e7bb2757361a
#5 resolve docker.io/lpicanco/java11-alpine@sha256:2146492ec807b8b0cadcfabe382a77ee5f92dbe2de1908b753d3405203f68289 0.0s done
#5 sha256:25d68f7e74ef54de49d436537e465a48ef6c27823710b82d3f50df25e3412aba 4.45kB / 4.45kB done
#5 sha256:c87736221ed0bcaa60b8e92a19bec2284899ef89226f2a07968677cf59e637a4 0B / 2.21MB 0.1s
#5 sha256:8e16f361ba88e695f66440c684fe002e890a05e8ac2820909165e62f2e64dec4 0B / 5.37MB 0.1s
#5 sha256:2146492ec807b8b0cadcfabe382a77ee5f92dbe2de1908b753d3405203f68289 1.16kB / 1.16kB done
#5 sha256:696681e6a12d44118e1c0f5da7ff2885ea48682722e493dca9e916fd9a0eea76 0B / 33.91MB 0.3s
#5 ...

#6 [internal] load build context
#6 sha256:2add75ae2ee2e1ad8021c3f14f358468409b8c7e22b147026d2fd0c9b320715c
#6 transferring context: 17.35MB 0.3s done
#6 DONE 0.4s

#5 [1/2] FROM docker.io/lpicanco/java11-alpine@sha256:2146492ec807b8b0cadcfabe382a77ee5f92dbe2de1908b753d3405203f68289
#5 sha256:e83809e535af21afb0faf3b95d32f7c4b2548b5388c9aa2d05a5e7bb2757361a
#5 sha256:c87736221ed0bcaa60b8e92a19bec2284899ef89226f2a07968677cf59e637a4 1.05MB / 2.21MB 0.4s
#5 extracting sha256:c87736221ed0bcaa60b8e92a19bec2284899ef89226f2a07968677cf59e637a4
#5 sha256:c87736221ed0bcaa60b8e92a19bec2284899ef89226f2a07968677cf59e637a4 2.21MB / 2.21MB 0.5s done
#5 sha256:8e16f361ba88e695f66440c684fe002e890a05e8ac2820909165e62f2e64dec4 1.05MB / 5.37MB 0.7s
#5 sha256:538ff3cf24ef4274694803fdbc4db69a7ee08dc612ea0eeea337c33da7b074ca 0B / 148B 0.7s
#5 sha256:8e16f361ba88e695f66440c684fe002e890a05e8ac2820909165e62f2e64dec4 3.15MB / 5.37MB 1.0s
#5 extracting sha256:c87736221ed0bcaa60b8e92a19bec2284899ef89226f2a07968677cf59e637a4 0.2s done
#5 sha256:538ff3cf24ef4274694803fdbc4db69a7ee08dc612ea0eeea337c33da7b074ca 148B / 148B 1.0s
#5 sha256:8e16f361ba88e695f66440c684fe002e890a05e8ac2820909165e62f2e64dec4 4.19MB / 5.37MB 1.1s
#5 sha256:538ff3cf24ef4274694803fdbc4db69a7ee08dc612ea0eeea337c33da7b074ca 148B / 148B 1.0s done
#5 sha256:8e16f361ba88e695f66440c684fe002e890a05e8ac2820909165e62f2e64dec4 5.37MB / 5.37MB 1.2s
#5 sha256:696681e6a12d44118e1c0f5da7ff2885ea48682722e493dca9e916fd9a0eea76 2.10MB / 33.91MB 1.2s
#5 sha256:8e16f361ba88e695f66440c684fe002e890a05e8ac2820909165e62f2e64dec4 5.37MB / 5.37MB 1.2s done
#5 sha256:696681e6a12d44118e1c0f5da7ff2885ea48682722e493dca9e916fd9a0eea76 5.24MB / 33.91MB 1.4s
#5 extracting sha256:8e16f361ba88e695f66440c684fe002e890a05e8ac2820909165e62f2e64dec4
#5 sha256:696681e6a12d44118e1c0f5da7ff2885ea48682722e493dca9e916fd9a0eea76 7.34MB / 33.91MB 1.6s
#5 sha256:696681e6a12d44118e1c0f5da7ff2885ea48682722e493dca9e916fd9a0eea76 12.58MB / 33.91MB 1.8s
#5 extracting sha256:8e16f361ba88e695f66440c684fe002e890a05e8ac2820909165e62f2e64dec4 0.3s done
#5 sha256:696681e6a12d44118e1c0f5da7ff2885ea48682722e493dca9e916fd9a0eea76 15.73MB / 33.91MB 1.9s
#5 sha256:696681e6a12d44118e1c0f5da7ff2885ea48682722e493dca9e916fd9a0eea76 22.02MB / 33.91MB 2.1s
#5 sha256:696681e6a12d44118e1c0f5da7ff2885ea48682722e493dca9e916fd9a0eea76 25.17MB / 33.91MB 2.2s
#5 sha256:696681e6a12d44118e1c0f5da7ff2885ea48682722e493dca9e916fd9a0eea76 31.46MB / 33.91MB 2.4s
#5 sha256:696681e6a12d44118e1c0f5da7ff2885ea48682722e493dca9e916fd9a0eea76 33.91MB / 33.91MB 2.5s
#5 sha256:696681e6a12d44118e1c0f5da7ff2885ea48682722e493dca9e916fd9a0eea76 33.91MB / 33.91MB 2.5s done
#5 extracting sha256:696681e6a12d44118e1c0f5da7ff2885ea48682722e493dca9e916fd9a0eea76 0.1s
#5 extracting sha256:696681e6a12d44118e1c0f5da7ff2885ea48682722e493dca9e916fd9a0eea76 0.9s done
#5 extracting sha256:538ff3cf24ef4274694803fdbc4db69a7ee08dc612ea0eeea337c33da7b074ca
#5 extracting sha256:538ff3cf24ef4274694803fdbc4db69a7ee08dc612ea0eeea337c33da7b074ca done
#5 DONE 4.2s

#7 [2/2] COPY docker-1.2.jar app.jar
#7 sha256:2cad8cbf0569af6b921285cc5f5d439e7352f73b651e26a5c3e11c14c42d5cb9
#7 DONE 0.7s

#8 exporting to image
#8 sha256:e8c613e07b0b7ff33893b694f7759a10d42e180f2b4dc349fb57dc6b71dcab00
#8 exporting layers
#8 exporting layers 0.2s done
#8 writing image sha256:cd4af33e637aded6ccace5119f6c487f88d855ad7ed0d2aef9df812d02f00fe3 0.0s done
#8 naming to docker.io/library/p8000:1.2
#8 naming to docker.io/library/p8000:1.2 done
#8 DONE 0.3s

Use 'docker scan' to run Snyk tests against images to find vulnerabilities and learn how to fix them

> Task :dockerTagDockerHub

> Task :dockerPushDockerHub
The push refers to repository [docker.io/coolguy239/p8000]
e7b55fef7c08: Preparing
b62ce2e20fb6: Preparing
974adc247f15: Preparing
9a8bad52f245: Preparing
d9ff549177a9: Preparing
d9ff549177a9: Layer already exists
9a8bad52f245: Layer already exists
b62ce2e20fb6: Layer already exists
974adc247f15: Layer already exists
e7b55fef7c08: Pushed
1.2: digest: sha256:6418f883e000bcb234f9ed9d91f1be95f6fdc61f6f212a8fd43b20ab7590a39e size: 1370

BUILD SUCCESSFUL in 44s
9 actionable tasks: 8 executed, 1 up-to-date
오후 12:21:06: Task execution finished 'dockerPushDockerHub'.
```  

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