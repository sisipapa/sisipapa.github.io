---
layout: post 
title: Springboot MongoDB
category: [mongo]
tags: [mongo, springboot]
redirect_from:

- /2021/07/27/

---

며칠전 무료 GCP 계정을 만들어서 Mongo Cluster를 구성을 해봤다. 그래서 오늘은 간단하게 Springboot 와 MongoDB 연동하는 예제를 만들어 보려고 한다.  

## PreSetting  

### mongos가 설치된 서버에 접속 후 Database,User를 생성 
```shell
[mongo@mongos ~]$ mongo --port 27017
MongoDB shell version v4.2.15
connecting to: mongodb://127.0.0.1:27017/?compressors=disabled&gssapiServiceName=mongodb
Implicit session: session { "id" : UUID("23d61600-f07b-4599-9376-7472fe89404a") }
MongoDB server version: 4.2.15
Welcome to the MongoDB shell.
For interactive help, type "help".
For more comprehensive documentation, see
	https://docs.mongodb.com/
Questions? Try the MongoDB Developer Community Forums
	https://community.mongodb.com
Server has startup warnings: 
2021-07-26T13:06:46.156+0000 I  CONTROL  [main] 
2021-07-26T13:06:46.156+0000 I  CONTROL  [main] ** WARNING: Access control is not enabled for the database.
2021-07-26T13:06:46.156+0000 I  CONTROL  [main] **          Read and write access to data and configuration is unrestricted.
2021-07-26T13:06:46.156+0000 I  CONTROL  [main] ** WARNING: You are running this process as the root user, which is not recommended.
2021-07-26T13:06:46.156+0000 I  CONTROL  [main] 
mongos> use test
switched to db test
mongos> db.createUser({user: "test", pwd: "testpwd", roles: ["readWrite"]})
Successfully added user: { "user" : "test", "roles" : [ "readWrite" ] }
```  

### GCP 방화벽 오픈
GCP의 경우 27017 포트는 기본적으로 오픈되어 있지 않기 때문에 외부에서 접속이 가능하도록 설정을 해야한다.  
<img src="https://sisipapa.github.io/assets/images/posts/gcp-mongo-firewall.PNG" >  

### Mongo Client 접속 툴 설치 후 접속확인
Mongo Client 툴을 꼭 설치할 필요는 없지만 Robo 3T를 다운로드해서 접속을 확인한다.  
[Robo 3T 홈페이지 Download](https://robomongo.org/)  
- Robo 3T 접속 Connection  
<img src="https://sisipapa.github.io/assets/images/posts/robo3t-0.PNG" >  
- Robo 3T 접속 Authentication    
<img src="https://sisipapa.github.io/assets/images/posts/robo3t-1.PNG" >  
- 접속완료  
<img src="https://sisipapa.github.io/assets/images/posts/robo3t-2.PNG" >  
  
이제 Application에서 접속할 수 있는 환경이 구축이 되었다.  

## Springboot MongoDB 연동    
Springboot와 MongoDB 연동테스트는 간단하게 MongoDB 연결 설정 및 최소한의 Config파일만 작성하고 Unit Test로 결과만 확인할 예정이다.  

### bulid.gradle
```properties
...
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-data-mongodb'
    compileOnly 'org.projectlombok:lombok'
    annotationProcessor 'org.projectlombok:lombok'
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
}
...
```  

### application.yaml
```yaml
spring:
  data:
    mongodb:
      host: 서버정보입력
      port: 27017
      database: test
      username: test
      password: testpwd
```  

### MongoDBConfig  
데이터 저장시 _class 필드 추가되는 현상 방지  
```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.mongodb.MongoDatabaseFactory;
import org.springframework.data.mongodb.core.convert.DbRefResolver;
import org.springframework.data.mongodb.core.convert.DefaultDbRefResolver;
import org.springframework.data.mongodb.core.convert.DefaultMongoTypeMapper;
import org.springframework.data.mongodb.core.convert.MappingMongoConverter;
import org.springframework.data.mongodb.core.mapping.MongoMappingContext;

@Configuration
public class MongoDBConfig {
    @Bean
    public MappingMongoConverter mappingMongoConverter(
            MongoDatabaseFactory mongoDatabaseFactory,
            MongoMappingContext mongoMappingContext
    ) {
        DbRefResolver dbRefResolver = new DefaultDbRefResolver(mongoDatabaseFactory);
        MappingMongoConverter converter = new MappingMongoConverter(dbRefResolver, mongoMappingContext);
        converter.setTypeMapper(new DefaultMongoTypeMapper(null));
        return converter;
    }
}
```  

### MongoDB Document 객체  
```java
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;
import org.springframework.data.annotation.Id;
import org.springframework.data.mongodb.core.mapping.Document;

@Document(collection = "users")
@NoArgsConstructor
@Data
public class User {
    @Id
    private String id;
    private String name;
    private String email;
    private String password;

    @Builder
    public User(String id, String name, String email, String password){
        this.id = id;
        this.name = name;
        this.email = email;
        this.password = password;
    }
}
```  

### Repository  
```java
import lombok.AllArgsConstructor;
import org.springframework.data.mongodb.core.MongoTemplate;
import org.springframework.data.mongodb.core.query.Criteria;
import org.springframework.data.mongodb.core.query.Query;
import org.springframework.data.mongodb.core.query.Update;
import org.springframework.stereotype.Repository;

@AllArgsConstructor
@Repository
public class MongoRepository {

    private MongoTemplate mongoTemplate;

    /**
     * save 등록
     * @param user
     */
    public void save(User user){
        mongoTemplate.save(user);
    }

    /**
     * insert 등록
     * @param user
     */
    public void insert(User user){
        mongoTemplate.insert(user);
    }

    /**
     * 수정과 동시에 조회
     * @param whereField
     * @param whereValue
     * @param updateField
     * @param updateValue
     */
    public void findAndModify(String whereField,
                              String whereValue,
                              String updateField,
                              String updateValue){
        mongoTemplate.findAndModify(
                Query.query(Criteria.where(whereField).is(whereValue)),
                Update.update(updateField, updateValue),
                User.class
        );
    }

    /**
     * 단일 데이터 수정
     * @param whereField
     * @param whereValue
     * @param updateField
     * @param updateValue
     */
    public void updateFirst(String whereField,
                            String whereValue,
                            String updateField,
                            String updateValue){
        mongoTemplate.updateFirst(
                Query.query(Criteria.where(whereField).is(whereValue)),
                Update.update(updateField, updateValue),
                User.class
        );
    }

    /**
     * 복수 데이터 수정
     * @param whereField
     * @param whereValue
     * @param updateField
     * @param updateValue
     */
    public void updateMulti(String whereField,
                            String whereValue,
                            String updateField,
                            String updateValue){
        mongoTemplate.updateMulti(
                Query.query(Criteria.where(whereField).is(whereValue)),
                Update.update(updateField, updateValue),
                User.class
        );
    }
}
```  

### RepositoryTest  
```java
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;

@SpringBootTest
class MongoRepositoryTest {

    @Autowired
    MongoRepository mongoRepository;

    @Test
    void save() {
        User user = User.builder().id(null).name("aaa").email("aaa@a.com").password("aaapwd").build();
        mongoRepository.save(user);
    }

    @Test
    void insert() {
        User user = User.builder().id(null).name("bbb").email("bbb@b.com").password("bbbpwd").build();
        mongoRepository.insert(user);
    }
    @Test
    void findAndModify() {
        User user = User.builder().id(null).name("bbb").email("bbb@b.com").password("bbbpwd").build();
        mongoRepository.findAndModify("name", "aaa", "email", "aaa-0@b.com");
    }

    @Test
    void updateFirst() {
        User user = User.builder().id(null).name("bbb").email("bbb@b.com").password("bbbpwd").build();
        mongoRepository.updateFirst("name", "bbb", "email", "bbb-0@b.com");
    }

    @Test
    void updateMulti() {
        User user = User.builder().id(null).name("bbb").email("bbb@b.com").password("bbbpwd").build();
        mongoRepository.updateMulti("name", "aaa", "email", "aaa-1@b.com");
    }
}
```  
오늘은 등록,수정 테스트만 진행을 했다. 추가로 조회 삭제테스트를 진행하고 추가할 예정이다.

## 참고  
[Be OK - MongoDB Spring 연동](https://sg-choi.tistory.com/388)  

## Github  
<https://github.com/sisipapa/Springboot-mongoDB.git> 