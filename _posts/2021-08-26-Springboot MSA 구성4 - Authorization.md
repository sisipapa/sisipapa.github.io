---
layout: post
title: Springboot MSA 구성4 - Authorization
category: [msa]
tags: [msa, springboot]
redirect_from:

- /2021/08/26/

---

오늘은 특정 API에 대한 접근 권한을 적용해서 인가/인증된 사용자만이 API 사용이 가능하도록 구현해 볼 예정이다. Auth 서버 구성은 얼마전에 정리했던 소스를 그대로 가지고 와서 사용할 예정이다.   
아래는 Auth 서버 구성 시 참고  
Springboot Oauth2 - AuthorizationServer - <https://sisipapa.github.io/blog/2021/07/12/Springboot-Oauth2-AuthorizationServer/>  
Springboot Oauth2 - AuthorizationServer2(JWT, DB인증) - <https://sisipapa.github.io/blog/2021/07/13/Springboot-Oauth2-AuthorizationServer2/>  
Auth 서버 구성 Git - <https://github.com/sisipapa/oauth2-AuthorizationServer.git>  
  
## Auth 서버 구성
이전 정리했던 노트내용 중에 Oauth2 Authorization 서버 작성했던 소스를 그대로 가져와서 Auth 서버에 적용을 했다. 소스를 그대로 복사해서 붙여넣기를 하고 변경된 package 구조만 변경했다. 기존에는 Authorization 서버를 하나의 어플리케이션으로 구성을 했었는데 지금은 MSA 구조의 Multi Module로 변경되면서 buiuld.gradle의 내용만 조금 변경되었다.   
  
### build.gradle  
공통부분을 제외하고 Authorization 모듈에서 필요한 모듈만 정리. 나머지 소스는 [Auth 서버 구성 전체소스](https://github.com/sisipapa/oauth2-AuthorizationServer.git)를 참고하면 된다.  
```properties
project(':Auth') {
    dependencies {
        implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
        implementation 'org.springframework.boot:spring-boot-starter-web'
        runtimeOnly 'com.h2database:h2'

        implementation 'com.google.code.gson:gson'
        implementation 'org.springframework.cloud:spring-cloud-starter-security'
        implementation 'org.springframework.cloud:spring-cloud-starter-oauth2'
    }

    dependencyManagement {
        imports {
            mavenBom "org.springframework.cloud:spring-cloud-dependencies:${springCloudVersion}"
        }
    }
}
```

## Gateway 모듈
### bootstrap.yml 
zuul.sensitiveHeaders 속성을 추가한다. sensitiveHeaders를 설정하여 내부에서 사용되는 헤더값이 노출되는 것을 막을 수 있다. Cookie, Set-Cookie, Authorization은 zuul의 기본값이며, sensitiveHeaders를 사용하지 않으려면 명시적으로 값을 비워둬야한다.  
```yaml
server:
  port: 9100
spring:
  application:
    name: gateway
  cloud:
    config:
      fail-fast: true
      discovery:
        service-id: config
        enabled: true
    # uri: http://localhost:9000
management:
  endpoints:
    web:
      exposure:
        include: refresh

zuul:
  sensitiveHeaders: Cookie,Set-Cookie
```  

### Config 추가
EnableResourceServer 어노테이션은 API 서버를 설정하기 위한 부분이다. MSA구조에서는 개별 API 서버마다 EnableResourceServer 어노테이션을 설정하는 것이 아니라 gateway 서버에 설정한다.   
Resource 서버의 /v1/member/* URI 패턴은 인증된 사용자만 호출이 가능하도록 설정한다.   
```java
@Configuration
@EnableResourceServer
public class Oauth2ResourceServerConfig extends ResourceServerConfigurerAdapter {

    @Override
    public void configure(HttpSecurity http) throws Exception {
        http.headers().frameOptions().disable();
        http.authorizeRequests()
                .antMatchers("/v1/member/*").authenticated();
    }

    @Bean
    public TokenStore tokenStore() {
        return new JwtTokenStore(accessTokenConverter());
    }

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
}
```  

### 공개키 추가  
인증서버는 리소스 서버에게 공개키를 공유하고 함호화해서 데이터를 전달한다. 암호화된 데이터와 공개키가 노출 되더라도 비밀키가 없이 데이터를 복호화 할 수 없기 때문에 보안이 높다.  
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

## 테스트  
### AccessToken 생성
인증서버의 /oauth/token API를 통해 access_token을 얻는다.  
```java
@SpringBootTest
class Oauth2ControllerTest {

    Logger log = (Logger) LoggerFactory.getLogger(Oauth2ControllerTest.class);

    @Autowired
    private Gson gson;
    @Autowired
    private RestTemplate restTemplate;

    @Test
    void callbackSocial(){
        String credentials = "testClientId:testSecret";
        String encodedCredentials = new String(Base64.encodeBase64(credentials.getBytes()));

        HttpHeaders headers = new HttpHeaders();
        headers.setContentType(MediaType.APPLICATION_FORM_URLENCODED);
        headers.add("Authorization", "Basic " + encodedCredentials);

        MultiValueMap<String, String> params = new LinkedMultiValueMap<>();
        params.add("refresh_token", "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VyX25hbWUiOiJzaXNpcGFwYTIzOUBnbWFpbC5jb20iLCJzY29wZSI6WyJyZWFkIl0sImF0aSI6ImU0ODM4MmEzLTk2OWMtNGFlYy05MThkLWQyNzc4ZTEzNzg0NiIsImV4cCI6MTYzMDE4NDkzMSwiYXV0aG9yaXRpZXMiOlsiUk9MRV9VU0VSIl0sImp0aSI6ImIyMjg4ZTg0LTEwNTctNDI5Yy1hNWY3LWQwMGQ4ZDU0ZjhkOSIsImNsaWVudF9pZCI6InRlc3RDbGllbnRJZCJ9.VQAyQ-y-IV6s-a-vHZ_h-wRbU_R03mwkb_XLKaPtQCll4KTOOuuLyHYvsiDcR7RblITbBBj4gnRzUdSNG0M87dAP3NYAarLq_DF-MHVrxalj7o_PbIoXmgrI7geO-51bvoj0bApd9Rgo9dF-yIKTUp7BkCJeJaIpzKg_TXO0KmaVCpltwenMnDUsbbcME9dLoMOquQ28w_Rcm46mdceT-vfq4gpFmy_fBhDVPc4702YyuKbfb0BxGDCZ_SXP4G7ykWqt7IuMdjuBrtPBqKZCrolQiVI3Dcsk3n9UADx6wfwwx-UBRtNtVSKpJMl3LbWAjxSnFqjVZskdFgCrvK68BA");
        params.add("grant_type", "refresh_token");
//        params.add("code", code);
//        params.add("grant_type", "authorization_code");
//        params.add("redirect_uri", "http://localhost:8081/oauth2/callback");
        HttpEntity<MultiValueMap<String, String>> request = new HttpEntity<>(params, headers);
        ResponseEntity<String> response = restTemplate.postForEntity("http://localhost:8081/oauth/token", request, String.class);
        if (response.getStatusCode() == HttpStatus.OK) {
            OAuthToken token = gson.fromJson(response.getBody(), OAuthToken.class);
            System.out.println("###############################################################################################################################");
            log.info("### access_token : " + token.getAccess_token());
            log.info("### refresh_token : " + token.getRefresh_token());
            log.info("### token_type : " + token.getToken_type());
            log.info("### scope : " + token.getScope());
            log.info("### expires_in : " + token.getExpires_in());
            System.out.println("###############################################################################################################################");
        }
    }
}
```   
단위테스트 결과 로그확인(access_token을 복사)    
```text
###############################################################################################################################
2021-08-28 21:07:24.398  INFO 15892 --- [    Test worker] c.s.s.m.a.c.Oauth2ControllerTest         : ### access_token : eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE2MzAxODg0NDQsInVzZXJfbmFtZSI6InNpc2lwYXBhMjM5QGdtYWlsLmNvbSIsImF1dGhvcml0aWVzIjpbIlJPTEVfVVNFUiJdLCJqdGkiOiJiOWMyNTE3Ni03MzczLTRjN2YtYjQ4NC0zNjM5NjRhNmFhZDQiLCJjbGllbnRfaWQiOiJ0ZXN0Q2xpZW50SWQiLCJzY29wZSI6WyJyZWFkIl19.hCWbWLJGlM4SFzy3AchClbDND3Tb5OT0l5kefMPw8fkXkIQB0Q6H6ZzCdE6_iuGWLGWoUYdm7chC6cwiflgUZ0T7zkNJEsOf4nY6lLh7oXwXBQ4aamUQlwLoUj0sFOriLOA7DLGywRoTglhXaZ02ROiphHDBq4jmzktz-v0zdRPr82ibIWiiR65yTYcLl2KpI8NNyI8HcrD3BhOJ1QqTD3PyqfvjIa1rSLYfQHqiDqbnu8ZvCNam6wXcc1EBvm69RUcEP97NjVjxKNUZzGhE0ulWHjxR_9c_zbJpTHY_tJ-iVd0pCQ6JK9uGK7TZHlMwQz7yZ6oJBkStoZehRMpMQg
2021-08-28 21:07:24.398  INFO 15892 --- [    Test worker] c.s.s.m.a.c.Oauth2ControllerTest         : ### refresh_token : eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VyX25hbWUiOiJzaXNpcGFwYTIzOUBnbWFpbC5jb20iLCJzY29wZSI6WyJyZWFkIl0sImF0aSI6ImI5YzI1MTc2LTczNzMtNGM3Zi1iNDg0LTM2Mzk2NGE2YWFkNCIsImV4cCI6MTYzMDE4NDkzMSwiYXV0aG9yaXRpZXMiOlsiUk9MRV9VU0VSIl0sImp0aSI6ImIyMjg4ZTg0LTEwNTctNDI5Yy1hNWY3LWQwMGQ4ZDU0ZjhkOSIsImNsaWVudF9pZCI6InRlc3RDbGllbnRJZCJ9.WSTm2-sm6ZZN7AlKXcazL_jFiOLzBCbs8DTTQFEtoW4X5naIgy8pReQ_uYCj5LJdFRgJFPMthOd4q4f3WeOE_2SyWuSF38kutG_gxE95MScfdVHPeuh70pem13p2Ms9AeFxD5GN27qazCSXQXEl-8IZWPq05l_q8TzxTE2pbvW1vjz3Jpmv6K1pfaQHxqjEbxcQ82S5m1-WuximQW52EJhQNEmC7_lDLSh5aJmO51JkSNrkgvE0koEhsyGFm4C5uZYR3-rGsVimYAE6ILzD_MWK37jhtXGnTWf5zLnYygyBvurLn-ANKS--rniEjZMNLg4Vv8ip97kWR3oOuef-0Eg
2021-08-28 21:07:24.398  INFO 15892 --- [    Test worker] c.s.s.m.a.c.Oauth2ControllerTest         : ### token_type : bearer
2021-08-28 21:07:24.399  INFO 15892 --- [    Test worker] c.s.s.m.a.c.Oauth2ControllerTest         : ### scope : read
2021-08-28 21:07:24.402  INFO 15892 --- [    Test worker] c.s.s.m.a.c.Oauth2ControllerTest         : ### expires_in : 35999
###############################################################################################################################
```

### /v1/member/health API 호출
#### Authorization header 설정 안한 경우  
```json
GET http://localhost:9100/v1/member/health

HTTP/1.1 401 
Cache-Control: no-store
Pragma: no-cache
WWW-Authenticate: Bearer realm="oauth2-resource", error="unauthorized", error_description="Full authentication is required to access this resource"
X-Content-Type-Options: nosniff
X-XSS-Protection: 1; mode=block
Content-Type: application/json
Transfer-Encoding: chunked
Date: Sat, 28 Aug 2021 12:10:13 GMT
Keep-Alive: timeout=60
Connection: keep-alive

{
  "error": "unauthorized",
  "error_description": "Full authentication is required to access this resource"
}

Response code: 401; Time: 264ms; Content length: 102 bytes
```    

#### Authorization header 설정 했을 경우  
```json
GET http://localhost:9100/v1/member/health

HTTP/1.1 200 
Date: Sat, 28 Aug 2021 12:09:05 GMT
Keep-Alive: timeout=60
X-Content-Type-Options: nosniff
X-XSS-Protection: 1; mode=block
Cache-Control: no-cache, no-store, max-age=0, must-revalidate
Pragma: no-cache
Expires: 0
Content-Type: application/json
Transfer-Encoding: chunked
Connection: keep-alive

MemberController running

Response code: 200; Time: 209ms; Content length: 24 bytes
```  

테스트를 통해 /v1/member/* URI 패턴은 Authorization Bearer 타입의 access_token이 없을 경우 401 HTTP 응답코드의 에러메세지를 응답한다.  

## 참고  
[Spring MSA (4) - 인증서비스](https://taes-k.github.io/2019/06/20/spring-msa-4/)  

## Github    
<https://github.com/sisipapa/Springboot-MSA.git>  
