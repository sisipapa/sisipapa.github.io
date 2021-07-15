---
layout: post 
title: Springboot Oauth2 - ResourceServer
category: [oauth2]
tags: [springboot, oauth2]
redirect_from:

- /2021/07/14/

---

아빠프로그래머님의 블로그를 참고해서 ResourceServer를 만들어 보려고한다. JWT의 대칭키, 비대칭키를 이용한 서명을 사용해서 구현할 예정이다.  

## Resource 서버 구성 
Resource 서버의 경우 대칭키와 비대칭키를 사용했을 때 Controller, Repository, Entity는 파일들은 동일하게 사용하고 Oauth2ResourceServerConfig.java 파일만 수정해서 테스트를 진행할 예정이다.  

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
    annotationProcessor 'org.projectlombok:lombok'
    testImplementation 'org.springframework.boot:spring-boot-starter-test'

    runtimeOnly 'com.h2database:h2'
    implementation 'org.springframework.cloud:spring-cloud-starter-security:2.1.2.RELEASE'
    implementation 'org.springframework.cloud:spring-cloud-starter-oauth2:2.1.2.RELEASE'
    testImplementation 'org.junit.jupiter:junit-jupiter-api'
    testImplementation 'org.junit.jupiter:junit-jupiter-engine'
}

test {
    useJUnitPlatform()
}
```  

### application.yml
Authorization Server는 8081포트를 사용하고, Resource Server는 8080포트를 사용하도록 설정.
```yaml
server:
  port: 8080

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

security:
  oauth2:
    #    client:
    #      client-id: testClientId
    #      client-secret: testSecret
    #    resource:
    #      token-info-uri: http://localhost:8081/oauth/check_token
    jwt:
      signkey: 123@#$
```  

### Entity
```java
import com.fasterxml.jackson.annotation.JsonProperty;
import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Getter;
import lombok.NoArgsConstructor;
import org.springframework.security.core.GrantedAuthority;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.security.core.userdetails.UserDetails;

import javax.persistence.*;
import java.util.ArrayList;
import java.util.Collection;
import java.util.List;
import java.util.stream.Collectors;

@Builder // builder를 사용할수 있게 합니다.
@Entity // jpa entity임을 알립니다.
@Getter // user 필드값의 getter를 자동으로 생성합니다.
@NoArgsConstructor // 인자없는 생성자를 자동으로 생성합니다.
@AllArgsConstructor // 인자를 모두 갖춘 생성자를 자동으로 생성합니다.
@Table(name = "user") // 'user' 테이블과 매핑됨을 명시
public class User implements UserDetails {
    @Id // pk
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private long msrl;
    @Column(nullable = false, unique = true, length = 50)
    private String uid;
    @JsonProperty(access = JsonProperty.Access.WRITE_ONLY)
    @Column(length = 100)
    private String password;
    @Column(nullable = false, length = 100)
    private String name;
    @Column(length = 100)
    private String provider;

    @ElementCollection(fetch = FetchType.EAGER)
    @Builder.Default
    private List<String> roles = new ArrayList<>();

    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() {
        return this.roles.stream().map(SimpleGrantedAuthority::new).collect(Collectors.toList());
    }

    @JsonProperty(access = JsonProperty.Access.WRITE_ONLY)
    @Override
    public String getUsername() {
        return this.uid;
    }

    @JsonProperty(access = JsonProperty.Access.WRITE_ONLY)
    @Override
    public boolean isAccountNonExpired() {
        return true;
    }

    @JsonProperty(access = JsonProperty.Access.WRITE_ONLY)
    @Override
    public boolean isAccountNonLocked() {
        return true;
    }

    @JsonProperty(access = JsonProperty.Access.WRITE_ONLY)
    @Override
    public boolean isCredentialsNonExpired() {
        return true;
    }

    @JsonProperty(access = JsonProperty.Access.WRITE_ONLY)
    @Override
    public boolean isEnabled() {
        return true;
    }
}
```  

### Repository
```java
import com.sisipapa.resource.entity.User;
import org.springframework.data.jpa.repository.JpaRepository;

public interface UserJpaRepository extends JpaRepository<User, Long> {
}
```  

### Controller  
회원목록을 조회하는 API를 생성.  
```java
import com.sisipapa.resource.entity.User;
import com.sisipapa.resource.repository.UserJpaRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import java.util.List;

@RequiredArgsConstructor
@RestController
@RequestMapping(value = "/v1")
public class UserController {

    private final UserJpaRepository userJpaRepository;

    @GetMapping(value = "/users")
    public List<User> findAllUser() {
        return userJpaRepository.findAll();
    }
}
```  

### Oauth2ResourceServerConfig(JWT 대칭키)  
Oauth2ResourceServerConfig 설정  
```java
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.oauth2.config.annotation.web.configuration.EnableResourceServer;
import org.springframework.security.oauth2.config.annotation.web.configuration.ResourceServerConfigurerAdapter;
import org.springframework.security.oauth2.provider.token.TokenStore;
import org.springframework.security.oauth2.provider.token.store.JwtAccessTokenConverter;
import org.springframework.security.oauth2.provider.token.store.JwtTokenStore;

@Configuration
@EnableResourceServer
public class Oauth2ResourceServerConfig extends ResourceServerConfigurerAdapter {

    @Value("${security.oauth2.jwt.signkey}")
    private String signKey;

    @Override
    public void configure(HttpSecurity http) throws Exception {
        http.headers().frameOptions().disable();
        http.authorizeRequests()
                .antMatchers("/v1/users").access("#oauth2.hasScope('read')")
                .anyRequest().authenticated();
    }

    @Bean
    public TokenStore tokenStore() {
        return new JwtTokenStore(accessTokenConverter());
    }

    @Bean
    public JwtAccessTokenConverter accessTokenConverter() {
        JwtAccessTokenConverter converter = new JwtAccessTokenConverter();
        converter.setSigningKey(signKey);
        return converter;
    }

}
```  

Oauth2ResourceServerConfig 설정  
```java
import com.sisipapa.oauth2.service.CustomUserDetailService;
import lombok.RequiredArgsConstructor;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.security.oauth2.config.annotation.configurers.ClientDetailsServiceConfigurer;
import org.springframework.security.oauth2.config.annotation.web.configuration.AuthorizationServerConfigurerAdapter;
import org.springframework.security.oauth2.config.annotation.web.configuration.EnableAuthorizationServer;
import org.springframework.security.oauth2.config.annotation.web.configurers.AuthorizationServerEndpointsConfigurer;
import org.springframework.security.oauth2.provider.token.store.JdbcTokenStore;
import org.springframework.security.oauth2.provider.token.store.JwtAccessTokenConverter;

import javax.sql.DataSource;

@RequiredArgsConstructor
@Configuration
@EnableAuthorizationServer
public class Oauth2AuthorizationConfig extends AuthorizationServerConfigurerAdapter {

    private final PasswordEncoder passwordEncoder;
    private final DataSource dataSource;
    private final CustomUserDetailService userDetailService;

    @Value("${security.oauth2.jwt.signkey}")
    private String signKey;

    /**
     * 클라이언트 정보를 DB 정보로 인증
     */
    @Override
    public void configure(ClientDetailsServiceConfigurer clients) throws Exception {
        clients.jdbc(dataSource).passwordEncoder(passwordEncoder);
    }

    /**
     * 토큰 정보를 DB 관리
     * @return
     */
//    @Override
//    public void configure(AuthorizationServerEndpointsConfigurer endpoints) throws Exception {
//        endpoints.tokenStore(new JdbcTokenStore(dataSource));
//    }

    /**
     * 토큰 발급 방식을 JWT 토큰 방식으로 변경한다. 이렇게 하면 토큰 저장하는 DB Table은 필요가 없다.
     *
     * @param endpoints
     * @throws Exception
     */
    @Override
    public void configure(AuthorizationServerEndpointsConfigurer endpoints) throws Exception {
        super.configure(endpoints);
        endpoints.accessTokenConverter(jwtAccessTokenConverter()).userDetailsService(userDetailService);
    }

    /**
     * jwt converter를 등록
     *
     * @return
     */
    @Bean
    public JwtAccessTokenConverter jwtAccessTokenConverter() {
        JwtAccessTokenConverter converter = new JwtAccessTokenConverter();
        converter.setSigningKey(signKey);
        return converter;
    }
}
```  

### Oauth2ResourceServerConfig(JWT 비대칭키)
데이터는 공개키로 암호화해서 비밀키로만 복호화가 가능하도록한다. 인증서버는 리소스서버에게 공개키를 공유하고 해당 키로 암호화하여 데이터를 전닳한다. 공개키와 암호화된 데이터가 노출이 되더라도, 비밀키가 없이는 복호화를 할 수 없기 때문에 높은 보안 수준의 네트워크 통신이 가능하다.  

#### Java KeyStore(JKS) - 자바 키 저장소  
```shell
$ keytool -genkeypair -alias oauth2jwt -keyalg RSA -keypass oauth2jwtpass -keystore oauth2jwt.jks -storepass oauth2jwtpass
What is your first and last name?
  [Unknown]:
What is the name of your organizational unit?
  [Unknown]:
What is the name of your organization?
  [Unknown]:
What is the name of your City or Locality?
  [Unknown]:
What is the name of your State or Province?
  [Unknown]:
What is the two-letter country code for this unit?
  [Unknown]:
Is CN=Unknown, OU=Unknown, O=Unknown, L=Unknown, ST=Unknown, C=Unknown correct?
  [no]:  y
```  

#### 공개키 확인  
```shell
$keytool -list -rfc --keystore oauth2jwt.jks | openssl x509 -inform pem -pubkey
키 저장소 비밀번호 입력:  oauth2jwtpass
-----BEGIN PUBLIC KEY-----
MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAplzVr93U21Dp9vZuzBJt
cGSdl4Cavf2Em1ISjYxEPvfYihhf70zYjYkq1XJQzWNOFxOWq9i/vric+0f7H8ET
eDJ/k4BOtyYhuWGGpr8N944xK09zMkOsDQUoJTlgj1K4HCS+9mYvC7gheG95JCWG
+pDvr4KmR23Wfe5ieopWs3UnyFU2hKqEhi7Cog/QTVPFVGUgxr8SbCW+L4T13uBD
kRAbkLWpB9/Q7fBm75QLza9zU1/B5N3c5lwOhs4z4hDF9bJ5XreaBUiP9boI9kFL
T0TAwmHMtTAPXunHqbPkdPQ97pN2cqkR1bJhlEVIZK2V0RECQb8ELBpp3bzo3obR
zwIDAQAB
-----END PUBLIC KEY-----
-----BEGIN CERTIFICATE-----
MIIDdzCCAl+gAwIBAgIEQ3sjnzANBgkqhkiG9w0BAQsFADBsMRAwDgYDVQQGEwdV
bmtub3duMRAwDgYDVQQIEwdVbmtub3duMRAwDgYDVQQHEwdVbmtub3duMRAwDgYD
VQQKEwdVbmtub3duMRAwDgYDVQQLEwdVbmtub3duMRAwDgYDVQQDEwdVbmtub3du
MB4XDTIxMDcxNTA3MjIyMFoXDTIxMTAxMzA3MjIyMFowbDEQMA4GA1UEBhMHVW5r
bm93bjEQMA4GA1UECBMHVW5rbm93bjEQMA4GA1UEBxMHVW5rbm93bjEQMA4GA1UE
ChMHVW5rbm93bjEQMA4GA1UECxMHVW5rbm93bjEQMA4GA1UEAxMHVW5rbm93bjCC
ASIwDQYJKoZIhvcNAQEBBQADggEPADCCAQoCggEBAKZc1a/d1NtQ6fb2bswSbXBk
nZeAmr39hJtSEo2MRD732IoYX+9M2I2JKtVyUM1jThcTlqvYv764nPtH+x/BE3gy
f5OATrcmIblhhqa/DfeOMStPczJDrA0FKCU5YI9SuBwkvvZmLwu4IXhveSQlhvqQ
76+Cpkdt1n3uYnqKVrN1J8hVNoSqhIYuwqIP0E1TxVRlIMa/Emwlvi+E9d7gQ5EQ
G5C1qQff0O3wZu+UC82vc1NfweTd3OZcDobOM+IQxfWyeV63mgVIj/W6CPZBS09E
wMJhzLUwD17px6mz5HT0Pe6TdnKpEdWyYZRFSGStldERAkG/BCwaad286N6G0c8C
AwEAAaMhMB8wHQYDVR0OBBYEFKMI+tE5ziMSUkhY7JGejEYkXk2/MA0GCSqGSIb3
DQEBCwUAA4IBAQAv1MAGvT+j1ooP14e6KpS5pp1mzXSAstggOJ11mn/hNy25n7DN
EJ5JGeqlkDdyaFi+5gOHbbd2pO1QKrP9EPmgmAAxennLAI0Nih9nMP9oEo3WwkEu
f/p5tt6qbtQP7denfDvQF2qALMav1s9XLTmjlFQSfMtsJ/3DX3YfwStct5kDP3tk
0jMZx70eM2x58gwa6l5MhhNDQcqAsE6/LzMduRxa4h+B4MtSPVHfCyG86TSVQNKF
pJEpHHDE9igvfGXAltcaxAAsKCc5X9hFP4WvmHj2Kxkdhgl8Uj+sGcmLul5zJI+K
C22l+rBHI2tZEUEcDVwd/G6t620x2ASyLAAo
-----END CERTIFICATE-----
```  

#### 인증서버 reources 폴더에 jsk 파일을 복사  
#### 인증서버 Oauth2AuthorizationConfig TokenCOnverter 수정
```java
/**
     * jwt converter를 등록
     *
     * @return
     */
//    @Bean
//    public JwtAccessTokenConverter jwtAccessTokenConverter() {
//        JwtAccessTokenConverter converter = new JwtAccessTokenConverter();
//        converter.setSigningKey(signKey);
//        return converter;
//    }

    /**
     * jwt converter - 비대칭 키 sign
     */
    @Bean
    public JwtAccessTokenConverter jwtAccessTokenConverter() {
        KeyStoreKeyFactory keyStoreKeyFactory = new KeyStoreKeyFactory(new FileSystemResource("src/main/resources/oauth2jwt.jks"), "oauth2jwtpass".toCharArray());
        JwtAccessTokenConverter converter = new JwtAccessTokenConverter();
        converter.setKeyPair(keyStoreKeyFactory.getKeyPair("oauth2jwt"));
        return converter;
    }
```  

#### Resource서버 reources 폴더에 공개키(public.txt) 등록  
```text
-----BEGIN PUBLIC KEY-----
MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAplzVr93U21Dp9vZuzBJt
cGSdl4Cavf2Em1ISjYxEPvfYihhf70zYjYkq1XJQzWNOFxOWq9i/vric+0f7H8ET
eDJ/k4BOtyYhuWGGpr8N944xK09zMkOsDQUoJTlgj1K4HCS+9mYvC7gheG95JCWG
+pDvr4KmR23Wfe5ieopWs3UnyFU2hKqEhi7Cog/QTVPFVGUgxr8SbCW+L4T13uBD
kRAbkLWpB9/Q7fBm75QLza9zU1/B5N3c5lwOhs4z4hDF9bJ5XreaBUiP9boI9kFL
T0TAwmHMtTAPXunHqbPkdPQ97pN2cqkR1bJhlEVIZK2V0RECQb8ELBpp3bzo3obR
zwIDAQAB
-----END PUBLIC KEY-----
```  

#### Oauth2ResourceServerConfig TokenConverter 수정
```java
//    @Bean
//    public JwtAccessTokenConverter accessTokenConverter() {
//        JwtAccessTokenConverter converter = new JwtAccessTokenConverter();
//        converter.setSigningKey(signKey);
//        return converter;
//    }

@Bean
public JwtAccessTokenConverter accessTokenConverter() {
    JwtAccessTokenConverter converter = new JwtAccessTokenConverter();
    Resource resource = new ClassPathResource("publicKey.txt");
    String publicKey = null;
    try {
        publicKey = IOUtils.toString(resource.getInputStream());
    } catch (final IOException e) {
        throw new RuntimeException(e);
    }
    converter.setVerifierKey(publicKey);
    return converter;
}
```  

### 테스트  
#### access token 취득  
<http://localhost:8081/oauth/authorize?client_id=testClientId&redirect_uri=http://localhost:8081/oauth2/callback&response_type=code&scope=read>  
```json
{
    "access_token": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE2MjYzNzIxMjEsInVzZXJfbmFtZSI6InNpc2lwYXBhMjM5QGdtYWlsLmNvbSIsImF1dGhvcml0aWVzIjpbIlJPTEVfVVNFUiJdLCJqdGkiOiJjODBjZGU5Ni1kYTQ0LTRiODEtYTAwOC1lYWZiM2IxZjY3ZWYiLCJjbGllbnRfaWQiOiJ0ZXN0Q2xpZW50SWQiLCJzY29wZSI6WyJyZWFkIl19.P-ahfq18W-e3OkHWTzt8MdKk3IVLjr1tzpGKQZafG-DYET9OeskPdDy4n9SS-7WdmebWCB-1LWNbKCU8nliB9UtxzRTKimVAYuxXCSV_4oWR3o7wOo7fqtJ2eBFwTC5yn_e9LZbGzVdJ-p1a2VwyW4V7tFzmuCpbxX0qrsXQIdqyIrTIv5tjPCy-3or7Eu4zfgUOWQPWlUl1sGM0PVfLDjqZD-3zDULpAvru4cIvICp0Wx4MRm69TynLwyN2QlaE7ClF5KfiOF0rkqZQLTTvQ9oXoc4qRJw7MQfye6blUaEqATASoATdmpiD_jzLht1Hx_28sqqMYQ7V-3roMRvowA",
    "token_type": "bearer",
    "refresh_token": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VyX25hbWUiOiJzaXNpcGFwYTIzOUBnbWFpbC5jb20iLCJzY29wZSI6WyJyZWFkIl0sImF0aSI6ImM4MGNkZTk2LWRhNDQtNGI4MS1hMDA4LWVhZmIzYjFmNjdlZiIsImV4cCI6MTYyNjM4NjEyMSwiYXV0aG9yaXRpZXMiOlsiUk9MRV9VU0VSIl0sImp0aSI6IjNlNjgwYWJhLTIyZDItNDgxMy1hMzZlLWMzYzRjZTcwYTNkMyIsImNsaWVudF9pZCI6InRlc3RDbGllbnRJZCJ9.i9dlZAjNJ2UeZGbxMBmDOxHLcazNH9FUQrmb7nEc5PlZrHA7mcCQsBTYR2Qm_W1B0H5J_1j-U0uywmYRpY6kjX2DeLZHtNdHcFTYMWTLeCx_cA97h7c-EgEnQIsXPPdy06HASc0EuOpQAaPS640XRZQ0LOcXm7lI8hvEGN32ebmwk0XdZFA9iJgfjLhU-yqpGFmVTqTU-B2Uttyi2MF4fTPZMSTGOjVNXH6Za4CmGYOWNraWPcI34svVLcpg8GosuARXIgQM9xDtD0WpVWRj5YH4YCui7dauB7Y7YQdri8oIITfc755ftipDrk8PwypoBzSn6WrLhUxvS9rh6r_JEw",
    "expires_in": 35999,
    "scope": "read"
}
```  

#### 리소스서버 api 요청
<img src="https://sisipapa.github.io/assets/images/posts/oauth2-postman.PNG" >    

## 참고    
[아빠프로그래머 Spring Boot Oauth2 - ResourceServer](https://daddyprogrammer.org/post/1754/spring-boot-oauth2-resourceserver/)  
[아빠프로그래머 Spring Boot Oauth2 - ResourceServer : 비대칭키를 이용한 서명](https://daddyprogrammer.org/post/1890/spring-boot-oauth2-resourceserver-asymmetric-keys-to-do-the-signing-process/)  

## Github  
<https://github.com/sisipapa/oauth2-ResourceServer.git> 

