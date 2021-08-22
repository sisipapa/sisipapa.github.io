---
layout: post
title: Springboot TDD
category: [tdd]
tags: [intellij, springboot, tdd]
redirect_from:

- /2021/08/18/

---


## TDD란?  
TDD란 Test Driven Development의 약자로 '테스트 주도 개발'이라고 한다. 반복 테스트를 이용한 소프트웨어 방법론으로, 작은 단위의 테스트 케이스를 작성하고 이를 통과하는 코드를 추가하는 단계를 반복하여 구현한다.    
  

## 나의 TDD    
나는 사실 항상 TDD에 맞게 개발을 하고 있지는 못하다. 하지만 시간과 여건이 허락된다면 TDD 방법대로 개발을 해보려고 한다. 내가 TDD을 활용해 개발하는 방식을 정리해 보려고 한다.    
정리는 Member의 등록,조회,삭제 기능 개발을 한다는 전제로 진행할 예정이다. Springboot2.5.4, JPA, QueryDSL, H2, Gradle, Junit5 환경에서 Rest API를 Member의 등록,조회,삭제 API를 만들어 보려고 한다.  

### PreSetting
build.grale  
build.gradle의 dependency 내용이다.  
```text
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    implementation 'org.springframework.boot:spring-boot-starter-web'
    compileOnly 'org.projectlombok:lombok'
    annotationProcessor 'org.projectlombok:lombok'
    testImplementation 'org.springframework.boot:spring-boot-starter-test'

    runtimeOnly  'com.h2database:h2'                                        // H2 DB설정
}
```  

application.properties  
DB 접속정보 및 hibernate.ddl-auto 설정 추가  
```properties
spring.h2.console.enabled=true
spring.h2.console.path=/h2

spring.jpa.show_sql = true
spring.datasource.url=jdbc:h2:tcp://localhost/~/test;
spring.datasource.driverClassName=org.h2.Driver
spring.datasource.username=sa
spring.datasource.password=

spring.jpa.hibernate.ddl-auto=create
```

entity/Member.java
id,name,gender,phone 네개의 필드를 갖는다.  
```java
@Entity
@NoArgsConstructor
@AllArgsConstructor(staticName = "of")
@Builder
@Data
public class Member {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String name;
    private String gender;
    private String phone;
}
```  

DB 초기값세팅  
Member Entity에 5Row 데이터를 저장한다.  
```java
@Component
@RequiredArgsConstructor
public class InitDb {

    private final InitService initService;

    @PostConstruct
    public void init() {
        initService.dbInit();
    }

    @Component
    @Transactional
    @RequiredArgsConstructor
    static class InitService {

        private final EntityManager em;

        public void dbInit() {
            LongStream.rangeClosed(1,5).forEach(index ->{

                String gender = "M";

                if(index % 2 == 0){
                    gender = "W";
                }

                int midNum = (int)(Math.random()*9000)+1000;
                int endNum = (int)(Math.random()*9000)+1000;

                Member initMember = Member.builder().name("name"+index).phone("010-"+midNum+"-"+endNum).gender(gender).build();
                em.persist(initMember);

            });
        }
    }
}
```  

### 1. RepositoryTest 작성  
Repository를 만들기 전에 RepositoryTest를 작성했기 때문에 아래와 같이 Class,Method가 앖다는 에러가 난다.
<img src="https://sisipapa.github.io/assets/images/posts/tdd_repository_create0.png" >   

### 2. Repository 작성
Intellij 기준 에러가 나고있는 @Autowired MemberRepository memberRepository 라인에서 Alt+Enter를 눌러 Class파일을 만든다.  
<img src="https://sisipapa.github.io/assets/images/posts/tdd_repository_create1.png" >    
MemberRepository를 만들 경로확인     
<img src="https://sisipapa.github.io/assets/images/posts/tdd_repository_create2.png" >  
Spring Data JPA의 JpaRepository의 기본 제공하는 메소드만을 사용해서 API를 작성할 예정이기 때문에 MemberRepository는 아래와 같은 Interface로 구현한다.  
```java
import com.sisipapa.study.tdd.entity.Member;
import org.springframework.data.jpa.repository.JpaRepository;

public interface MemberRepository extends JpaRepository<Member, Long> {
}
```  

### 3. ServiceTest 작성
Service를 만들기 전에 ServiceTest를 작성했기 때문에 아래와 같이 Class,Method가 앖다는 에러가 난다.  
<img src="https://sisipapa.github.io/assets/images/posts/tdd_service_create0.png" >  

### 5. Service 작성
Intellij 기준 에러가 나고있는 @Autowired MemberService memberService 라인에서 Alt+Enter를 눌러 Class파일을 만든다.  
<img src="https://sisipapa.github.io/assets/images/posts/tdd_service_create1.png" >  

MemberService 만들 경로확인
<img src="https://sisipapa.github.io/assets/images/posts/tdd_service_create2.png" >  

MemberService Method 생성  
<img src="https://sisipapa.github.io/assets/images/posts/tdd_service_create3.png" >  

### 6. ContollerTest 작성
```java
@SpringBootTest
public class MemberControllerTest {

    @Autowired
    private MemberRepository repository;

    @Autowired
    private ObjectMapper objectMapper;

    @Autowired
    private WebApplicationContext wac;

    private MockMvc mockMvc;

    @BeforeEach
    void beforeEach() {
        mockMvc = MockMvcBuilders
                .webAppContextSetup(wac)
                .alwaysDo(print())
                .build();
    }

    @Test
    void getPerson() throws Exception {
        mockMvc.perform(
                        MockMvcRequestBuilders.get("/api/member/5"))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.name").value("name5"))
                .andExpect(jsonPath("$.id").value(5L));
    }

    @Test
    void creatgePerson() throws Exception {
        MemberDTO dto = MemberDTO.builder().name("name100").gender("M").phone("010-2222-3333").build();

        mockMvc.perform(
                        MockMvcRequestBuilders.post("/api/member")
                                .contentType(MediaType.APPLICATION_JSON_UTF8)
                                .content(toJsonString(dto)))
                .andExpect(status().isCreated());

        Member result = repository.findAll(Sort.by(Sort.Direction.DESC, "id")).get(0);

        assertAll(
                () -> assertThat(result.getName()).isEqualTo("name100"),
                () -> assertThat(result.getGender()).isEqualTo("M"),
                () -> assertThat(result.getPhone()).isEqualTo("010-2222-3333")
        );
    }

    @Test
    void deleteMember() throws Exception {
        mockMvc.perform(
                        MockMvcRequestBuilders.delete("/api/member/1"))
                .andExpect(status().isOk());

        Optional<Member> member = repository.findById(1L);
        assertThat(member).isEqualTo(Optional.empty());
    }

    private String toJsonString(MemberDTO dto) throws JsonProcessingException {
        return objectMapper.writeValueAsString(dto);
    }

}
```  

### 7. Controller 작성  
```java
@RestController
@RequiredArgsConstructor
@RequestMapping("/api/member")
public class MemberController {

    private final MemberService service;

    @GetMapping("/{id}")
    public MemberDTO getPerson(@PathVariable Long id){
        return service.findOneMember(id);
    }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public MemberDTO createPerson(@RequestBody MemberDTO dto){
        return service.saveMember(dto);
    }

    @DeleteMapping("/{id}")
    public void deletePerson(@PathVariable Long id){
        service.deleteMember(id);
    }
}
```  

나의 경우 Repository,Service,Controller 순서로 단위테스트를 진행한다. 실제 업무에서 DB 또는 Mongo에 비지니스 비중이 크다면 Repository 단위테스트만 만들어서 진행하고 Service 단계에서 업무 비지니스가 많다면 Service 단위테스트만 만들어서 진행을 했다. 지금처럼 Repository,Service,Controller 모든 구간별로 단위테스트를 진행해 본적은 없는 것 같아 정리를 한번 해보았다. TDD 정말 좋은 개발 방법론인데 내가 내 업무에 제대로 녹여서 사용하고 있지 못한것 같다. 더 나은 내일을 위해 정리를 한다. 이상........ 

## 참고  
[Taes-k DevLog - TDD Spring 실무에서 적용하기](https://taes-k.github.io/2021/03/19/spring-tdd-practice/)  
[찰나의 개발흔적 - [springboot] 연락처 관리 프로젝트(TDD)](https://daddyprogrammer.org/post/4347/spring-cloud-msa-configuration-server/)  

## Github  
<https://github.com/sisipapa/Springboot-TDD.git>