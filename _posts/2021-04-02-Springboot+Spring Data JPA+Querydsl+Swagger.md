---
layout: post
title: Springboot+Spring Data JPA+Querydsl+Swagger
category: [Springboot]
tags: [JPA,QueryDSL,Swagger,RestAPI]
redirect_from:
  - /2021/04/02/
---

# Springboot+Spring Data JPA+Querydsl+Swagger Rest API  

앞으로 내가 개발을 하면서 경험했던 다양한 환경과 설정들을 간단하게 정리를 해보려고 한다.  
오늘은 Springboot,JPA,QueryDSL,Gradle 환경에서 Rest API를 만들 예정이다.  

개발 진행순서는 아래와 같다.
1. H2 DB설치
2. build.gradle 설정
3. application.properties 설정
4. Rest API 개발(POST,GET,PATCH,DELETE)
5. Swagger Doc 확인

## 1. H2 DB설치  
- H2 database 다운로드 후 설치  
  http://www.h2database.com/html/main.html 페이지에서 OS맞는 설치파일을 다운로드 후 설치한다.
- 접속 확인
  http://localhost:8082/login.jsp?jsessionid=3ca85e5c51fb926f441f097c1a6d0434  
- JDBC URL을 TCP 모드로 세팅후 연결
  jdbc:h2:tcp://localhost/~/DB명

<img src="https://sisipapa.github.io/assets/images/posts/2021-04-02-h2.PNG" >   

## 2. build.gradle 설정  
- h2,jpa,querydsl,springdoc-openapi 설정추가  
```yaml
dependencies {
    ... 생략
    runtimeOnly  'com.h2database:h2'                                        // H2 DB설정
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'  // spring data JPA
    implementation 'com.querydsl:querydsl-jpa'                              // querydsl 설정
    implementation 'org.springdoc:springdoc-openapi-ui:1.2.30'              // openapi
    ... 생략
}
```

- querydsl generate 설정추가  
```yaml
plugins {
  id 'org.springframework.boot' version '2.4.4'
  id 'io.spring.dependency-management' version '1.0.11.RELEASE'
  id 'com.ewerk.gradle.plugins.querydsl' version '1.0.10'   // querydsl 설정 추가
  id 'java'
}

//querydsl 추가 시작
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
//querydsl 추가 끝
```

## 3. application.properties   
```properties
# H2 설정
spring.h2.console.enabled=true
spring.h2.console.path=/h2

# sql 보기
spring.jpa.show_sql = true
# h2 문법을 mysql로 변경
#spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.MySQL5InnoDBDialect
#spring.jpa.properties.hibernate.dialect.storage_engine=innodb
spring.datasource.url=jdbc:h2:tcp://localhost/~/test;
spring.datasource.driverClassName=org.h2.Driver
spring.datasource.username=sa
spring.datasource.password=

#create : 서버 시작에 모든 테이블 생성
#create-drop : 서버 시작에 모든 테이블 생성, 서버 종료에 테이블 삭제
#update : 서버 시작에 변경된 내용 반영. 테이블이 없으면 생성
#validate : 서버 시작에 엔티티와 테이블 비교, 다르면 종료
#none : 아무 처리하지 않음
spring.jpa.hibernate.ddl-auto=update


springdoc.api-docs.groups.enabled=true
springdoc.swagger-ui.path=/swagger-ui.html
springdoc.swagger-ui.displayRequestDuration=true
springdoc.swagger-ui.groups-order=DESC
```

## 4. Rest API 개발(POST,GET,PATCH,DELETE)
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
## 5. Swagger Doc 확인(http://localhost:8080/swagger-ui.html)  
<img src="https://sisipapa.github.io/assets/images/posts/2021-04-02-main.PNG" >

## 6. Github  
<https://github.com/sisipapa/Springboot-JPA-QueryDSL-H2.git>


다음에는 Springboot + Spring Security + Oauth2 + JWT를 활용한 OAuth2 Server 환경을 구성해 봐야겠다.

