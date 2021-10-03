---
layout: post
title: ELK Stack으로 데이터 분석-kibana
category: [elk]
tags: [elasticsearch, logstash, kibana]
redirect_from:

- /2021/10/01/

---

오늘은 [ELK 스택 (ElasticSearch, Logstash, Kibana) 으로 데이터 분석](https://www.inflearn.com/course/elk-%EC%8A%A4%ED%83%9D-%EB%8D%B0%EC%9D%B4%ED%84%B0-%EB%B6%84%EC%84%9D/) 강의 중 아래 4개를 실습해 보면서 정리를 해보려고 한다.
1. 키바나 디스커버(Kibana discover)   
2. 키바나 비주얼라이즈(Kibana Visualize) - 막대그래프, 파이차트   
3. 키바나 비주얼라이즈(Kibana Visualize) - 타일맵, 지도에 표시, 키바나 대시보드(Dashboard)   

## 1. 키바나 디스커버(Kibana discover)
강의와 동일한 환경 구성을 위해 basketball index를 삭제하고 재생성한다. 그리고 basketball의 mapping도 설정한다. mapping 설정 후 bulk로 데이터를 저장한다.
### 1-1. basketball index 삭제
```shell
$ curl -XDELETE -H 'Content-Type: application/json' http://localhost:9200/basketball
{"acknowledged":true}
```  

### 1-2. basketball index 생성
```shell
$ curl -XPUT -H 'Content-Type: application/json' http://localhost:9200/basketball?pretty
{
  "acknowledged" : true,
  "shards_acknowledged" : true,
  "index" : "basketball"
}
```  

### 1-3. basketball mapping 설정
```shell
$ curl -XPUT -H 'Content-Type: application/json' 'localhost:9200/basketball/record/_mapping?pretty&include_type_name=true' -d @basketball_mapping.json
{
  "acknowledged" : true
}
```  

### 1-4. basketball bulk  
정상 수행 구문  
```shell
$ curl -XPOST -H 'Content-Type: application/json' http://localhost:9200/_bulk?pretty --data-binary  @bulk_basketball.json
```  

위의 bulk 관련 curl 요청시 두가지를 지키지 않으면 아래와 같은 오류가 발생한다.   
1. bulk data가 존재하는 json 파일의 끝에서 반드시 Enter로 줄바꿈을 해준다.  
2. -d 약자옵션이 아닌 --data-binary로 풀네임 옵션을 줘야한다. -d 옵션의 경우는 뉴라인을 보존하지 않고 json format을 지원하지 않는다.  
```json
{
  "error" : {
    "root_cause" : [
      {
        "type" : "illegal_argument_exception",
        "reason" : "The bulk request must be terminated by a newline [\\n]"
      }
    ],
    "type" : "illegal_argument_exception",
    "reason" : "The bulk request must be terminated by a newline [\\n]"
  },
  "status" : 400
}
```  

### 1-5. kibana Discover  
원하는 날짜, 필드조건 필터를 적용해서 데이터를 조회할 수 있다.
<img src="https://sisipapa.github.io/assets/images/posts/kibana-discover.png" >  

현재 보고있는 강의의 kibana 버전과 내가 테스트 진행중인 kibana 버전이 상이해서 강의를 보고 그대로 따라서 진행하기가 어렵다. 따로 공부를 해서 키바나 비주얼라이즈에 대한 정리를 해야 할 것 같다....


## 참고  
[ELK 스택 (ElasticSearch, Logstash, Kibana) 으로 데이터 분석](https://www.inflearn.com/course/elk-%EC%8A%A4%ED%83%9D-%EB%8D%B0%EC%9D%B4%ED%84%B0-%EB%B6%84%EC%84%9D/lecture/5498?tab=curriculum)
[BigData Git Repository](https://github.com/minsuk-heo/BigData)

