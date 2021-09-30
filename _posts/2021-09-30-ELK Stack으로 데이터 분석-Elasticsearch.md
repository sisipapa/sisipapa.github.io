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
API는 Comman에서 curl 요청을 한다. pretty 파라미터를 붙이면 json포맷으로 깔끔하게 출력된다.  

|ElasticSearch|Relational DB|
|:---:|:---:|
|Index|Database|
|Type|Table|
|Document|Row|
|Field|Column|
|Mapping|Schema|

|ElasticSearch|Relational DB|CRUD|
|:---:|:---:|:---:|
|GET|Select|Read|
|PUT|Update|Update|
|POST|Insert|Create|
|DELETE|Delete|Delete|

### [GET] Index 조회 - classes
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

### [PUT] Index 생성 - classes
```shell
$ curl -XPUT http://localhost:9200/classes

{"acknowledged":true,"shards_acknowledged":true,"index":"classes"}

$ curl -XGET http://localhost:9200/classes?pretty
{
  "classes" : {
    "aliases" : { },
    "mappings" : { },
    "settings" : {
      "index" : {
        "routing" : {
          "allocation" : {
            "include" : {
              "_tier_preference" : "data_content"
            }
          }
        },
        "number_of_shards" : "1",
        "provided_name" : "classes",
        "creation_date" : "1633013709858",
        "number_of_replicas" : "1",
        "uuid" : "_KC1HJL3St-eskMyqU5tQw",
        "version" : {
          "created" : "7150099"
        }
      }
    }
  }
}
```  

### [DELETE] Index 삭제 - classes
```shell
$ curl -XDELETE http://localhost:9200/classes
{"acknowledged":true}

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

### [POST] Document 생성(직접입력)
Document는 Index가 있을 때 만들어도 되고 없을 떄도 Index,Type명을 명시해 주면 바로 생성이 가능하다.  
```shell
$ curl -XPOST -H 'Content-Type: application/json' http://localhost:9200/classes/class/1 -d '{"title" : "Algorithm", "professor" : "John"}'
{"_index":"classes","_type":"class","_id":"1","_version":1,"result":"created","_shards":{"total":2,"successful":1,"failed":0},"_seq_no":0,"_primary_term":1}

$ curl -XGET http://localhost:9200/classes/class/1?pretty
{
  "_index" : "classes",
  "_type" : "class",
  "_id" : "1",
  "_version" : 1,
  "_seq_no" : 0,
  "_primary_term" : 1,
  "found" : true,
  "_source" : {
    "title" : "Algorithm",
    "professor" : "John"
  }
}
```  

### [POST] Document 생성(파일)  
```shell
$ echo {"title" : "Math", "professor" : "KKH"} > data.json
$ curl -XPOST -H 'Content-Type: application/json' http://localhost:9200/classes/class/1/ -d @data.json
```  

### [POST] Document 필드추가 - unit  
```shell
$ curl -XPOST -H 'Content-Type: application/json' http://localhost:9200/classes/class/1/_update?pretty -d '{"doc" : {"unit" : 1}}'
{
  "_index" : "classes",
  "_type" : "class",
  "_id" : "1",
  "_version" : 2,
  "result" : "updated",
  "_shards" : {
    "total" : 2,
    "successful" : 1,
    "failed" : 0
  },
  "_seq_no" : 1,
  "_primary_term" : 1
}

$ curl -XGET http://localhost:9200/classes/class/1/?pretty
{
  "_index" : "classes",
  "_type" : "class",
  "_id" : "1",
  "_version" : 2,
  "_seq_no" : 1,
  "_primary_term" : 1,
  "found" : true,
  "_source" : {
    "title" : "Algorithm",
    "professor" : "John",
    "unit" : 1
  }
}
```  


### [POST] Document 필드값 변경 - unit 1에서 2로 변경, professor John에서 John2로 변경  
```shell
$ curl -XPOST -H 'Content-Type: application/json' http://localhost:9200/classes/class/1/_update?pretty -d '{"doc" : {"unit" : 2, "professor" : "John2"}}'
{
  "_index" : "classes",
  "_type" : "class",
  "_id" : "1",
  "_version" : 3,
  "result" : "updated",
  "_shards" : {
    "total" : 2,
    "successful" : 1,
    "failed" : 0
  },
  "_seq_no" : 2,
  "_primary_term" : 1
}

$ curl -XGET http://localhost:9200/classes/class/1/?pretty
{
  "_index" : "classes",
  "_type" : "class",
  "_id" : "1",
  "_version" : 3,
  "_seq_no" : 2,
  "_primary_term" : 1,
  "found" : true,
  "_source" : {
    "title" : "Algorithm",
    "professor" : "John2",
    "unit" : 2
  }
}
```  

### [POST] Document 필드값 변경 - script 사용
```shell
$ curl -XPOST -H 'Content-Type: application/json' http://localhost:9200/classes/class/1/_update?pretty -d '{"script" : "ctx._source.unit += 5"}'
{
  "_index" : "classes",
  "_type" : "class",
  "_id" : "1",
  "_version" : 5,
  "result" : "updated",
  "_shards" : {
    "total" : 2,
    "successful" : 1,
    "failed" : 0
  },
  "_seq_no" : 4,
  "_primary_term" : 1
}

$ curl -XGET http://localhost:9200/classes/class/1/?pretty
{
  "_index" : "classes",
  "_type" : "class",
  "_id" : "1",
  "_version" : 5,
  "_seq_no" : 4,
  "_primary_term" : 1,
  "found" : true,
  "_source" : {
    "title" : "Algorithm",
    "professor" : "John2",
    "unit" : 12
  }
}
```

## 참고  
[ELK 스택 (ElasticSearch, Logstash, Kibana) 으로 데이터 분석](https://www.inflearn.com/course/elk-%EC%8A%A4%ED%83%9D-%EB%8D%B0%EC%9D%B4%ED%84%B0-%EB%B6%84%EC%84%9D/lecture/5498?tab=curriculum)     

