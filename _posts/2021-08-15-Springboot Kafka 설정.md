---
layout: post 
title: Springboot Kafka 설정
category: [kafka]
tags: [kafka, springboot]
redirect_from:

- /2021/08/15/

---

이전 정리에서 Kafka Cluster 구성 정리가 완료되었다. 오늘부터는 Kafka Cluster와 Springboot와의 연동을 정리해 보려고 한다. Kafka Application의 Producer와 Consumer는 보통은 다른 서버로 구성되지만 여기서는 하나의 Application으로 진행할 예정이다.

## Procucer, Consumer 공통
### build.gralde  
spring-kafka를 추가해준다.
```properties
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter'
    implementation 'org.springframework.kafka:spring-kafka'
    implementation 'org.springframework.boot:spring-boot-starter-web'
    compileOnly 'org.projectlombok:lombok'
    annotationProcessor 'org.projectlombok:lombok'
    testImplementation('org.springframework.boot:spring-boot-starter-test') {
        exclude group: 'org.junit.vintage', module: 'junit-vintage-engine'
    }
    testImplementation 'org.springframework.kafka:spring-kafka-test'

}
```  

### PushDto.java
Producer와 Consumer가 주고받을 메세지Dto
```java
@Data
@NoArgsConstructor
@AllArgsConstructor
public class PushDto {

    private String token;
    private String message;

}
```  

### application.properties
Producer 서버의 Kafka서버주소, Topic, Consumer 서버의 Kafka서버주소, Topic, Consumer만 설정한다.

```properties
kafka.producer.bootstrap.servers=카프카1서버IP:9092,카프카2서버IP:9093카프카3서버IP:9094
kafka.producer.topic.name=test1-push


kafka.consumer.bootstrap.servers=카프카1서버IP:9092,카프카2서버IP:9093,카프카3서버IP:9094
kafka.consumer.topic.name=test1-push
kafka.consumer.topic.group.name=test1-push-group
```  

## Producer 구성  

### KafkaAdminConfig.java  
KafkaAdmin 객체 초기화하고 어플리케이션 로딩 시 topicName에 해당하는 Topic을 등록한다. topicName이름의 Topic이 등록되어 있으면 아무런 동작을 하지 않는다.  
```java
@Configuration
public class KafkaAdminConfig {

    @Value("${kafka.producer.bootstrap.servers}")
    private String bootstrapServers;

    @Value("${kafka.producer.topic.name}")
    private String topicName;

    @Bean
    public KafkaAdmin kafkaAdmin() {
        Map<String, Object> configs = new HashMap<>();
        configs.put(AdminClientConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
        return new KafkaAdmin(configs);
    }

    @Bean
    public NewTopic newTopic() {
        return new NewTopic(topicName, 3, (short) 3); // topicName, 파티션수, replication 수
    }
}
```  

KafkaAdminConfig의 newTopic은 아래 kafka-topics.sh
```shell
$ ./kafka-topics.sh --create --zookeeper localhost:2181,localhost:2182,localhost:2183 --replication-factor 3 --partitions 3 --topic test1-push
```

### KafkaProducerConfig.java  
KafkaTemplate 객체 설정해준다. Consumer에게 전달 메세지 타입 설정을 한다.
```java
@Configuration
public class KafkaProducerConfig {

    @Value("${kafka.producer.bootstrap.servers}")
    private String bootstrapServers;

    @Bean
    public ProducerFactory<String, PushDto> producerFactory() {
        Map<String, Object> configProps = producerFactoryConfig();
        return new DefaultKafkaProducerFactory<>(configProps);
    }

    private Map<String, Object> producerFactoryConfig() {
        Map<String, Object> configProps = new HashMap<>();
        configProps.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
        configProps.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        configProps.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, JsonSerializer.class);
        return configProps;
    }

    @Bean
    public KafkaTemplate<String, PushDto> kafkaTemplate() {
        return new KafkaTemplate<>(producerFactory());
    }

}
```  

### KafkaMessageSender.java  
kafkaTemplate.send(message); 는 리턴타입은 ListenableFuture 이다. ListenableFuture을 통해 Producer의 메세지 송신여부, 현재 보낸 파티션의 offset값을 확인할 수 있다.
```java
@Slf4j
@Component
public class KafkaMessageSender {

    @Autowired
    private KafkaTemplate<String, PushDto> kafkaTemplate;

    @Value("${kafka.producer.topic.name}")
    private String topicName;

    public void send(PushDto pushDto) {

        Message<PushDto> message = MessageBuilder
                .withPayload(pushDto)
                .setHeader(KafkaHeaders.TOPIC, topicName)
                .build();

        ListenableFuture<SendResult<String, PushDto>> future =
                kafkaTemplate.send(message);

        future.addCallback(new ListenableFutureCallback<SendResult<String, PushDto>>() {

            @Override
            public void onSuccess(SendResult<String, PushDto> stringObjectSendResult) {
                log.info("Sent message=[" + stringObjectSendResult.getProducerRecord().value().toString() + "] with offset=[" + stringObjectSendResult.getRecordMetadata().offset() + "]");
            }

            @Override
            public void onFailure(Throwable e) {
                log.info("KafkaMessageSender onFailure : {}" + e.getMessage());
            }
        });
    }

}
```  

### KafkaPushController.java  
클라이언트의 post요청을 받아서 KafkaMessageSender.send한다.
```java
@RestController
public class KafkaPushController {

    @Autowired
    private KafkaMessageSender kafkaMessageSender;

    @PostMapping("/push")
    public String push(@RequestBody PushDto pushDto) {
        kafkaMessageSender.send(pushDto);
        return "success";
    }

}
```  

## Consumer  
### KafkaConsumerConfig.java  
consumerFactoryConfig() 메소드에서 Deserializer들은 Producer 구성과 동일하게 한다. @EnableKafka 어노테이션을 붙여준다.  
```java
@EnableKafka
@Configuration
public class KafkaConsumerConfig {

    @Value("${kafka.consumer.bootstrap.servers}")
    private String bootstrapServers;

    @Value("${kafka.consumer.topic.group.name}")
    private String groupId;

    @Bean
    public ConsumerFactory<String, PushDto> pushEntityConsumerFactory() {
        JsonDeserializer<PushDto> deserializer = pushEntityJsonDeserializer();
        return new DefaultKafkaConsumerFactory<>(
                consumerFactoryConfig(deserializer),
                new StringDeserializer(),
                deserializer);
    }

    private Map<String, Object> consumerFactoryConfig(JsonDeserializer<PushDto> deserializer) {
        Map<String, Object> props = new HashMap<>();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
        props.put(ConsumerConfig.GROUP_ID_CONFIG, groupId);
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, deserializer);
        return props;
    }

    private JsonDeserializer<PushDto> pushEntityJsonDeserializer() {
        JsonDeserializer<PushDto> deserializer = new JsonDeserializer<>(PushDto.class);
        deserializer.setRemoveTypeHeaders(false);
        deserializer.addTrustedPackages("*");
        deserializer.setUseTypeMapperForKey(true);
        return deserializer;
    }

    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, PushDto>
    pushEntityKafkaListenerContainerFactory() {
        ConcurrentKafkaListenerContainerFactory<String, PushDto> factory =
                new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(pushEntityConsumerFactory());
        return factory;
    }

}
```  

### KafkaMessageListener.java  
@KafkaListener 어노테이션을 추가하고 topics, groupId를 설정해서 원하는 메세지를 Concumer한다. 
```java
@Slf4j
@Service
public class KafkaMessageListener {

    @KafkaListener(topics = "${kafka.consumer.topic.name}"
            , groupId = "${kafka.consumer.topic.group.name}"
            , containerFactory = "pushEntityKafkaListenerContainerFactory")
    public void listenWithHeaders(@Payload PushDto pushDto,
                                  @Headers MessageHeaders messageHeaders) {

        log.info("Received Message: {}, headers: {}", pushDto.toString() ,messageHeaders);
    }
}
```  

## 테스트

### POSTMAN 요청
http://localhost:8080/push URI에 POST 요청을 한다.(token0001~token0004)  
<img src="https://sisipapa.github.io/assets/images/posts/kafka-postman.png" >  

### kafka 로그
```shell
$ ./kafka-console-consumer.sh --bootstrap-server localhost:9092,localhost:9093,localhost:9094 --topic test1-push --from-beginning
{"token":"token0001","message":"message0001"}
{"token":"token0002","message":"message0002"}
{"token":"token0003","message":"message0003"}
{"token":"token0004","message":"message0004"}
```  

### Application 로그
```shell
Sent message=[PushDto(token=token0001, message=message0001)] with offset=[0]
Received Message: PushDto(token=token0001, message=message0001), headers: {kafka_offset=0, kafka_consumer=org.apache.kafka.clients.consumer.KafkaConsumer@304b24a6, kafka_timestampType=CREATE_TIME, kafka_receivedPartitionId=1, kafka_receivedTopic=test1-push, kafka_receivedTimestamp=1629017795471, __TypeId__=[B@6f89e668, kafka_groupId=test1-push-group}
Sent message=[PushDto(token=token0002, message=message0002)] with offset=[1]
Received Message: PushDto(token=token0002, message=message0002), headers: {kafka_offset=1, kafka_consumer=org.apache.kafka.clients.consumer.KafkaConsumer@304b24a6, kafka_timestampType=CREATE_TIME, kafka_receivedPartitionId=2, kafka_receivedTopic=test1-push, kafka_receivedTimestamp=1629017805100, __TypeId__=[B@425da97b, kafka_groupId=test1-push-group}
Sent message=[PushDto(token=token0003, message=message0003)] with offset=[1]
Received Message: PushDto(token=token0003, message=message0003), headers: {kafka_offset=1, kafka_consumer=org.apache.kafka.clients.consumer.KafkaConsumer@304b24a6, kafka_timestampType=CREATE_TIME, kafka_receivedPartitionId=1, kafka_receivedTopic=test1-push, kafka_receivedTimestamp=1629017810611, __TypeId__=[B@5e6e6f3b, kafka_groupId=test1-push-group}
Sent message=[PushDto(token=token0004, message=message0004)] with offset=[9]
Received Message: PushDto(token=token0004, message=message0004), headers: {kafka_offset=9, kafka_consumer=org.apache.kafka.clients.consumer.KafkaConsumer@304b24a6, kafka_timestampType=CREATE_TIME, kafka_receivedPartitionId=0, kafka_receivedTopic=test1-push, kafka_receivedTimestamp=1629017815910, __TypeId__=[B@a1e0d70, kafka_groupId=test1-push-group}
```  

Kafka Cluster 구성이후 Springboot와의 연동테스트를 진행해 보았다. 다음에 시간이 된다면 Kafka Cluster의 설정값들에 대한 정리를 추가로 해봐야 겠다.  

## 참고
[[SpringBoot] Kafka 연동하기](https://victorydntmd.tistory.com/348)
[Kafka Cluster 구성하고 Spring Boot에서 Kafka 사용하기](https://sup2is.github.io/2020/06/03/spring-boot-with-kafka-cluster.html)

## Github
<https://github.com/sisipapa/Springboot-KafkaCluster.git>
