---
layout: post
title: Springboot Oauth2 - AuthorizationServer2(JWT, DB인증)
category: [oauth2]
tags: [springboot, oauth2]
redirect_from:

- /2021/07/13/

---

이전에는 inMemory 방식으로 서버에서 하드코딩된 인증정보를 통해 인증을 진행 했던 부분을 DB를 사용해 처리할 수 있도록 수정할 예정이다.  
[아빠프로그래머 Spring Boot Oauth2 - AuthorizationServer : DB처리,JWT토큰 방식 적용](https://daddyprogrammer.org/post/1287/spring-oauth2-authorizationserver-database/) 블로그를 참고해서 진행할 예정이다.

## 변경사항
- 클라이언트 DB 인증
- 로그인 사용자 DB 인증
- 인증 및 토큰정보 DB 인증

## 클라이언트 DB인증
resources > db 디렉토리 하위에 schema.sql 중 아래 쿼리를 H2 DB에서 실행한다.    
oauth_client_details 테이블은 인증 전 인가된 client인지를 확인하기 위한 테이블이다.

클라이언트 인가를 위한 테이블 생성 및 클라이언트 데이터 Insert
```sql  
create table IF NOT EXISTS oauth_client_details (
  client_id VARCHAR(256) PRIMARY KEY,
  resource_ids VARCHAR(256),
  client_secret VARCHAR(256),
  scope VARCHAR(256),
  authorized_grant_types VARCHAR(256),
  web_server_redirect_uri VARCHAR(256),
  authorities VARCHAR(256),
  access_token_validity INTEGER,
  refresh_token_validity INTEGER,
  additional_information VARCHAR(4096),
  autoapprove VARCHAR(256)
);

insert into oauth_client_details(client_id, resource_ids,client_secret,scope,authorized_grant_types,web_server_redirect_uri,authorities,access_token_validity,refresh_token_validity,additional_information,autoapprove)
values('testClientId',null,'{bcrypt}$2a$10$MtkK9P2c4GC4isH1GujIF.D98iO1j1BfyJxVwtHnhf8LYHswwghjO','read,write','authorization_code,refresh_token','http://localhost:8081/oauth2/callback','ROLE_USER',36000,50000,null,null);
```  

Oauth2AuthorizationConfig 수정
```java
@RequiredArgsConstructor
@Configuration
@EnableAuthorizationServer
public class Oauth2AuthorizationConfig extends AuthorizationServerConfigurerAdapter {

    private final PasswordEncoder passwordEncoder;
    private final DataSource dataSource;

    /**
     * 클라이언트 정보를 DB 정보로 인증
     */
    @Override
    public void configure(ClientDetailsServiceConfigurer clients) throws Exception {
        clients.jdbc(dataSource).passwordEncoder(passwordEncoder);
    }
}
```  

## 로그인 사용자 DB인증
### User Entity 생성
```java  
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
### User Repository 생성
```java
import com.sisipapa.oauth2.model.User;
import org.springframework.data.jpa.repository.JpaRepository;

import java.util.Optional;

public interface UserJpaRepository extends JpaRepository<User, Long> {
    Optional<User> findByUid(String email);
}
```  

### 로그인 유효성 검증을 위한 AuthenticationProvider 생성
```java
import com.sisipapa.oauth2.model.User;
import com.sisipapa.oauth2.repository.UserJpaRepository;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.security.authentication.AuthenticationProvider;
import org.springframework.security.authentication.BadCredentialsException;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.Authentication;
import org.springframework.security.core.userdetails.UsernameNotFoundException;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.stereotype.Component;

@Slf4j
@RequiredArgsConstructor
@Component
public class CustomAuthenticationProvider implements AuthenticationProvider {

    private final PasswordEncoder passwordEncoder;

    private final UserJpaRepository userJpaRepository;

    @Override
    public Authentication authenticate(Authentication authentication) {

        String name = authentication.getName();
        String password = authentication.getCredentials().toString();

        User user = userJpaRepository.findByUid(name).orElseThrow(() -> new UsernameNotFoundException("user is not exists"));

        if (!passwordEncoder.matches(password, user.getPassword()))
            throw new BadCredentialsException("password is not valid");

        return new UsernamePasswordAuthenticationToken(name, password, user.getAuthorities());
    }

    @Override
    public boolean supports(Class<?> authentication) {
        return authentication.equals(
                UsernamePasswordAuthenticationToken.class);
    }
}
```  

### SpringSecurity 관련 Config 수정
```java
import com.sisipapa.oauth2.provider.CustomAuthenticationProvider;
import lombok.RequiredArgsConstructor;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.authentication.builders.AuthenticationManagerBuilder;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.WebSecurityConfigurerAdapter;

@RequiredArgsConstructor
@Configuration
public class SecurityConfig extends WebSecurityConfigurerAdapter {

    private final CustomAuthenticationProvider authenticationProvider;

    @Override
    protected void configure(AuthenticationManagerBuilder auth) {
        auth.authenticationProvider(authenticationProvider);
    }

    @Override
    protected void configure(HttpSecurity security) throws Exception {
        security
                .csrf().disable()
                .headers().frameOptions().disable()
                .and()
                .authorizeRequests().antMatchers("/oauth/**", "/oauth/token", "/oauth2/callback", "/h2-console/*").permitAll()
                .and()
                .formLogin().and()
                .httpBasic();
    }
}
```  

### 로그인 사용자 DB인증을 위한 테스트 데이터 등록
지금까지 작업한 Server를 실행하고 아래 클릭해서 user 테이블에 등록한 테스트 데이터로 로그인을 하면 정상 로그인을 확인할 수 있다.  
[TEST URI 클릭](http://localhost:8081/oauth/authorize?client_id=testClientId&redirect_uri=http://localhost:8081/oauth2/callback&response_type=code&scope=read)
```java
import com.sisipapa.oauth2.model.User;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.security.crypto.password.PasswordEncoder;

import java.util.Collections;

@SpringBootTest
class UserJpaRepositoryTest {
    @Autowired
    private UserJpaRepository userJpaRepository;

    @Autowired
    private PasswordEncoder passwordEncoder;

    @Test
    public void insertNewUser() {
        userJpaRepository.save(User.builder()
                .uid("sisipapa239@gmail.com")
                .password(passwordEncoder.encode("1234"))
                .name("sisipapa")
                .roles(Collections.singletonList("ROLE_USER"))
                .build());
    }
}
```  

## 인증 및 토큰정보 DB 인증
### 토큰정보 DB 관리를 위한 테이블 생성 sql 실행
```sql
create table IF NOT EXISTS oauth_client_token (
    token_id VARCHAR(256),
    token LONGVARBINARY,
    authentication_id VARCHAR(256) PRIMARY KEY,
    user_name VARCHAR(256),
    client_id VARCHAR(256)
    );

create table IF NOT EXISTS oauth_access_token (
    token_id VARCHAR(256),
    token LONGVARBINARY,
    authentication_id VARCHAR(256) PRIMARY KEY,
    user_name VARCHAR(256),
    client_id VARCHAR(256),
    authentication LONGVARBINARY,
    refresh_token VARCHAR(256)
    );

create table IF NOT EXISTS oauth_refresh_token (
    token_id VARCHAR(256),
    token LONGVARBINARY,
    authentication LONGVARBINARY
    );

create table IF NOT EXISTS oauth_code (
    code VARCHAR(256), authentication LONGVARBINARY
    );

create table IF NOT EXISTS oauth_approvals (
    userId VARCHAR(256),
    clientId VARCHAR(256),
    scope VARCHAR(256),
    status VARCHAR(10),
    expiresAt TIMESTAMP,
    lastModifiedAt TIMESTAMP
    );
```  

### Token정보 DB 관리를 위한 설정추가(Oauth2AuthorizationConfig Config 추가)
```java
    /**
     * 토큰 정보를 DB 관리
     * @return
     */
    @Override
    public void configure(AuthorizationServerEndpointsConfigurer endpoints) throws Exception {
        endpoints.tokenStore(new JdbcTokenStore(dataSource));
    }
```  

### Token정보 DB 관리가 아닌 JWT으로 변경
JdbcTokenStore가 아닌 jwtAccessTokenConverter를 사용하도록 설정한다. JWT를 사용하게 되면 토큰 자체로 인증정보가 관리가 되어 DB테이블을 사용하지 않게 된다.  
[JWT Token 발급 테스트 URI 클릭](http://localhost:8081/oauth/authorize?client_id=testClientId&redirect_uri=http://localhost:8081/oauth2/callback&response_type=code&scope=read)

```java
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
        endpoints.accessTokenConverter(jwtAccessTokenConverter());
    }

    /**
     * jwt converter를 등록
     *
     * @return
     */
    @Bean
    public JwtAccessTokenConverter jwtAccessTokenConverter() {
        return new JwtAccessTokenConverter();
    }
```  

### refresh_token을 이용한 access_token 재발급
refresh_token이 정상인지 확인을 위해서는 회원정보를 조회해 봐야 하기때문에 Oauth2AuthorizationConfig에 userDetailsService를 설정해준다.
```java
    /**
     * 토큰 발급 방식을 JWT 토큰 방식으로 변경한다. 이렇게 하면 토큰 저장하는 DB Table은 필요가 없다.
     *
     * @param endpoints
     * @throws Exception
     */
    @Override
    public void configure(AuthorizationServerEndpointsConfigurer endpoints) throws Exception {
        super.configure(endpoints);
//        endpoints.accessTokenConverter(jwtAccessTokenConverter());
        endpoints.accessTokenConverter(jwtAccessTokenConverter()).userDetailsService(userDetailService);
    }
```

### jwt signkey 세팅(application.yaml 파일에 추가)
이전 테스트까지는 signKey를 설정하지 않아서 임의의 키로 암호화가 되었지만 refresh_token 재발급을 위해서는 복호화가 되어야 하는데 이때 signKey가 필요하기 때문에 설정이 필요하다.
```yaml
  security:
    oauth2:
      jwt:
        signkey: 123@#$
```  

### Oauth2AuthorizationConfig의 JwtAccessTokenConverter에 signKey를 추가
```java

    @Value("${security.oauth2.jwt.signkey}")
    private String signKey;

    /**
     * jwt converter를 등록
     *
     * @return
     */
    @Bean
    public JwtAccessTokenConverter jwtAccessTokenConverter() {
//        return new JwtAccessTokenConverter();
        JwtAccessTokenConverter converter = new JwtAccessTokenConverter();
        converter.setSigningKey(signKey);
        return converter;
    }
```  

### refresh 토큰을 위한 Controller API 추가
로그인 할 때 발급받은 refresh_token을 아래 API의 파라미터로 넣고 호출하면 새로운 refresh_token이 발급된다.  
[refresh 토큰 테스트 클릭](http://localhost:8081/oauth2/token/refresh?refreshToken=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VyX25hbWUiOiJzaXNpcGFwYTIzOUBnbWFpbC5jb20iLCJzY29wZSI6WyJyZWFkIl0sImF0aSI6IjUyOGVkMDliLTIwN2ItNDM2NS1hNTgxLWQyNzEzYmU2OWViNiIsImV4cCI6MTYyNjM2OTYzOCwiYXV0aG9yaXRpZXMiOlsiUk9MRV9VU0VSIl0sImp0aSI6IjY5OTU0ODJkLTAwMjUtNDg4My1iYTQ2LWFiZWI2ZGE0YmVmNiIsImNsaWVudF9pZCI6InRlc3RDbGllbnRJZCJ9.c0Zv4wu85cSgwfLBbfZeeXS3e87LFLrYz3FIde7sBo0)
```java
    @GetMapping(value = "/token/refresh")
    public OAuthToken refreshToken(@RequestParam String refreshToken) {

        String credentials = "testClientId:testSecret";
        String encodedCredentials = new String(Base64.encodeBase64(credentials.getBytes()));

        HttpHeaders headers = new HttpHeaders();
        headers.setContentType(MediaType.APPLICATION_FORM_URLENCODED);
        headers.add("Accept", MediaType.APPLICATION_JSON_VALUE);
        headers.add("Authorization", "Basic " + encodedCredentials);

        MultiValueMap<String, String> params = new LinkedMultiValueMap<>();
        params.add("refresh_token", refreshToken);
        params.add("grant_type", "refresh_token");
        HttpEntity<MultiValueMap<String, String>> request = new HttpEntity<>(params, headers);
        ResponseEntity<String> response = restTemplate.postForEntity("http://localhost:8081/oauth/token", request, String.class);
        if (response.getStatusCode() == HttpStatus.OK) {
            return gson.fromJson(response.getBody(), OAuthToken.class);
        }
        return null;
    }
```  

## 참고
[아빠프로그래머 Spring Boot Oauth2 - AuthorizationServer : DB처리,JWT토큰 방식 적용](https://daddyprogrammer.org/post/1287/spring-oauth2-authorizationserver-database/)

## Github
<https://github.com/sisipapa/oauth2-AuthorizationServer.git> 

