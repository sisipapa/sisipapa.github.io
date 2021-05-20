---
layout: post 
title: Springboot GraphQL Example
category: [graphql]
tags: [graphql, springboot]
redirect_from:

- /2021/05/20/

---

## GraphQL
GraphQL은 REST API의 대안으로 Facebook에서 제시한 새로운 Web API 이다. 오늘은 GraphQL이 무엇인지 급하게 확인을 해보기 위해 아래 참고 블로그를 그대로 따라해 보면서 개념을 파악해 보려고 한다. 이제 Springboot + JPA + H2 환경에서 GraphQL서버를 구축시작!!  

### Dependency 설정(build.gradle)
```text  
...  
dependencies {
    compileOnly 'org.projectlombok:lombok'
    annotationProcessor 'org.projectlombok:lombok'
    testImplementation 'org.springframework.boot:spring-boot-starter-test'

    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'

    implementation group: 'com.graphql-java-kickstart', name: 'graphql-spring-boot-starter', version: '11.0.0'
    implementation group: 'com.graphql-java-kickstart', name: 'graphql-java-tools', version: '11.0.0'

    implementation group: 'com.h2database', name: 'h2', version: '1.4.200'
}
...  
```  

### application.yml 설정  
참고 블로그와는 다르게 8080 포트로 로컬에서 배치테스트를 하는 중에 GraphQL Example을 만들게 되서 port정보를 8081로 변경하고 h2 DB 접속 url 정보를 변경했습니다.  
```yaml  
server.port: 8081

spring:
  datasource:
    #url: jdbc:h2:mem:testdb;DB_CLOSE_ON_EXIT=FALSE
    url: jdbc:h2:tcp://localhost/~/test;
    username: sa
    password:
    driverClassName: org.h2.Driver
    hikari:
      maximum-pool-size: 24
  h2:
    console:
      enabled: true
  jpa:
    generate-ddl: true
    hibernate:
      ddl-auto: update
    properties:
      hibernate:
        dialect: org.hibernate.dialect.H2Dialect
        format_sql: true
    show-sql: true
graphql:
  servlet:
    enabled: true
    mapping: /graph
    corsEnabled: false
    cors:
      allowed-origins: http://localhost:8081
      allowed-methods: GET, HEAD, POST, PATCH
    exception-handlers-enabled: true
    context-setting: PER_REQUEST_WITH_INSTRUMENTATION
    async-mode-enabled: true
  tools:
    schema-location-pattern: "**/*.graphqls"
    introspection-enabled: true
```  

### Domain 만들기  
기존에 H2 DB에 사용중인 Member Entity명이 중복되어 Domain 클래스 명을 변경했습니다.  
Member.java => MemberGraphQL.java  
Role.java => RoleGraphQL.java  

```java  
@Builder
@Entity
@Setter
@Getter
@AllArgsConstructor
@NoArgsConstructor
public class MemberGraphQL {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private int id;

    private String loginId;

    private String password;

    private String name;

    @OneToMany
    @JoinColumn(name = "memberId", referencedColumnName = "id", insertable = false, updatable = false)
    private List<RoleGraphQL> role;
}
```  
```java  
@Entity
@Setter
@Getter
@AllArgsConstructor
@NoArgsConstructor
public class RoleGraphQL {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private int id;

    private int memberId;

    private String role;
}
```  

### DTO 생성
참고 블로그의 DTO와 동일하게 생성했습니다.  
MemberDto.java  
```java  
@Setter
@Getter
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class MemberDto {

    private Integer id;

    private String loginId;

    private String password;

    private String name;

    private List<RoleDto> roles;

    public static List<MemberDto> from(Collection<MemberGraphQL> entities) {
        return entities.stream().map(MemberDto::from).collect(Collectors.toList());
    }

    public static MemberDto from(MemberGraphQL entity) {
        return MemberDto.builder()
                .id(entity.getId())
                .loginId(entity.getLoginId())
                .password(entity.getPassword())
                .name(entity.getName())
                .roles(RoleDto.from(entity.getRole()))
                .build();
    }
}
```  
RoleDto.java  
```java  
@Setter
@Getter
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class RoleDto {
    private Integer id;

    private Integer memberId;

    private String role;

    public static List<RoleDto> from(Collection<RoleGraphQL> entities) {
        if(entities == null) {
            return null;
        }
        return entities.stream().map(RoleDto::from).collect(Collectors.toList());
    }

    public static RoleDto from(RoleGraphQL entity) {
        return RoleDto.builder()
                .id(entity.getId())
                .memberId(entity.getMemberId())
                .role(entity.getRole())
                .build();
    }
}
```  

### Repository 생성
참고 블로그의 Repository와 동일하게 생성했습니다.    
MemberRepository.java  
```java  
@Repository
public interface MemberRepository extends JpaRepository<MemberGraphQL, Integer> {}
```  
RoleRepository.java  
```java  
@Repository
public interface RoleRepository extends JpaRepository<RoleGraphQL, Integer> {
    RoleGraphQL findByMemberId(int memberId);
}
```
### Mutation 생성  
```java  
@Component
@RequiredArgsConstructor
@Transactional
public class MemberMutation implements GraphQLMutationResolver {

    private final MemberRepository memberRepository;

    private final RoleRepository roleRepository;

    public MemberDto createMember(MemberDto memberDto) {
        MemberGraphQL member = memberRepository.save(MemberGraphQL.builder()
                .loginId(memberDto.getLoginId())
                .name(memberDto.getName())
                .password(memberDto.getPassword())
                .build());
        return MemberDto.from(member);
    }

    public Boolean deleteMember(int id){
        Optional<MemberGraphQL> optionalMember = memberRepository.findById(id);
        RoleGraphQL role = roleRepository.findByMemberId(id);
        if(optionalMember.isPresent()) {
            roleRepository.delete(role);
            memberRepository.delete(optionalMember.get());
        }
        return true;
    }
}
```

### Query 생성
```java  
@Component
@RequiredArgsConstructor
@Transactional
public class MemberQuery implements GraphQLQueryResolver {

    private final MemberRepository memberRepository;

    public MemberDto getMember(int id) {
        MemberGraphQL member = memberRepository.findById(id)
                .orElse(null);
        return MemberDto.from(member);
    }
}
```

### DB 초기데이터 저장  
H2 Console에 접속해서 아래 Insert 쿼리 수행  
```sql  
INSERT INTO member_graphql (id, login_id, password, name) VALUES
(1, 'member', 'password', '멤버'),
(2, 'account', 'password', '계정'),
(3, 'user', 'password', '사용자');

INSERT INTO role_graphql (id, member_id, role) VALUES
(1, 1, 'ROLE_ADMIN'),
(2, 1, 'ROLE_MEMBER'),
(3, 2, 'ROLE_MEMBER'),
(4, 3, 'ROLE_MANAGER');
```  

### Postman Test  
Postman > Body >  raw(JSON) 호출  
URL - localhost:8081/graph
Body
```json
{  
    "query": "query { getMember(id: 1) {id login_id roles {id role}}}"
}
```  
<img src="https://sisipapa.github.io/assets/images/posts/postman-body.PNG" >  

Postman > Body >  GraphQL 호출    
URL - localhost:8081/graph  
QUERY  
```text
query { getMember(id: 1) {id login_id roles {id role}}}
```
<img src="https://sisipapa.github.io/assets/images/posts/postman-graphql.PNG" >  

오늘은 GraphQL이 Springboot에서 어떤 설정으로 어떻게 실행되는지 확인만을 해보았다. 개념에 대한 학습을 해서 이번 정리에 대한 보강을 해야겠다.   

## Github
<https://github.com/sisipapa/GraphQL.git>

## 참고  
[spring-boot-graphql 설정하기](https://myborn.tistory.com/m/13?category=869291)  
[쿼리 & 뮤테이션](https://graphql-kr.github.io/learn/queries/)  

