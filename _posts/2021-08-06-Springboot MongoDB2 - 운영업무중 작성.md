---
layout: post 
title: Springboot MongoDB2 - 운영업무
category: [mongo]
tags: [mongo, springboot]
redirect_from:

- /2021/08/06/

---

현재 운영중인 시스템의 몽고에 적재된 데이터 중에 2017년에 적재된 데이터 중 날짜필드의 타입이 yyyyMMdd로 저장된 데이터로 인해 서버에서 날짜 parsing 에러가 발생했다. 현재는 해당 필드의 날짜 타입은 yyyyMMddHHmmss로 사용하고 있다. yyyyMMdd 날짜 타입으로 저장된 데이터는 활용할 수 없는 데이터로 판단하고 해당 날짜타입의 데이터를 삭제해 달라는 업무를 전달 받았다. MongoDB 데이터의 구조는 아래와 같다.  
```json
"AAA" : {
    "syncTime" : ISODate("2021-08-05T09:57:40.871Z"),
    "data" : [ 
        {
            "aaa" : "",
            "bbb" : "",
            "ccc" : "",
            "ddd" : "",
            "eee" : "",
            "fff" : "yyyyMMddHHmmss",
        },
        {
        "aaa" : "",
        "bbb" : "",
        "ccc" : "",
        "ddd" : "",
        "eee" : "",
        "fff" : "yyyyMMdd",
        }
...
    ]
}
```  
운영중인 시스템의 몽고는 20개의 Shard와 각 Shard별로 ReplicaSet으로 구성되어 있다 서버 대수로는 총 40대이다. 보관되어 있는 데이터의 양도 Shard 서버당 3억개가 보관되어 있고 20개의 Shard에는 60억개의 데이터가 존재한다. 처음에는 각 Shard 서버별로 삭제 쿼리를 날려서 삭제하면 되겠다라고 생각했는데 키값에 해당하는 Document전체를 삭제하는 것이 아닌 Document의 데이터 중 AAA라는 필드중 List형태의 data필드 중에 fff필드가 yyyyMMdd형태인 데이터만 삭제를 해야 하기때문에 단순 쿼리로 진행이 어려울 것이라고 판단했다. 배치성으로 계속해서 반복되는 업무가 아닌 일회성으로 한번 사용되고 말거라고 판단되어 개인 프로젝트를 새로 만들어 진행해 보려고 한다.

## 개발진행
웹어플리케이션이 아닌 비지니스 로직을 처리할 Service 레이어까지만 만들고 Test클래스 내에서 Service를 호출해서 업무를 처리 할 예정이다.

### build.gradle
spring-boot-starter-data-mongodb dependency만 추가하고 Test클래스에서 activeProfile변수를 사용하기 위한 bootRun 설정만 추가했다.

```properties
...

dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-data-mongodb'
    compileOnly 'org.projectlombok:lombok'
    annotationProcessor 'org.projectlombok:lombok'
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
}
... 

bootRun {
    String activeProfile = System.properties['spring.profiles.active']
    systemProperty "spring.profiles.active", activeProfile
}
```  

### yaml
yaml 파일에는 몽고DB 접속정보만 있다. dev mongos 인증 정보를 넣고 접속을 하게 구성이 되어있고 prod mongos는 인증정보없이 접속이 가능하다. 접속정보 및 DB정보는 보안상 삭제처리 
```yaml
# application-dev.yaml
spring:
  data:
    mongodb:
      AAA:
        host: 
        port: 
        database: AAA
        id: 
        password: 
      BBB:
        host: 
        port: 
        database: BBB
        id: 
        password: 

# application-prod.yaml
spring:
  data:
    mongodb:
      AAA:
        host: 
        port: 
        database: AAA
      BBB:
        host: 
        port: 
        database: BBB
```  

### Config  
Config는 두 곳의 Collection에 접속이 가능하도록 Multi Connection을 위한 설정을 했다. AAA,BBB 접속이 가능하도록 두 개의 Config 파일로 구성
```java
// AAAConfig
@Configuration
@Slf4j
public class MongoAAAConfig {

  @Autowired
  Environment environment;

  public MongoClient getMongoClient() {

    String host = environment.getProperty("spring.data.mongodb.AAA.host");
    int port = Integer.parseInt(environment.getProperty("spring.data.mongodb.AAA.port"));
    String activeProfile = environment.getActiveProfiles()[0];
    String id = "";
    String password = "";
    String database = "";
    MongoCredential credential = null;

    if("dev".equals(activeProfile)){
      id = environment.getProperty("spring.data.mongodb.AAA.id");
      password = environment.getProperty("spring.data.mongodb.AAA.password");
      database = environment.getProperty("spring.data.mongodb.AAA.database");
      credential = MongoCredential.createCredential(id, database, password.toCharArray());
    }

    List<ServerAddress> serverAddressList = List.of(new ServerAddress(host, port));

    SocketSettings socketSettings = SocketSettings.builder()
            .connectTimeout(30000, TimeUnit.MILLISECONDS)
            .readTimeout(1200000, TimeUnit.MILLISECONDS)        
            .build();
    ClusterSettings clusterSettings = ClusterSettings.builder()
            .hosts(serverAddressList)
            .serverSelectionTimeout(200, TimeUnit.MILLISECONDS)
            .build();
    ConnectionPoolSettings connectionPoolSettings = ConnectionPoolSettings.builder()
            .maxSize(100)
            .minSize(50)
            .maxWaitTime(3000, TimeUnit.MILLISECONDS)
            .maxConnectionIdleTime(300000, TimeUnit.MILLISECONDS)
            .maxConnectionLifeTime(86400000, TimeUnit.MILLISECONDS)
            .build();
    ServerSettings serverSettings = ServerSettings.builder()
            .heartbeatFrequency(200, TimeUnit.MILLISECONDS)
            .minHeartbeatFrequency(100, TimeUnit.MILLISECONDS)
            .build();

    MongoClientSettings clientSettings = null;
    if("dev".equals(activeProfile)){
      clientSettings = MongoClientSettings.builder()
              .applyToServerSettings(builder -> builder.applySettings(serverSettings))
              .applyToClusterSettings(builder -> builder.applySettings(clusterSettings))
              .applyToSocketSettings(builder -> builder.applySettings(socketSettings))
              .applyToConnectionPoolSettings(builder -> builder.applySettings(connectionPoolSettings))
              .readPreference(ReadPreference.secondary())
              .credential(credential)
              .retryWrites(true)
              .retryReads(true)
              .build();
    }else{
      clientSettings = MongoClientSettings.builder()
              .applyToServerSettings(builder -> builder.applySettings(serverSettings))
              .applyToClusterSettings(builder -> builder.applySettings(clusterSettings))
              .applyToSocketSettings(builder -> builder.applySettings(socketSettings))
              .applyToConnectionPoolSettings(builder -> builder.applySettings(connectionPoolSettings))
              .readPreference(ReadPreference.secondary())
              .retryWrites(true)
              .retryReads(true)
              .build();
    }

    return create(clientSettings);
  }

  @Primary
  @Bean(name="AAAMongoTemplate")
  public MongoTemplate AAAMongoTemplate() {
    MongoDatabaseFactory factory = new SimpleMongoClientDatabaseFactory(getMongoClient(), "AAA");
    MappingMongoConverter converter = new MappingMongoConverter(new DefaultDbRefResolver(factory), new MongoMappingContext());
    converter.setTypeMapper(new DefaultMongoTypeMapper(null));
    return new MongoTemplate(factory, converter);
  }
}



//BBBConfig
@Configuration
@Slf4j
public class MongoBBBConfig {
    @Autowired
    Environment environment;

    public MongoClient getMongoClient() {

        String host = environment.getProperty("spring.data.mongodb.BBB.host");
        int port = Integer.parseInt(environment.getProperty("spring.data.mongodb.BBB.port"));
        String activeProfile = environment.getActiveProfiles()[0];
        String id = "";
        String password = "";
        String database = "";
        MongoCredential credential = null;

        if("dev".equals(activeProfile)){
            id = environment.getProperty("spring.data.mongodb.BBB.id");
            password = environment.getProperty("spring.data.mongodb.BBB.password");
            database = environment.getProperty("spring.data.mongodb.BBB.database");
            credential = MongoCredential.createCredential(id, database, password.toCharArray());
        }

        List<ServerAddress> serverAddressList = List.of(new ServerAddress(host, port));

        SocketSettings socketSettings = SocketSettings.builder()
                .connectTimeout(30000, TimeUnit.MILLISECONDS)
                .readTimeout(1200000, TimeUnit.MILLISECONDS)        
                .build();
        ClusterSettings clusterSettings = ClusterSettings.builder()
                .hosts(serverAddressList)
                .serverSelectionTimeout(200, TimeUnit.MILLISECONDS)
                .build();
        ConnectionPoolSettings connectionPoolSettings = ConnectionPoolSettings.builder()
                .maxSize(100)
                .minSize(50)
                .maxWaitTime(3000, TimeUnit.MILLISECONDS)
                .maxConnectionIdleTime(300000, TimeUnit.MILLISECONDS)
                .maxConnectionLifeTime(86400000, TimeUnit.MILLISECONDS)
                .build();
        ServerSettings serverSettings = ServerSettings.builder()
                .heartbeatFrequency(200, TimeUnit.MILLISECONDS)
                .minHeartbeatFrequency(100, TimeUnit.MILLISECONDS)
                .build();

        MongoClientSettings clientSettings = null;
        if("dev".equals(activeProfile)){
            clientSettings = MongoClientSettings.builder()
                    .applyToServerSettings(builder -> builder.applySettings(serverSettings))
                    .applyToClusterSettings(builder -> builder.applySettings(clusterSettings))
                    .applyToSocketSettings(builder -> builder.applySettings(socketSettings))
                    .applyToConnectionPoolSettings(builder -> builder.applySettings(connectionPoolSettings))
                    .readPreference(ReadPreference.secondary())
                    .credential(credential)
                    .retryWrites(true)
                    .retryReads(true)
                    .build();
        }else{
            clientSettings = MongoClientSettings.builder()
                    .applyToServerSettings(builder -> builder.applySettings(serverSettings))
                    .applyToClusterSettings(builder -> builder.applySettings(clusterSettings))
                    .applyToSocketSettings(builder -> builder.applySettings(socketSettings))
                    .applyToConnectionPoolSettings(builder -> builder.applySettings(connectionPoolSettings))
                    .readPreference(ReadPreference.secondary())
                    .retryWrites(true)
                    .retryReads(true)
                    .build();
        }

        return create(clientSettings);
    }

    @Bean(name="BBBMongoTemplate")
    public MongoTemplate BBBMongoTemplate() {
        MongoDatabaseFactory factory = new SimpleMongoClientDatabaseFactory(getMongoClient(), "BBB");
        MappingMongoConverter converter = new MappingMongoConverter(new DefaultDbRefResolver(factory), new MongoMappingContext());
        converter.setTypeMapper(new DefaultMongoTypeMapper(null));
        return new MongoTemplate(factory, converter);
    }
}
```  

### Repository  
```java
@Slf4j
@Repository
public class AAARepository {

    @Autowired
    @Qualifier("AAAMongoTemplate")
    private MongoTemplate AAAMongoTemplate;

    private static final int COLLECTION_COUNT = 20;
    private static final String PREFIX_COLLECTION_NAME = "PREFIX_";
    private static final String CREATED_DATE = "regist_datetime";
    private static final String LAST_MODIFIED_DATE = "update_datetime";

    /**
     * 단건 조회
     * @param key
     * @param value
     * @return
     */
    public Document findById(String key, String value){

        String collectionName = "COLLECTION";

        Document document = null;
        Query query = new Query();
        try {
            query.addCriteria(where(key).is(value));
            document = AAAMongoTemplate.findOne(query, Document.class, collectionName);
        } catch (MongoException e) {
            e.printStackTrace();
        }
        return document;
    }

    /**
     * Aggregation Pipeline 쿼리 수행
     * @param collName
     * @param pipelines
     * @return
     */
    public AggregateIterable<Document> findByQquery(String collName, List<Document> pipelines){

        AggregateIterable<Document> cursor = AAAMongoTemplate.getCollection(collName)
                .aggregate(pipelines, Document.class)
                .allowDiskUse(true);

        return cursor;
    }

    /**
     * id에 해당하는 document upsert
     * @param id
     * @param document
     * @return
     */
    public UpdateResult writeByObjectOfUserLog(String id, Document document) {
        String collectionName = getCollectionName(id);

        Query query = new Query();
        Update update;
        UpdateResult result = null;
        try {
            query.addCriteria(where("_id").is(id));
            document.append(LAST_MODIFIED_DATE, new Date());
            update = Update.fromDocument(document);
            update.setOnInsert("_id", id);
            update.setOnInsert(CREATED_DATE, new Date());

            result = AAAMongoTemplate.upsert(query, update, Objects.requireNonNull(collectionName));

            if (log.isDebugEnabled()) {
                log.debug(String.format("CollectionName : %s , UpdateOne Status : %s", collectionName, result.wasAcknowledged()));
                log.debug(String.format("No of Record Modified : %d", result.getModifiedCount()));
            }
        } catch (MongoException e) {
            e.printStackTrace();
        }
        return result;
    }
}
```  

### Service  
Aggregation 쿼리를 수행해서 나온 결과 ID값을 반복문을 수행해서 비지니스 처리 - Java stream에 익숙해져 보기 위해 굳이 Stream으로 구현..
```java
@Slf4j
@AllArgsConstructor
@Service
public class AAAService {

    private AAARepository repository;

    public int upsert(){
        AtomicInteger totalCount = new AtomicInteger();

        Document match1 = new Document("$match", Document.parse("{'AAA' : {'$exists' : true}}"));
        Document project1 = new Document("$project", Document.parse("{'AAA.data' : true}"));
        Document unwind1 = new Document("$unwind", "$AAA.data");
        Document project2 = new Document("$project", Document.parse("{_id : true, 'tdLength' : {$strLenCP : '$AAA.data.fff'}}"));
        Document match2 = new Document("$match", Document.parse("{'fffLength' : {$ne : 14}}"));
        Document group1 = new Document("$group", Document.parse("{'_id' : '$_id'}"));

        List<Document> pipelines = List.of(match1, project1, unwind1, project2, match2, group1);

        IntStream intStream = IntStream.rangeClosed(0, 19);
        String collNamePrefix = "COLLECTION_";
        intStream.forEach(index -> {
            String collName = "";
            if(index < 10){
                collName = collNamePrefix + "0" + index;
            }else{
                collName = collNamePrefix + index;
            }
            log.info(collName + " start!");

            AggregateIterable<Document> cursor = repository.findByQquery(collName, pipelines);
            Iterator<Document> iter = cursor.iterator();
            Iterable<Document> iterable = () -> iter;
            List<Document> list = StreamSupport
                    .stream(iterable.spliterator(), false)
                    .collect(Collectors.toList());

            // 잘못된 날짜가 포함된 au_id 목록을 반복문 실행하면서 체크
            int listSize = list.size();
            totalCount.set(totalCount.get() + listSize);
            log.info("### " + collName + " size : {}", listSize);
            log.info("### total size : {}", totalCount.get());
            list.stream().forEach(document -> {
                // userlog의 경우 au_id에 따라서 collection 이름이 다르기 때문에 IN쿼리로 처리불가
                // au_id에 해당하는 Document 조회
                String id = document.getString("_id");
                Document fullDocument =  repository.findByAuId("_id", id);
                Document AAADocument = (Document)fullDocument.get("AAA");
                List<Document> AAAList = AAADocument.getList("data", Document.class);
                log.info("### before size : " + AAAList.size());
                var filterAAAList = AAAList.stream()
                        .filter(doc -> Objects.nonNull(doc))
                        .filter(doc -> Objects.nonNull(doc.getString("fff")))
                        .filter(doc -> doc.getString("fff").length() == 14)
                        .collect(Collectors.toList());
                log.info("### after size : " + filterAAAList.size());
                AAADocument.replace("syncTime", new Date());
                AAADocument.replace("data", filterAAAList);
                fullDocument.replace("AAA", AAADocument);
                repository.writeByObjectOfUserLog(auId, fullDocument);
            });

            log.info(collName + " end!");
        });

        return totalCount.get();
    }
}
```  

### Test
ActiveProfiles("dev") 로 설정 후 테스트 진행 후 정상을 확인. 다음 새벽작업이 있을 때 운영 데이터 삭제 예정
```java
@SpringBootTest
@ActiveProfiles("dev")
//@ActiveProfiles("prod")
class MobonMongoApplicationTests {

    @Autowired
    AAAService AAAService;

    @Test
    void contextLoads() {
    }

    @Test
    void upsert(){
        int totalCount = AAAService.iSrTdUpsert();
        System.out.println("### upsert : " + totalCount);
    }

}
```

MongoDB에 대한 운영경험이 많지가 않아서 단순 쿼리로 처리할 수 있는 부분을 굳이 Application으로 만들어서 처리를 하는 생각도 든다. 더 좋은 처리 방법을 찾게 되면 추가로 정리를 해야겠다.  
