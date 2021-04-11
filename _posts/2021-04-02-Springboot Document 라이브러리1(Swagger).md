---
layout: post 
title: Springboot Document 라이브러리1(Swagger) 
category: [Document]
tags: [Swagger]
redirect_from:

- /2021/04/02/

---

# Springboot+Spring Data JPA+Querydsl+Swagger Rest API

오늘은 Springboot,JPA,QueryDSL,Gradle 환경에서 Rest API를 만들 예정이다.

개발 진행순서는 아래와 같다.

1. H2 DB설치
2. build.gradle 설정
3. application.properties 설정
4. Configration 설정
5. Rest API 개발(POST,GET,PATCH,DELETE)
6. Swagger Doc 확인

## 1. H2 DB설치

- H2 database 다운로드 후 설치  
  http://www.h2database.com/html/main.html 페이지에서 OS맞는 설치파일을 다운로드 후 설치한다.
- 접속 확인
  http://localhost:8082/login.jsp?jsessionid=3ca85e5c51fb926f441f097c1a6d0434
- JDBC URL을 TCP 모드로 세팅후 연결 jdbc:h2:tcp://localhost/~/DB명

<img src="https://sisipapa.github.io/assets/images/posts/2021-04-02-h2.PNG" >   

## 2. build.gradle 설정

```yaml
plugins {
  id 'org.springframework.boot' version '2.4.4'
  id 'io.spring.dependency-management' version '1.0.11.RELEASE'
  id 'com.ewerk.gradle.plugins.querydsl' version '1.0.10'   // querydsl 설정 추가
  id 'java'
}

dependencies {
    runtimeOnly  'com.h2database:h2'                                        // H2 DB설정
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'  // spring data JPA
    implementation 'com.querydsl:querydsl-jpa'                              // querydsl 설정
    implementation 'org.springdoc:springdoc-openapi-ui:1.2.30'              // openapi
}

def querydslDir = '$buildDir/generated/querydsl'

querydsl {
  jpa = true
  querydslSourcesDir = querydslDir
}

sourceSets {
  main.java.srcDir querydslDir
}

configurations {
  querydsl.extendsFrom compileClasspath
}

compileQuerydsl {
  options.annotationProcessorPath = configurations.querydsl
}
```  

## 3. application.properties 설정

```properties
spring.h2.console.enabled=true
spring.h2.console.path=/h2

spring.jpa.show_sql = true
spring.datasource.url=jdbc:h2:tcp://localhost/~/test;
spring.datasource.driverClassName=org.h2.Driver
spring.datasource.username=sa
spring.datasource.password=

spring.jpa.hibernate.ddl-auto=update

springdoc.api-docs.groups.enabled=true
springdoc.swagger-ui.path=/swagger-ui.html
springdoc.swagger-ui.displayRequestDuration=true
springdoc.swagger-ui.groups-order=DESC
```

## 4. Configuration 설정
```java
@OpenAPIDefinition(
    info = @Info(
    title = "Posts API 명세서",
    description = "Spring Boot를 이용한 Demo 웹 애플리케이션 API입니다.",
    version = "v1",
    contact = @Contact(
    name = "sisipapa",
    email = "sisipapa239@gmail.com"
),
    license = @License(
    name = "Apache 2.0",
    url = "http://www.apache.org/licenses/LICENSE-2.0.html"
)
)
)
@Configuration
public class OpenApiConfig {
    public OpenApiConfig() {
    }

    @Bean
    public GroupedOpenApi postsOpenApi() {
        String[] paths = new String[]{"/v1/post/**"};
        return GroupedOpenApi.builder().setGroup("Posts관련 API").pathsToMatch(paths).build();
    }
}
```

## 5. Rest API 개발(POST,GET,PATCH,DELETE)

```java
@AllArgsConstructor
@RequestMapping("/v1")
@RestController
public class PostsController {

    private final PostsService service;

    /**
     * 등록(Create)
     * @param dto
     * @return
     */
    @PostMapping("/post")
    @Operation(summary = "Post등록",
            description = "Post 레코드를 등록한다.",
            responses = {
                    @ApiResponse(responseCode = "201", description = "post 등록 성공", content = @Content(schema = @Schema(implementation = PostsDTO.class))),
                    @ApiResponse(responseCode = "500", description = "서버오류", content = @Content(schema = @Schema(implementation = ErrorResponse.class)))
            })
    public ResponseEntity<? extends BasicResponse> insertPost(@RequestBody PostsDTO dto) {

        PostsDTO rDto = service.savePost(dto);
        if(rDto == null){
            return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
                    .body(new ErrorResponse("등록 실패", "500"));
        }

        return ResponseEntity.noContent().build();
    }

    /**
     * 조회(Read)
     * @param id
     * @return
     */
    @GetMapping("/post/{id}")
    @Operation(summary = "Post조회",
                description = "id를 이용하여 post 레코드를 조회한다.",
                responses = {
                    @ApiResponse(responseCode = "200", description = "post 조회 성공", content = @Content(schema = @Schema(implementation = PostsDTO.class))),
                    @ApiResponse(responseCode = "404", description = "존재하지 않는 리소스 접근", content = @Content(schema = @Schema(implementation = ErrorResponse.class)))
                })
    public ResponseEntity<? extends BasicResponse> getPost(@Parameter(name = "id", description = "post 의 id", in = ParameterIn.PATH) @PathVariable Long id) {

        PostsDTO dto = service.getPost(id);
        if(dto == null) {
            return ResponseEntity.status(HttpStatus.NOT_FOUND)
                    .body(new ErrorResponse("일치하는 정보가 없습니다. id를 확인해주세요."));
        }
        return ResponseEntity.ok().body(new PostsResponse(dto));
    }

    /**
     * 수정(Update)
     * @param dto
     * @return
     */
    @PatchMapping("/post/{id}")
    @Operation(summary = "Post수정",
            description = "id의 post 레코드를 수정한다.",
            responses = {
                    @ApiResponse(responseCode = "204", description = "컨텐츠 없음", content = @Content(schema = @Schema(implementation = PostsDTO.class))),
                    @ApiResponse(responseCode = "404", description = "존재하지 않는 리소스 접근", content = @Content(schema = @Schema(implementation = ErrorResponse.class)))
            })
    public ResponseEntity<? extends BasicResponse> patchPost(@Parameter(name = "id", description = "post 의 id", in = ParameterIn.PATH) @PathVariable Long id,
                                                        @RequestBody PostsDTO dto) {
        dto.setId(id);
        PostsDTO rDto = service.savePost(dto);
        if(rDto == null) {
            return ResponseEntity.status(HttpStatus.NOT_FOUND)
                    .body(new ErrorResponse("일치하는 정보가 없습니다. id를 확인해주세요."));
        }
        return ResponseEntity.noContent().build();
    }

    /**
     * 삭제(Delete)
     * @param id
     * @return
     */
    @DeleteMapping("/post/{id}")
    @Operation(summary = "Post삭제",
            description = "id의 post 레코드를 삭제한다.",
            responses = {
                    @ApiResponse(responseCode = "204", description = "컨텐츠 없음", content = @Content(schema = @Schema(implementation = PostsDTO.class))),
                    @ApiResponse(responseCode = "404", description = "존재하지 않는 리소스 접근", content = @Content(schema = @Schema(implementation = ErrorResponse.class)))
            })
    public ResponseEntity<? extends BasicResponse> deletePost(@Parameter(name = "id", description = "post 의 id", in = ParameterIn.PATH) @PathVariable Long id) {

        if(!service.deletePost(id)) {
            return ResponseEntity.status(HttpStatus.NOT_FOUND)
                    .body(new ErrorResponse("일치하는 정보가 없습니다. id를 확인해주세요."));
        }
        return ResponseEntity.noContent().build();
    }

}
```

## 6. Swagger Doc 확인(http://localhost:8080/swagger-ui.html)

<img src="https://sisipapa.github.io/assets/images/posts/2021-04-02-main.PNG" >

## 참고 Github
<https://github.com/sisipapa/study1.git>



