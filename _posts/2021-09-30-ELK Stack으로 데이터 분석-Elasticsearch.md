---
layout: post
title: ELK Stack으로 데이터 분석-Elasticsearch  
category: [elk]
tags: [elasticsearch, logstash, kibana]
redirect_from:

- /2021/09/30/

---

한달 전쯤 Docker-Compose를 활용해서 ELK 로컬 환경을 구성하고 Springboot-MSA 예제에 코드중앙화 관리를 위해 연결하는 실습을 해보았다. 오늘은 ELK 스택 (ElasticSearch, Logstash, Kibana) 으로 데이터 분석 인프런 강의를 보고 ELK에 대해 조금 더 공부를 해보려고 한다.  
환경구성은 아래 ELK Stack Docker-Compose 설치 내용을 그대로 따라하면 된다.  
[ELK Stack Docker-Compose 설치](https://sisipapa.github.io/blog/2021/08/19/ELK-Stack-Docker-Compose-%EC%84%A4%EC%B9%98/)  
[Springboot MSA 구성5 - Logging(ELK 연동)](https://sisipapa.github.io/blog/2021/08/27/Springboot-MSA-%EA%B5%AC%EC%84%B15-Logging(ELK-%EC%97%B0%EB%8F%99)/)  

Docker에 설치된 ElasticSearch에 접속해서 직접 API를 호출해 보려고 한다.  

## Docker-Compose ElasticSearch 접속

### docker ps 명령으로 CONTAINER ID 확인
```shell
$ docker ps
CONTAINER ID   IMAGE                      COMMAND                  CREATED          STATUS          PORTS                                                                                                                                                                        NAMES
6c5d6a6b3ac7   docker-elk_logstash        "/usr/local/bin/dock…"   33 minutes ago   Up 33 minutes   0.0.0.0:5000->5000/tcp, :::5000->5000/tcp, 0.0.0.0:5044->5044/tcp, :::5044->5044/tcp, 0.0.0.0:9600->9600/tcp, 0.0.0.0:5000->5000/udp, :::9600->9600/tcp, :::5000->5000/udp   docker-elk_logstash_1
a8fb561da378   docker-elk_kibana          "/bin/tini -- /usr/l…"   33 minutes ago   Up 33 minutes   0.0.0.0:5601->5601/tcp, :::5601->5601/tcp                                                                                                                                    docker-elk_kibana_1
e32954d58bda   docker-elk_elasticsearch   "/bin/tini -- /usr/l…"   33 minutes ago   Up 33 minutes   0.0.0.0:9200->9200/tcp, :::9200->9200/tcp, 0.0.0.0:9300->9300/tcp, :::9300->9300/tcp                                                                                         docker-elk_elasticsearch_1
```  

### docker exec 명령으로 CONTAINER 내부로 접속한다.
```shell
$ docker exec -ti e32954d58bda bash
bash-4.4# 
```  

## ElasticSearch GET,POST,DELETE,PUT API 호출
API는 Comman에서 curl 요청을 한다.  
|ElasticSearch|Relational DB|CRUD|
|---|---|---|
|GET|Select|Read|
|PUT|Update|Update|
|POST|Insert|Create|
|DELETE|Delete|Delete|

### [GET] classes라는 Index 조회
```shell
$ curl -XGET http://localhost:9200/classes
{"error":{"root_cause":[{"type":"index_not_found_exception","reason":"no such index [classes]","resource.type":"index_or_alias","resource.id":"classes","index_uuid":"_na_","index":"classes"}],"type":"index_not_found_exception","reason":"no such index [classes]","resource.type":"index_or_alias","resource.id":"classes","index_uuid":"_na_","index":"classes"},"status":404}

# pretty 파라미터 
$ curl -XGET http://localhost:9200/classes?pretty
{
  "error" : {
    "root_cause" : [
      {
        "type" : "index_not_found_exception",
        "reason" : "no such index [classes]",
        "resource.type" : "index_or_alias",
        "resource.id" : "classes",
        "index_uuid" : "_na_",
        "index" : "classes"
      }
    ],
    "type" : "index_not_found_exception",
    "reason" : "no such index [classes]",
    "resource.type" : "index_or_alias",
    "resource.id" : "classes",
    "index_uuid" : "_na_",
    "index" : "classes"
  },
  "status" : 404
}
```



## 참고  
[ELK 스택 (ElasticSearch, Logstash, Kibana) 으로 데이터 분석](https://www.inflearn.com/course/elk-%EC%8A%A4%ED%83%9D-%EB%8D%B0%EC%9D%B4%ED%84%B0-%EB%B6%84%EC%84%9D/lecture/5498?tab=curriculum)     

