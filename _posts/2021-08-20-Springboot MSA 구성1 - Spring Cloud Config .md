---
layout: post
title: Springboot MSA 구성1 - Spring Cloud Config
category: [msa]
tags: [msa, springboot, config]
redirect_from:

- /2021/08/20/

---

Spring Cloud Config는 MSA 시스템의 환경 설정들을 중앙화해서 한 곳에서 관리를 할 수 있게 해주고 설정파일의 변경되어도 어플리케이션의 재배포 없이 적용이 가능하다.  
진행할 프로젝트 구성은 아래와 같이 진행할 예정이다.  
1. Spring Cloud Config 서버  
2. Spring Cloud Zuul(API Gateway) 서버  
3. Spring Cloud Eureka 서버  

1~3번까지 완료 후 시간이 된다면 인증서버까지 적용해 볼 예정이다. 오늘은 MSA 프로젝트의 첫번째 Spring Cloud Config 서버를 구성해 보려고 한다. 그리고 Config 서버에서 설정한 프로퍼티들을 Resource 서버에서 확인을 해 볼 예정이다.

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
### build.gradle
```properties
# 공통 Dependency
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter'
    
    compileOnly 'org.projectlombok:lombok'
    annotationProcessor 'org.projectlombok:lombok'
    testImplementation('org.springframework.boot:spring-boot-starter-test') {
        exclude group: 'org.junit.vintage', module: 'junit-vintage-engine'
    }
}

# Config Dependency
project(':Config') {
    dependencies {
        implementation 'org.springframework.cloud:spring-cloud-config-server'
    }
    
    dependencyManagement {
        imports {
            mavenBom "org.springframework.cloud:spring-cloud-dependencies:${springCloudVersion}"
        }
    }
}
```  

### Application.java  
EnableConfigServer 어노테이션 추가
```java
@EnableConfigServer
@SpringBootApplication
public class ConfigApplication {

    public static void main(String[] args) {
        SpringApplication.run(ConfigApplication.class, args);
    }

}
```  

### application-{env}.yml  
#### local
```yaml
server:
  port: 9000
spring:
  application:
    name: configServer
  cloud:
    config:
      server:
        git:
          uri: https://github.com/sisipapa/Springboot-MSA
          search-paths: config-repo/local
```
#### prod  
```yaml
server:
  port: 9000
spring:
  application:
    name: configServer
  cloud:
    config:
      server:
        git:
          uri: https://github.com/sisipapa/Springboot-MSA
          search-paths: config-repo/prod
```  

### config-repo 
config-repo 모듈에 application 별 active.profiles 별로 설정파일을 만든다. 여기서는 message의 내용만 조금 다르게 설정했다.
#### Resource/local 
```yaml
spring:
  profiles: local
  message: Resource Service Local Server!!!!!
```  

#### Resource2/local  
```yaml
spring:
  profiles: local
  message: Resource2 Service Local Server!!!!!
```

여기까지만 하면 Spring Cloud Config 서버의 설정은 끝이다. 로컬서버 구동시 -Dspring.profiles.active=local로 설정 후 서버를 기동한다.  

### Config 서버 테스트
```json
GET http://localhost:9000/resource/local

HTTP/1.1 200 
Content-Type: application/json
Transfer-Encoding: chunked
Date: Tue, 24 Aug 2021 09:31:28 GMT
Keep-Alive: timeout=60
Connection: keep-alive

{
  "name": "resource",
  "profiles": [
    "local"
  ],
  "label": null,
  "version": "5eb3f2ee05568a9878370171ccfc4a88af79a71c",
  "state": null,
  "propertySources": [
    {
      "name": "https://github.com/sisipapa/Springboot-MSA/file:C:\\Users\\user\\AppData\\Local\\Temp\\config-repo-17387822938375408351\\config-repo\\local\\resource-local.yml",
      "source": {
        "spring.profiles": "local",
        "spring.message": "Resource Service Local Server!!!!!"
      }
    }
  ]
}

Response code: 200; Time: 3933ms; Content length: 401 bytes  


GET http://localhost:9000/resource2/local

HTTP/1.1 200 
Content-Type: application/json
Transfer-Encoding: chunked
Date: Tue, 24 Aug 2021 09:32:15 GMT
Keep-Alive: timeout=60
Connection: keep-alive

{
  "name": "resource2",
  "profiles": [
    "local"
  ],
  "label": null,
  "version": "5eb3f2ee05568a9878370171ccfc4a88af79a71c",
  "state": null,
  "propertySources": [
    {
      "name": "https://github.com/sisipapa/Springboot-MSA/file:C:\\Users\\user\\AppData\\Local\\Temp\\config-repo-17387822938375408351\\config-repo\\local\\resource2-local.yml",
      "source": {
        "spring.profiles": "local",
        "spring.message": "Resource Service Local Server!!!!!"
      }
    }
  ]
}

Response code: 200; Time: 822ms; Content length: 403 bytes
```  

## Resource 서버  
### application-{env}.yml
```yaml
server:
  port: 8080
spring:
  application:
    name: resource
  config:
    import: "optional:configserver:http://localhost:9000"
management:
  endpoints:
    web:
      exposure:
        include: info, refresh
```  

### Controller  
GIT 에 저장된 property 값이 변경 된 경우, client에서 최신 파일을 다시 받아 값을 refresh 하기 위해서 RefreshScope 어노테이션을 추가해준다.  
```java
@RequestMapping("/v1/resource")
@RestController
@RefreshScope
public class ResourceController {

    @Value("${spring.message}")
    private String message;

    @GetMapping("/message")
    public String message() {
        return "ResourceController message : " + message;
    }
}
```  

### Resource 서버에서 Config 조회  
```json
GET http://localhost:8080/v1/resource/message

HTTP/1.1 200 
Content-Type: application/json
Content-Length: 63
Date: Tue, 24 Aug 2021 09:42:19 GMT
Keep-Alive: timeout=60
Connection: keep-alive

ResourceController message : Resource Service Local Server!!!!!

Response code: 200; Time: 327ms; Content length: 63 bytes
```  

Resource 서버의 [GET]/message API 호출 결과로 Config property를 정상적으로 읽어 온 것을 확인했다. 이제 config-repo의 설정파일의 내용을 수정하고 동일한 API를 호출하면 변경된 결과가 나오지 않는다. 변경된 설정을 Resource 서버에서도 적용을 하기 위해서는 [POST]/actuator/refresh API를 호출해야 변경된 설정이 적용된다. API 결과로 변경된 property 목록이 리턴된다.     
```json
POST http://localhost:8080/actuator/refresh

HTTP/1.1 200 
Content-Type: application/json
Transfer-Encoding: chunked
Date: Tue, 24 Aug 2021 09:47:06 GMT
Keep-Alive: timeout=60
Connection: keep-alive

[
  "config.client.version",
  "spring.message"
]

Response code: 200; Time: 1540ms; Content length: 42 bytes
```   

## Property value 암호화/복호화  
Config 서버에서 Resource 서버에 제공하는 설정 정보 중에는 외부에 노출되어서는 안되는 정보들이 있을 수 있다. 그래서 value값을 저장 시 암호화된 값을 저장할 수 있도록 지원한다.  

### keypair 생성
keytool -genkeypair 옵션 설명   

```shell
$ keytool -genkeypair -help
keytool -genkeypair [OPTION]...

Generates a key pair

Options:

 -alias <alias>          alias name of the entry to process
 -keyalg <alg>           key algorithm name
 -keysize <size>         key bit size
 -groupname <name>       Group name. For example, an Elliptic Curve name.
 -sigalg <alg>           signature algorithm name
 -destalias <alias>      destination alias
 -dname <name>           distinguished name
 -startdate <date>       certificate validity start date/time
 -ext <value>            X.509 extension
 -validity <days>        validity number of days
 -keypass <arg>          key password
 -keystore <keystore>    keystore name
 -storepass <arg>        keystore password
 -storetype <type>       keystore type
 -providername <name>    provider name
 -addprovider <name>     add security provider by name (e.g. SunPKCS11)
   [-providerarg <arg>]    configure argument for -addprovider
 -providerclass <class>  add security provider by fully-qualified class name
   [-providerarg <arg>]    configure argument for -providerclass
 -providerpath <list>    provider classpath
 -v                      verbose output
 -protected              password through protected mechanism

Use "keytool -?, -h, or --help" for this help message
```  

keypair 생성  
Window PC의 경우 JDK를 환경변수 Path등록을 하지 않았다면 JDK가 설치된 경로로 이동해서 아래 명령어를 실행한다.
```shell
$ keytool -genkeypair -alias config-server-key -keyalg RSA \
  -dname "CN=Config Server,OU=Spring Cloud,O=sisipapa" \
  -keypass keypass123 -keystore config-server.jks -storepass storepass123
```  
위의 명령을 실행하면 config-server.jks 파일이 생성된다. jks 파일을 Config 모듈의 resources 디렉토리에 복사한다.  

### 암호화 설정추가  
application-{env}.yml 파일에 암호화 관련 설정을 추가한다.  
```yaml
encrypt:
  key-store:
    location: classpath:/config-server.jks
    password: storepass123
    alias: config-server-key
    secret: keypass123
```  

### 데이터 암호화  
암호화 설정을 추가하고 Config 모듈을 재기동 후 Config 서버의 /encrypt API를 통해 데이터를 암호화한다. 아래는 config-repo의 resource-local.yml db.ip, db.port
```json
POST http://localhost:9000/encrypt

HTTP/1.1 200 
Content-Type: text/plain;charset=UTF-8
Content-Length: 388
Date: Tue, 24 Aug 2021 15:06:10 GMT
Keep-Alive: timeout=60
Connection: keep-alive
        
AQBXjueGBbxNGPkv7DNYGs/zbQzJKC7rov1IgLG+Qm4l+OQ+ItN108j8Kw+bCcE57O2nD5zaH/pu8796iqxDRpZfdHJD+TS5Ej0zggdfvdaT+8nPOa3gwy9b5WfBi9PtZXgjeimqVwo9xJ4h4YE7I69dP09w4Y0FYJuciE6t+D5oCSbur1joySz8ausDFvw72YUtmpuEw1jLTwka+T3zQfgMBQsnAmV0QH+p/xnCJJ6UHHS0fu6bu+rqyViHfFbB1NRBzUn/b9sVgx9xbMWN80AeBFZebsspHBA76B+RG7N2Kksk5WtmoebsjDPVvlESCKNY0LnXtrOT2s8+7BqO6nYKF8+s/S8x350UrmC2cWSQ5fRDuGTZ05vNpBHFaGl3MP4=
AQAWG7JW48DuhyEqaFRsja62We9Fw94lGqCJBDRv1nXvolWV17nMarXqVqHQSgdD86l8WasXh6k9SlIH660fJclruOgfM0EBwJK+bXmSrGsfzqQs7UmWnJGAraH2Xe6vUL2dj9uq/nXih8glq3UPjpzoK5c4iYEr3G7xdesQhzupG9Yqu7Dqv53HzRVxQnooj2WiSRD2tiqYw1JCGR79Q4RtNs7t0d3ObpcaVOSGQLsl/8L5Rw9GFZfZbfiuYwYQfZ+ooHxJf3XjiO2qCiKuk8e64hy+PDk0WhF9zlZK0svj1HxBdDRNrYjWZI7TfIXinOmaVu5wVDkaNg14oS7y6Toa9vcTCfi1HK24r94GD0naOhDHCWmL5Caek/0JQk7ihjI=
AQBdqizzBGEwvI9n09gVVbPM9yapxvluq1nOVWSHYlfRrQXHf2ONsYsUqYtIP0TE/ijG8O6raD8L02PflI1NdUe/TCwo59VJX2wdvf041HuNVnJN0poYuwdWSg0BWZ7MiDDmID0m202u249exj91nnOjQ+fzbRp80y39QMtaZ90X74e0rAWxU+8kqePjk7z+Y4Yx+iqRTY88Cxulk05ijnyR4oG/vv6+Q4DQH/h9U5Rdcjh0Ndttc3XJGnDa7DmILGWD1gZ3+dbpXU2GisaZgEk2MRc21J20Ac0NFZWJvmSMwm9hLpaCGB4CceghDYZ/yAZJvUMgA3aZtFWd2LJ0qKekL5SSVA/4oGLi8pBVvaCSmKgqvzHS94opZz+srLHwljw=
AQCGy8uXlX8tFPHJTCEhMVYRiJFkaN3yvucHeXrsNsoxFO3l5ds3oHnnwZlbNv23HLbLCt7CmFsw1V7i8b3yqgtDSfgL8nMgmZUJ0rDRAToUEqIkcohkKS8tHreYbjFTurz632S+m4AouFsq6LaGIZU6ftctwTH6bb8v2SccCejcIuvTWoKOr8QKzJhszZH17pJLG4NJDcKqktv0+ZwA2tdr5nb+IE4Eg8RMO6sPnZUHKibqiwmle5C0hDMSBE0KlL4UvNAQg1yyJfLdk/Lm/P1tNyii6WWgG79IWwitbYDtBfSj4nm3mQyMq6zdcS1wYyCb+LvmMejp0r1LwXwYbYFqpwhFUtAkTKrvAhVnYpbQbtfJvdpyu1ZWq3WDKNseE3Y=
        
Response code: 200; Time: 316ms; Content length: 388 bytes
```  

resource-local.yml 설정파일에 암호화된 db.ip, db.port, db.id, db.password 설정추가  
```yaml
spring:
  profiles: local
  message: Resource Service Local Server!!!!!=>수정1
db:
  ip: '{cipher}AQBXjueGBbxNGPkv7DNYGs/zbQzJKC7rov1IgLG+Qm4l+OQ+ItN108j8Kw+bCcE57O2nD5zaH/pu8796iqxDRpZfdHJD+TS5Ej0zggdfvdaT+8nPOa3gwy9b5WfBi9PtZXgjeimqVwo9xJ4h4YE7I69dP09w4Y0FYJuciE6t+D5oCSbur1joySz8ausDFvw72YUtmpuEw1jLTwka+T3zQfgMBQsnAmV0QH+p/xnCJJ6UHHS0fu6bu+rqyViHfFbB1NRBzUn/b9sVgx9xbMWN80AeBFZebsspHBA76B+RG7N2Kksk5WtmoebsjDPVvlESCKNY0LnXtrOT2s8+7BqO6nYKF8+s/S8x350UrmC2cWSQ5fRDuGTZ05vNpBHFaGl3MP4='
  port: '{cipher}AQAWG7JW48DuhyEqaFRsja62We9Fw94lGqCJBDRv1nXvolWV17nMarXqVqHQSgdD86l8WasXh6k9SlIH660fJclruOgfM0EBwJK+bXmSrGsfzqQs7UmWnJGAraH2Xe6vUL2dj9uq/nXih8glq3UPjpzoK5c4iYEr3G7xdesQhzupG9Yqu7Dqv53HzRVxQnooj2WiSRD2tiqYw1JCGR79Q4RtNs7t0d3ObpcaVOSGQLsl/8L5Rw9GFZfZbfiuYwYQfZ+ooHxJf3XjiO2qCiKuk8e64hy+PDk0WhF9zlZK0svj1HxBdDRNrYjWZI7TfIXinOmaVu5wVDkaNg14oS7y6Toa9vcTCfi1HK24r94GD0naOhDHCWmL5Caek/0JQk7ihjI='
  id: '{cipher}AQBdqizzBGEwvI9n09gVVbPM9yapxvluq1nOVWSHYlfRrQXHf2ONsYsUqYtIP0TE/ijG8O6raD8L02PflI1NdUe/TCwo59VJX2wdvf041HuNVnJN0poYuwdWSg0BWZ7MiDDmID0m202u249exj91nnOjQ+fzbRp80y39QMtaZ90X74e0rAWxU+8kqePjk7z+Y4Yx+iqRTY88Cxulk05ijnyR4oG/vv6+Q4DQH/h9U5Rdcjh0Ndttc3XJGnDa7DmILGWD1gZ3+dbpXU2GisaZgEk2MRc21J20Ac0NFZWJvmSMwm9hLpaCGB4CceghDYZ/yAZJvUMgA3aZtFWd2LJ0qKekL5SSVA/4oGLi8pBVvaCSmKgqvzHS94opZz+srLHwljw='
  password: '{cipher}AQCGy8uXlX8tFPHJTCEhMVYRiJFkaN3yvucHeXrsNsoxFO3l5ds3oHnnwZlbNv23HLbLCt7CmFsw1V7i8b3yqgtDSfgL8nMgmZUJ0rDRAToUEqIkcohkKS8tHreYbjFTurz632S+m4AouFsq6LaGIZU6ftctwTH6bb8v2SccCejcIuvTWoKOr8QKzJhszZH17pJLG4NJDcKqktv0+ZwA2tdr5nb+IE4Eg8RMO6sPnZUHKibqiwmle5C0hDMSBE0KlL4UvNAQg1yyJfLdk/Lm/P1tNyii6WWgG79IWwitbYDtBfSj4nm3mQyMq6zdcS1wYyCb+LvmMejp0r1LwXwYbYFqpwhFUtAkTKrvAhVnYpbQbtfJvdpyu1ZWq3WDKNseE3Y='
```  

### 복호화 테스트
config-repo 모듈에서 resource 서버의 local 서버가 바라보는 설정파일에 db접속정보를 조회하는 API 호출테스트
```java
@RequestMapping("/v1/resource")
@RestController
@RefreshScope
public class ResourceController {

    @Value("${spring.message}")
    private String message;

    @Value("${db.ip}")
    private String ip;
    
    @Value("${db.port}")
    private String port;
    
    @Value("${db.id}")
    private String id;
    
    @Value("${db.password}")
    private String password;

    @GetMapping("/message")
    public String message() {
        return "ResourceController message : " + message;
    }

    @GetMapping("/db")
    public String db() {return "ResourceController db : " + ip + ":" + port + " [" + id +" : " + password + "]";}
}
```  

```json
GET http://localhost:8080/v1/resource/db

HTTP/1.1 200 
Content-Type: application/json
Content-Length: 56
Date: Tue, 24 Aug 2021 15:28:00 GMT
Keep-Alive: timeout=60
Connection: keep-alive

ResourceController db : 10.10.10.10: 2222 [dbid: dbpass]

Response code: 200; Time: 235ms; Content length: 56 bytes
```  

Config 서버에서 암호화 되있던 property db.ip,db.port,db.id,db.password 정보가 resource 서버에서 복호화 된 데이터로 출력되는 것을 확인할 수 있다.  

## 참고
[DaddyProgrammer Spring CLoud MSA(1)](https://daddyprogrammer.org/post/4347/spring-cloud-msa-configuration-server/)  
[Jorten [Spring] Cloud Config 구축하기](https://goateedev.tistory.com/167)  
[Spring Cloud Config 에서 변경된 정보를 마이크로서비스 인스턴스에서 Spring Boot Actuator 를 이용하여 반영하기](https://wonit.tistory.com/505)  

## Github
<https://github.com/sisipapa/Springboot-MSA.git>