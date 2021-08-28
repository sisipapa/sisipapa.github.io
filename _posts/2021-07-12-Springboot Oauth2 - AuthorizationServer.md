---
layout: post 
title: Springboot Oauth2 - AuthorizationServer
category: [oauth2]
tags: [springboot, oauth2]
redirect_from:

- /2021/07/12/

---

늦었지만 이전 직장에서 인증서버 담당자로 일하면서 운영했던 Springboot + Oauth2 + JWT를 활용한 인증서버에 대해 정리해보려고 한다. 참고 링크에 포함된 아빠 프로그래머님의 블로그 oauth2 인증서버 관련 내용이 정리가 잘되어 있어 참고해서 정리를 하려고 한다.  

## Pre Setting
- H2 DB설치  
이전 블로그 [Springboot Document 라이브러리2(Spring Restdoc)](https://sisipapa.github.io/blog/2021/04/09/Springboot-Document-%EB%9D%BC%EC%9D%B4%EB%B8%8C%EB%9F%AC%EB%A6%AC2(Spring-Restdoc)/)의 H2 DB설치 부분을 참고해서 H2 DB를 설치한다. 
  
## build.gradle 및 Configration 파일 작성  
### build.gradle  
```properties
plugins {
    id 'org.springframework.boot' version '2.1.4.RELEASE'
    id 'io.spring.dependency-management' version '1.0.11.RELEASE'
    id 'java'
}

group = 'com.sisipapa'
version = '0.0.1-SNAPSHOT'
sourceCompatibility = '11'

configurations {
    compileOnly {
        extendsFrom annotationProcessor
    }
}

repositories {
    mavenCentral()
}

dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    implementation 'org.springframework.boot:spring-boot-starter-web'
    compileOnly 'org.projectlombok:lombok'
    runtimeOnly 'com.h2database:h2'
    annotationProcessor 'org.projectlombok:lombok'
    testImplementation 'org.springframework.boot:spring-boot-starter-test'

    implementation 'org.springframework.cloud:spring-cloud-starter-security:2.1.2.RELEASE'
    implementation 'org.springframework.cloud:spring-cloud-starter-oauth2:2.1.2.RELEASE'
    implementation 'com.google.code.gson:gson'
}

test {
    useJUnitPlatform()
}
```  

### Authorization Server Config 생성  
SpringSecurity 5버전에서는 NoOpPasswordEncoder가 deprecate 되어 secret의 value값 앞에 {noop}을 붙여야 오류없이 테스트 진행이 가능하다.  
withClient : 인증서버에 인가된 client인지 확인을 위한 ID  
secret : 인증서버에 인가된 client인지 확인을 위한 SECRET  
redirectUris : 인증 완료 후 이동할 클라이언트 웹 페이지 주소로 code 값을 같이 보내준다.
authorizedGrantTypes : 인증방식은 총 4가지가 있다. 여기서는 authorization_code를 사용한다.  
scopes : accessToken으로 접근할 수 있는 리소스의 범위  
accessTokenValiditySeconds : 발급된 accessToken의 유효시간(초)  

```java

import org.springframework.context.annotation.Configuration;
import org.springframework.security.oauth2.config.annotation.configurers.ClientDetailsServiceConfigurer;
import org.springframework.security.oauth2.config.annotation.web.configuration.AuthorizationServerConfigurerAdapter;
import org.springframework.security.oauth2.config.annotation.web.configuration.EnableAuthorizationServer;

@Configuration
@EnableAuthorizationServer
public class Oauth2AuthorizationConfig extends AuthorizationServerConfigurerAdapter {

    public void configure(ClientDetailsServiceConfigurer clients) throws Exception {
        clients.inMemory()
                .withClient("testClientId")
                .secret("{noop}testSecret")
                .redirectUris("http://localhost:8081/oauth2/callback")
                .authorizedGrantTypes("authorization_code")
                .scopes("read", "write")
                .accessTokenValiditySeconds(30000);

    }
}
```  

### SpringSecurity Config 생성  
SpringSecurity 5버전에서는 NoOpPasswordEncoder가 deprecate 되어 password의 value값 앞에 {noop}을 붙여야 오류없이 테스트 진행이 가능하다.    

```java
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.authentication.builders.AuthenticationManagerBuilder;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.annotation.web.configuration.WebSecurityConfigurerAdapter;

@Configuration
@EnableWebSecurity
public class SecurityConfig extends WebSecurityConfigurerAdapter {

    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        auth.inMemoryAuthentication()
                .withUser("test")
                .password("{noop}test")
                .roles("USER");
    }
    @Override
    protected void configure(HttpSecurity security) throws Exception {
        security
                .csrf().disable()
                .headers().frameOptions().disable()
                .and()
                .authorizeRequests().antMatchers("/oauth/**", "/oauth2/callback", "/h2-console/*").permitAll()
                .and()
                .formLogin().and()
                .httpBasic();
    }
}
```  

### WebMvcConfig 생성
```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.crypto.factory.PasswordEncoderFactories;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.web.client.RestTemplate;
import org.springframework.web.servlet.config.annotation.CorsRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;

@Configuration
public class WebMvcConfig implements WebMvcConfigurer {
    private static final long MAX_AGE_SECONDS = 3600;

    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/**")
                .allowedOrigins("*")
                .allowedMethods("GET", "POST", "PUT", "DELETE")
                .allowedHeaders("*")
                .allowCredentials(true)
                .maxAge(MAX_AGE_SECONDS);
    }

    @Bean
    public RestTemplate getRestTemplate() {
        return new RestTemplate();
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return PasswordEncoderFactories.createDelegatingPasswordEncoder();
    }
}
```  

### 토큰정보를 받을 모델
```java  

import lombok.Getter;
import lombok.Setter;

@Getter
@Setter
public class OAuthToken {
    private String access_token;
    private String token_type;
    private String refresh_token;
    private long expires_in;
    private String scope;
}
```  

### application.yml 설정
```yaml  
server:
  port: 8081
spring:
  h2:
    console:
      enabled: true
      settings:
        web-allow-others: true
  datasource:
    url: jdbc:h2:tcp://localhost/~/test
    driver-class-name: org.h2.Driver
    username: sa
  jpa:
    database-platform: org.hibernate.dialect.H2Dialect
    properties.hibernate.hbm2ddl.auto: update
    showSql: true
```  

## 테스트
1. 로그인페이지 접속 : <http://localhost:8081/oauth/authorize?client_id=testClientId&redirect_uri=http://localhost:8081/oauth2/callback&response_type=code&scope=read>  
2. 아이디/패스워드 입력 후 Sign in 버튼 클릭(test / test)  
   <img src="https://sisipapa.github.io/assets/images/posts/oauth2-login.png" >  
3. Oauth Approval 설정 - Approve 체크박스 선택 후 Authorize 버튼 클릭  
   <img src="https://sisipapa.github.io/assets/images/posts/oauth2-approval.png" >  
4. 토큰 확인  
   <img src="https://sisipapa.github.io/assets/images/posts/oauth2-token.png" >  
   
## 프로젝트 설정시 오류  
아빠프로그래머님의 블로그의 build.gradle의 springboot version과 spring-cloud-starter-security, spring-cloud-starter-oauth2 버전을 최신 버전으로 변경 시 오류가 발생했다. 여기서는 오류원인 파악 및 해결에 초점을 두기 보다는 인증서버 구성을 빠르게 진행해 보기 위해 블로그와 동일한 설정을 적용했다.  
```text
Caused by: java.lang.NoClassDefFoundError: org/springframework/boot/context/properties/ConfigurationPropertiesBean
	at org.springframework.cloud.context.properties.ConfigurationPropertiesBeans.postProcessBeforeInitialization(ConfigurationPropertiesBeans.java:94) ~[spring-cloud-context-2.2.4.RELEASE.jar:2.2.4.RELEASE]
	at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.applyBeanPostProcessorsBeforeInitialization(AbstractAutowireCapableBeanFactory.java:414) ~[spring-beans-5.1.6.RELEASE.jar:5.1.6.RELEASE]
	at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.initializeBean(AbstractAutowireCapableBeanFactory.java:1770) ~[spring-beans-5.1.6.RELEASE.jar:5.1.6.RELEASE]
	at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.doCreateBean(AbstractAutowireCapableBeanFactory.java:593) ~[spring-beans-5.1.6.RELEASE.jar:5.1.6.RELEASE]
	... 26 common frames omitted
Caused by: java.lang.ClassNotFoundException: org.springframework.boot.context.properties.ConfigurationPropertiesBean
	at java.base/jdk.internal.loader.BuiltinClassLoader.loadClass(BuiltinClassLoader.java:583) ~[na:na]
	at java.base/jdk.internal.loader.ClassLoaders$AppClassLoader.loadClass(ClassLoaders.java:178) ~[na:na]
	at java.base/java.lang.ClassLoader.loadClass(ClassLoader.java:521) ~[na:na]
	... 30 common frames omitted
```  

## 참고    
[아빠프로그래머 Spring Boot Oauth2 - AuthorizationServer](https://daddyprogrammer.org/post/1239/spring-oauth-authorizationserver/)

