---
layout: post
title: ELK Stack Docker-Compose 설치
category: [docker]
tags: [elk, elasticsearch, docker, docker-compose]
redirect_from:

- /2021/09/02/

---

얼마전 Springboot MSA 프로젝트를 구성하고 중앙 집중식 로깅을 위해 ELK Stack을 적용해 봐야겠다고 생각을 했다. 처음에는 개인 AWS 클라우드 서버에 구성을 하려고 헀으나 비용의 압박으로 로컬 PC의 Docker환경으로 구성을 해보려고 한다. 여기서는 X-Pack: Elastic Stack의 확장팩은 설치하지 않을 것이다.     

## ELK Docker Repository Clone  
```shell
$ git clone https://github.com/deviantony/docker-elk.git
$ cd docker-elk
```  

## 1. Elasticsearch 설정 변경  
### 1-1. elasticsearch.yml 수정  
X-Pack settings 관련 내용 주석.  
```shell
$ vi elasticsearch/config/elasticsearch.yml
```  
```yaml
---
## Default Elasticsearch configuration from Elasticsearch base image.
## https://github.com/elastic/elasticsearch/blob/master/distribution/docker/src/docker/config/elasticsearch.yml
#
cluster.name: "docker-cluster"
network.host: 0.0.0.0

## X-Pack settings
## see https://www.elastic.co/guide/en/elasticsearch/reference/current/setup-xpack.html
#
# xpack.license.self_generated.type: trial
# xpack.security.enabled: true
# xpack.monitoring.collection.enabled: true
```  

### 1-2. Dockerfile  
Dockerfile에 한글 분석기 nori 설치 관련 설정 추가.  
```shell
$ vi elasticsearch/Dockerfile
```  

```dockerfile
ARG ELK_VERSION

# https://www.docker.elastic.co/
FROM docker.elastic.co/elasticsearch/elasticsearch:${ELK_VERSION}

# Add your elasticsearch plugins setup here
# Example: RUN elasticsearch-plugin install analysis-icu
RUN elasticsearch-plugin install analysis-nori
```  

## 2. Kibana 설정 변경  
### 2-1. kibana.yml 수정
X-Pack security credentials 관련 내용 주석
```shell
vi kibana/config/kibana.yml
```  
```yaml
---
## Default Kibana configuration from Kibana base image.
## https://github.com/elastic/kibana/blob/master/src/dev/build/tasks/os_packages/docker_generator/templates/kibana_yml.template.ts
#
server.name: kibana
server.host: 0.0.0.0
elasticsearch.hosts: [ "http://elasticsearch:9200" ]
monitoring.ui.container.elasticsearch.enabled: true

## X-Pack security credentials
#
# elasticsearch.username: elastic
# elasticsearch.password: changeme
```  
## 3. Logstash 설정 변경
### 3-1. logstash.yml 수정  
X-Pack security credentials 관련 내용 주석
```shell
$ vi kibana/config/kibana.yml
```  
```yaml
---
## Default Logstash configuration from Logstash base image.
## https://github.com/elastic/logstash/blob/master/docker/data/logstash/config/logstash-full.yml
#
http.host: "0.0.0.0"
xpack.monitoring.elasticsearch.hosts: [ "http://elasticsearch:9200" ]

## X-Pack security credentials
#
# xpack.monitoring.enabled: true
# xpack.monitoring.elasticsearch.username: elastic
# xpack.monitoring.elasticsearch.password: changeme
```  

### 3-2. logstash.conf 수정  
output.elasticsearch.user 수정   
output.elasticsearch.password 수정  
output.elasticsearch.index 추가  
```properties
input {
	beats {
		port => 5044
	}

	tcp {
		port => 5000
	}
}

## Add your filters / logstash plugins configuration here

output {
	elasticsearch {
		hosts => "elasticsearch:9200"
		user => "sisipapa"
		password => "sisipapa!23$"
		ecs_compatibility => disabled
		index => "logstash-sisipapa"
	}
}
```  

## 4. docker-compose.yml 설정 변경  
services.elasticsearch.environment.ELASTIC_PASSWORD 수정  
```yaml
version: '3.2'

services:
  elasticsearch:
    build:
      context: elasticsearch/
      args:
        ELK_VERSION: $ELK_VERSION
    volumes:
      - type: bind
        source: ./elasticsearch/config/elasticsearch.yml
        target: /usr/share/elasticsearch/config/elasticsearch.yml
        read_only: true
      - type: volume
        source: elasticsearch
        target: /usr/share/elasticsearch/data
    ports:
      - "9200:9200"
      - "9300:9300"
    environment:
      ES_JAVA_OPTS: "-Xmx256m -Xms256m"
      ELASTIC_PASSWORD: sisipapa!23$
      # Use single node discovery in order to disable production mode and avoid bootstrap checks.
      # see: https://www.elastic.co/guide/en/elasticsearch/reference/current/bootstrap-checks.html
      discovery.type: single-node
    networks:
      - elk

  logstash:
    build:
      context: logstash/
      args:
        ELK_VERSION: $ELK_VERSION
    volumes:
      - type: bind
        source: ./logstash/config/logstash.yml
        target: /usr/share/logstash/config/logstash.yml
        read_only: true
      - type: bind
        source: ./logstash/pipeline
        target: /usr/share/logstash/pipeline
        read_only: true
    ports:
      - "5044:5044"
      - "5000:5000/tcp"
      - "5000:5000/udp"
      - "9600:9600"
    environment:
      LS_JAVA_OPTS: "-Xmx256m -Xms256m"
    networks:
      - elk
    depends_on:
      - elasticsearch

  kibana:
    build:
      context: kibana/
      args:
        ELK_VERSION: $ELK_VERSION
    volumes:
      - type: bind
        source: ./kibana/config/kibana.yml
        target: /usr/share/kibana/config/kibana.yml
        read_only: true
    ports:
      - "5601:5601"
    networks:
      - elk
    depends_on:
      - elasticsearch

networks:
  elk:
    driver: bridge

volumes:
  elasticsearch:
```  

## 5. docker-stack.yml 수정  
services.elasticsearch.environment.ELASTIC_PASSWORD 수정  
```yaml
version: '3.3'

services:

  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:7.14.0
    ports:
      - "9200:9200"
      - "9300:9300"
    configs:
      - source: elastic_config
        target: /usr/share/elasticsearch/config/elasticsearch.yml
    environment:
      ES_JAVA_OPTS: "-Xmx256m -Xms256m"
      ELASTIC_PASSWORD: sisipapa!23$
      # Use single node discovery in order to disable production mode and avoid bootstrap checks.
      # see: https://www.elastic.co/guide/en/elasticsearch/reference/current/bootstrap-checks.html
      discovery.type: single-node
      # Force publishing on the 'elk' overlay.
      network.publish_host: _eth0_
    networks:
      - elk
    deploy:
      mode: replicated
      replicas: 1

  logstash:
    image: docker.elastic.co/logstash/logstash:7.14.0
    ports:
      - "5044:5044"
      - "5000:5000"
      - "9600:9600"
    configs:
      - source: logstash_config
        target: /usr/share/logstash/config/logstash.yml
      - source: logstash_pipeline
        target: /usr/share/logstash/pipeline/logstash.conf
    environment:
      LS_JAVA_OPTS: "-Xmx256m -Xms256m"
    networks:
      - elk
    deploy:
      mode: replicated
      replicas: 1

  kibana:
    image: docker.elastic.co/kibana/kibana:7.14.0
    ports:
      - "5601:5601"
    configs:
      - source: kibana_config
        target: /usr/share/kibana/config/kibana.yml
    networks:
      - elk
    deploy:
      mode: replicated
      replicas: 1

configs:

  elastic_config:
    file: ./elasticsearch/config/elasticsearch.yml
  logstash_config:
    file: ./logstash/config/logstash.yml
  logstash_pipeline:
    file: ./logstash/pipeline/logstash.conf
  kibana_config:
    file: ./kibana/config/kibana.yml

networks:
  elk:
    driver: overlay
```  

## 6. Docker-Compose 실행 및 종료  
```shell
# 실행
$ docker-compose build && docker-compose up -d  

# 종료
$ docker-compose down -v  
```  

## 7. 키바나 접속
- Elasticsearch : 9200, 9300  
- Logstash : 5044, 5000, 9600
- Kibana : 5601  

http://{kibana-server-ip}:5601 웹브라우저 접속.  
<img src="https://sisipapa.github.io/assets/images/posts/kibana.PNG" >  

## 참고  
[Git Repository - docker-elk](https://github.com/deviantony/docker-elk.git)  
[Docker로 ELK스택 설치하기](https://velog.io/@dion/Docker%EB%A5%BC-%EC%9D%B4%EC%9A%A9%ED%95%B4%EC%84%9C-ELK-%EC%8A%A4%ED%83%9D-%EC%84%A4%EC%B9%98%ED%95%98%EA%B8%B0)    
[MSA 와 Log - 중앙 집중식 로깅 ELK stack 편](https://bravenamme.github.io/2021/01/28/elk-stack/)    

## Github
<https://github.com/sisipapa/ELK-7.1.4-Docker-Compose.git>  