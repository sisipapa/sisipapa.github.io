---
layout: post 
title: Intellij Springboot Multiple Module(ver.Gradle)
category: [intellij]
tags: [intellij, springboot, gradle]
redirect_from:

- /2021/08/16/

---

Springboot로 MSA 환경을 구성해보려고 생각을 하다가 Intellij에서 Springboot관련한 Multi Module 환경을 구성에 대한 정리를 먼저 해야 겠다는 생각이 들어 정리를 시작합니다. 

## Intellij Multi Module 환경구성   
### Root 프로젝트 생성(Gradle 프로젝트)  
1. New Project > 왼쪽메뉴 Gradle 선택 | Java선택 후 Next버튼 클릭  
   <img src="https://sisipapa.github.io/assets/images/posts/intellij-multi1.PNG" >  
2. Name,Location 정보 입력 후 Finish  
   <img src="https://sisipapa.github.io/assets/images/posts/intellij-multi2.PNG" >  
   
### Sub module 프로젝트 생성(Spring 프로젝트 - common, sub1, sub2)  
1. Root프로젝트를 선택하고 Ctrl + Alt + Insert 버튼을 클릭  
   <img src="https://sisipapa.github.io/assets/images/posts/intellij-sub1.PNG" >
2. Spring Initializer 선택하고 Java선택 후 Next버튼 클릭  
   <img src="https://sisipapa.github.io/assets/images/posts/intellij-sub2.PNG" >  
3. common과 동일한 과정으로 sub1,sub2 module 생성   
   
### Root 프로젝트 settings.gradle
Root 프로젝트의 settings.grale에 module이 추가되었는지 확인하고 추가가 안되어 있다면 수동으로 추가를 해준다.  
```properties
rootProject.name = 'multimodule'
include 'common'
include 'sub1'
include 'sub2'
```    

### Root 프로젝트 build.gradle  
subprojects 하위의 내용은 Root프로젝트 하위의 모든 모듈에 공통으로 적용이 되는 내용이다. project(':common')의 경우 bootJar enabled=false, jar enabled=true 설정을 해줘야 sub1, sub2 모듈에서 common 모듈을 사용할 수 있다.  
```properties
plugins {
    id 'org.springframework.boot' version '2.5.3'
    id 'io.spring.dependency-management' version '1.0.11.RELEASE'
    id 'java'
}

subprojects {

    apply plugin: 'java'
    apply plugin: 'org.springframework.boot'
    apply plugin: 'io.spring.dependency-management'

    group = 'com.sisipapa.study.multimodule'
    version = '0.0.1-SNAPSHOT'
    sourceCompatibility = '11'

    repositories {
        mavenCentral()
    }

    dependencies {
        implementation 'org.springframework.boot:spring-boot-starter'
        implementation 'org.springframework.boot:spring-boot-starter-web'
        compileOnly 'org.projectlombok:lombok'
        annotationProcessor 'org.projectlombok:lombok'
        testImplementation('org.springframework.boot:spring-boot-starter-test') {
            exclude group: 'org.junit.vintage', module: 'junit-vintage-engine'
        }
    }
}

project(':common') {
    bootJar {
        enabled = false
    }
    jar {
        enabled = true
    }
    dependencies {
        implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
        runtimeOnly 'com.h2database:h2'
        runtimeOnly 'org.postgresql:postgresql'
    }
}

project(':sub1') {
    dependencies {
        implementation project(':common')
    }
}

project(':sub2') {
    dependencies {
        implementation project(':common')
    }
}
```  

### Sub module 프로젝트 build.gradle
common, sub1, sub2 Sub Module의 build.gradle의 내용을 모두 삭제했다. 파일을 삭제하지는 않았다.  

여기까지 해주면 Intellij에서 Multi Module에 대한 기본설정은 되었다.  

## 설정 확인 및 테스트
### 설정 확인
화면 우측 Gradle 메뉴를 클릭 후 Root프로젝트 하위의 sub1, sub2 Sub Module의 Dependency를 확인해 보면 추가하지 않은 Dependency가 존재하는 것을 확인할 수 있다.  
<img src="https://sisipapa.github.io/assets/images/posts/intellij-sub3.PNG" >  

### 테스트  
common module에 Stirng 값을 return 해주는 CommonService의 Method를 만들고 sub1, sub2 module의 Controller에서 사용해 보는 예제이다.  

- common Module에 만든 Service  
```java  
package com.sisipapa.study.multimodule.common.service;

import org.springframework.stereotype.Service;

@Service
public class CommonService {
    public String getIndex(String index){
        return index;
    }
}
```   

- sub1 Module Controller   
```java  
@AllArgsConstructor
@RestController
public class Sub1Controller {

    private CommonService service;

    @GetMapping("/sub1")
    public String index(){
        return service.getIndex("sub1");
    }

}
```    

- sub2 Module COntroller    
```java
@AllArgsConstructor
@RestController
public class Sub2Controller {

    private CommonService service;

    @GetMapping("/sub2")
    public String index(){
        return service.getIndex("sub2");
    }

}
```    

- sub1 module application.properties  
```properties
server.port=8081
```  

- sub2 module application.properties  
```properties
server.port=8082
```    

- sub1 module 결과화면  
  <img src="https://sisipapa.github.io/assets/images/posts/sub1_index.PNG" >  
- sub2 module 결과화면  
  <img src="https://sisipapa.github.io/assets/images/posts/sub2_index.PNG" >  

  
## 참고
[springboot - gradle을 이용한 멀티모듈 프로젝트 만들기](https://www.hanumoka.net/2019/10/04/springBoot-20191004-springboot-gradle-multimodule/)
[Spring Boot2 멀티 모듈 프로젝트 with Gradle](https://blog.selectjun.com/9)

## Github
<https://github.com/sisipapa/Springboot-MultiModule-Gradle.git>
