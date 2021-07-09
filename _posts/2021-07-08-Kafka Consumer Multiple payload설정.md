---
layout: post 
title: Kafka Consumer Multiple Payload 설정 
category: [kafka]
tags: [kafka, KafkaConsumer]
redirect_from:

- /2021/07/08/

---

## 장애내용 및 원인파악
현재 운영중인 플랫폼의 특정 서비스에서 밤 10시~12시 트래픽이 많은 시간이 되면 pod의 CPU가 차고 결국에는 해당 pod가 제기능을 못하게 되고 연쇄적으로 나머지 pod까지 리소스가 부족하게 되어 서비스가 안되는 상태가 되었다. 해당 서비스는 6개의 Kafka Topic을 처리하는 컨슈머가 있었고 컨슈머는 List형태의 payload를 파라미터로 전달받아 쓰레드로 비지니스를 처리하도록 되어있다.
Thread Dump를 확인해서 CPU가 많이 발생할 수 있는 부분은 아래 두가지 이유라고 판단하고 조치를 시작하게 되었다.  
1. Topic별로 Message처리를 위해 고정 30개의 Thread를 만들어서 처리를 하는데 개선이 필요.    
2. MongoDB 저장,조회시 ReadTimeException, SocketTimeException - MongoDB Exception의 경우는 Timeout 설정을 통해 해결을 했고 여기서는 자세히 다루지는 않겠다.  

## AS-IS KafkaConsumer 설정  
Kafka 설정은 기존에 되어있었는데 Topic당 유입되는 Message의 수량을 파악해야 Tharea의 적정 수량을 설정할 수 있겠다 판단되어 List로 넘어오는 payload의 사이즈를 확인했는데 항상 1이 넘어오고 있었다.  
- Consumer Configuration  
```java  
  ... 생략
  
  @Bean(name = "jsonKafkaListenerContainerFactory")
  public ConcurrentKafkaListenerContainerFactory<String, String> jsonKafkaListenerContainerFactory() {
    ConcurrentKafkaListenerContainerFactory<String, String> factory = new ConcurrentKafkaListenerContainerFactory<>();
//    factory.setErrorHandler(errorHandler());
//    factory.setBatchErrorHandler(new BatchLoggingErrorHandler()); //batchErrorHandler()
    factory.setBatchErrorHandler(batchErrorHandler()); //batchErrorHandler()
    factory.setConsumerFactory(jsonConsumerFactory());
    factory.setConcurrency(5);
    return factory;
  }
  
  ... 생략
```

- Properties  
```properties
spring.kafka.consumer.bootstrap-servers=11.111.111.11:1111,11.111.111.12:1111,11.111.111.13:1111,11.111.111.14:1111
spring.kafka.consumer.enable-auto-commit=true
spring.kafka.consumer.auto-commit-interval=1000
spring.kafka.consumer.session-timeout=10000
spring.kafka.consumer.heartbeat-interval=3000
spring.kafka.consumer.max-poll-interval=9000
spring.kafka.consumer.max-poll-records=1000
spring.kafka.consumer.fetch-min-size=0
```  

## TO-BE KafkaConsumer 설정  
- 카프카 컨슈머 설정 중 Consumer의 양과 주기는 시스템에 맞게 아래 설정을 조절해 주면 된다.  
1. fetch.min.bytes : 한번에 가져올 수 있는 최소 사이즈로, 만약 가져오는 데이터가 지정한 사이즈보다 작으면 요청에 응답하지 않고, 데이터가 누적될 때 까지 기다린다.  
2. fetch.max.wait.ms(df: 500sec = 0.5sec) : fetch.min.bytes에 설정된 데이터보다 데이터 양이 적은 경우 요청에 응답을 기다리는 최대시간을 설정  
3. max.poll.records(df: 500) : 폴링루프에서 이뤄지는 한건의 KafkaConsumer.poll() 메소드에 대한 최대 레코드수를 조정한다.

- Consumer Configuration
```java  
  ... 생략
  
  @Bean(name = "jsonKafkaListenerContainerFactory")
  public ConcurrentKafkaListenerContainerFactory<String, String> jsonKafkaListenerContainerFactory() {
    ConcurrentKafkaListenerContainerFactory<String, String> factory = new ConcurrentKafkaListenerContainerFactory<>();
//    factory.setErrorHandler(errorHandler());
//    factory.setBatchErrorHandler(new BatchLoggingErrorHandler()); //batchErrorHandler()
    factory.setBatchErrorHandler(batchErrorHandler()); //batchErrorHandler()
    factory.setConsumerFactory(jsonConsumerFactory());
    factory.setConcurrency(5);
    factory.setBatchListener(true);
    return factory;
  }
  
  ... 생략
```

- Properties - 처음 spring.kafka.consumer.fetch-max-wait를 1초를 설정하고 테스트를 진행했는데 렉이 쌓여서 0으로 변경. 
```properties
spring.kafka.consumer.bootstrap-servers=11.111.111.11:1111,11.111.111.12:1111,11.111.111.13:1111,11.111.111.14:1111
spring.kafka.consumer.enable-auto-commit=true
spring.kafka.consumer.auto-commit-interval=1000
spring.kafka.consumer.session-timeout=10000
spring.kafka.consumer.heartbeat-interval=3000
spring.kafka.consumer.max-poll-interval=9000
spring.kafka.consumer.max-poll-records=1000
spring.kafka.consumer.fetch-min-size=10000000
spring.kafka.consumer.fetch-max-wait=0
```  
```java

```

##  성능향상을 Thread 로직 개선  
컨슈머에서 각 토픽별로 Thread를 생성해서 사용했던 부분을 parallelStream을 통한 병렬처리로 변경했더니 CPU사용량이 크게 감소했다. parallelStream을 그냥 사용하게 되면 CPU갯수만큼 Thread가 생성되어 처리를 하게 되는데 운영중인 서비스의 pod당 CPU가 4~5G 였기때문에 Topic을 처리하는 Thread가 부족하다고 판단되어 forkJoinPool을 토픽별로 설정해서 Thread수를 조정했다.
```java  
... 생략
private static ForkJoinPool forkJoinPool = new ForkJoinPool(10);
...생략

forkJoinPool.submit(() -> {
    payload.parallelStream().forEach(jsonObject -> {
      try {
        TimeUnit.MILLISECONDS.sleep(10);
      } catch (InterruptedException ignore) {
      }
      // 비지니스 로직!
    });
});
```  
앞으로 기존 사용하던 로직의 Resource가 많이 소모되던 부분이 어디였는지 parallelStream을 사용해서 왜 성능이나 리소스 사용량이 개선되었는지에 대한 확인이 필요해 보인다....  

## 참고  
[YABOONG](https://yaboong.github.io/spring/2020/06/07/kafka-batch-consumer-unintended-listener-invoking/)  
[Jinhyy-5장 카프카 컨슈머](https://jinhyy.tistory.com/63)

