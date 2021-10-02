---
layout: post
title: ELK Stack으로 데이터 분석-Elasticsearch2 
category: [elk]
tags: [elasticsearch, logstash, kibana]
redirect_from:

- /2021/10/01/

---



## 

오늘은 [ELK 스택 (ElasticSearch, Logstash, Kibana) 으로 데이터 분석](https://www.inflearn.com/course/elk-%EC%8A%A4%ED%83%9D-%EB%8D%B0%EC%9D%B4%ED%84%B0-%EB%B6%84%EC%84%9D/) 강의 중 Mapping, Search, Metric Aggregation, Bucket Aggregation 강의를 보면서 정리를 해보려고 한다.  

## 엘라스틱서치 매핑(Mapping)
매핑없이 엘라스틱서치에 데이터 저장은 가능하지만 숫자나 날짜를 매핑없이 저장을 하면 문자열로 저장이 될 수있고 이를 시각화해주는 키바나에서 숫자나 날짜데이터를 사용할 떄 문제가 될 수도 있다.  
ElasticSearch 6.x 이상의 버전을 사용하는 경우 string type을 text로 변경해야 한다. 매핑의 형태는 아래와 같다.    
```json
{
  "class" : {
    "properties" : {
      "title" : {
        "type" : "text"
      },
      "professor" : {
        "type" : "text"
      },
      "major" : {
        "type" : "text"
      },
      "semester" : {
        "type" : "text"
      },
      "student_count" : {
        "type" : "integer"
      },
      "unit" : {
        "type" : "integer"
      },
      "rating" : {
        "type" : "integer"
      },
      "submit_date" : {
        "type" : "date",
        "format" : "yyyy-MM-dd"
      },
      "school_location" : {
        "type" : "geo_point"
      }
    }
  }
}
```  

### mapping API 호출(오류 확인중...)
강사님의 강의에서 사용하는 동일한 json파일을 다운받아서 실행을 했는데 아래와 같은 오류가 났다. Mapping API가 어떻게 돌아가는지 강의 내용을 보고 확인했고 오류는 차차 잡도록 하겠다.   
```javascript
curl -XPUT -H 'Content-Type: application/json' 'http://localhost:9200/classes/class/_mapping?pretty' -d @classesRating_mappgin.json
Warning: Couldn't read data from file "classesRating_mappgin.json", this makes
Warning: an empty POST.
{
  "error" : {
    "root_cause" : [
      {
        "type" : "parse_exception",
        "reason" : "request body is required"
      }
    ],
    "type" : "parse_exception",
    "reason" : "request body is required"
  },
  "status" : 400
}
```  

## 엘라스틱서치 데이터 조회(Search)
search를 확인하기 위해 아래 bulk_basketball.json 파일을 bulk 삽입하려고 한다. 

### bulk_basketball.json
```json
{ "index" : { "_index" : "basketball", "_type" : "record", "_id" : "1" } }
{"team" : "Chicago Bulls","name" : "Michael Jordan", "points" : 30,"rebounds" : 3,"assists" : 4, "submit_date" : "1996-10-11"}
{ "index" : { "_index" : "basketball", "_type" : "record", "_id" : "2" } }
{"team" : "Chicago Bulls","name" : "Michael Jordan","points" : 20,"rebounds" : 5,"assists" : 8, "submit_date" : "1996-10-11"}

```  

### bulk 삽입
```javascript
$ curl -XPOST -H 'Content-Type: application/json' http://localhost:9200/_bulk?pretty --data-binary @simple_basketball.json
{
    "took" : 282,
    "errors" : false,
    "items" : [
    {
        "index" : {
            "_index" : "basketball",
            "_type" : "record",
            "_id" : "1",
            "_version" : 1,
            "result" : "created",
            "_shards" : {
                "total" : 2,
                "successful" : 1,
                "failed" : 0
            },
            "_seq_no" : 0,
            "_primary_term" : 1,
            "status" : 201
        }
    },
    {
        "index" : {
            "_index" : "basketball",
            "_type" : "record",
            "_id" : "2",
            "_version" : 1,
            "result" : "created",
            "_shards" : {
                "total" : 2,
                "successful" : 1,
                "failed" : 0
            },
            "_seq_no" : 1,
            "_primary_term" : 1,
            "status" : 201
        }
    }
]
}
```  

### search 전체조회
```javascript
curl -XGET http://localhost:9200/basketball/record/_search?pretty
{
  "took" : 1,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 2,
      "relation" : "eq"
    },
    "max_score" : 1.0,
    "hits" : [
      {
        "_index" : "basketball",
        "_type" : "record",
        "_id" : "1",
        "_score" : 1.0,
        "_source" : {
          "team" : "Chicago Bulls",
          "name" : "Michael Jordan",
          "points" : 30,
          "rebounds" : 3,
          "assists" : 4,
          "submit_date" : "1996-10-11"
        }
      },
      {
        "_index" : "basketball",
        "_type" : "record",
        "_id" : "2",
        "_score" : 1.0,
        "_source" : {
          "team" : "Chicago Bulls",
          "name" : "Michael Jordan",
          "points" : 20,
          "rebounds" : 5,
          "assists" : 8,
          "submit_date" : "1996-10-11"
        }
      }
    ]
  }
}
```  

### search GET방식 파라미터 조회
```javascript
$ curl -XGET 'http://localhost:9200/basketball/record/_search?pretty&q=rebounds:5'
{
    "took" : 0,
    "timed_out" : false,
    "_shards" : {
    "total" : 1,
        "successful" : 1,
        "skipped" : 0,
        "failed" : 0
},
    "hits" : {
    "total" : {
        "value" : 1,
            "relation" : "eq"
    },
    "max_score" : 1.0,
        "hits" : [
        {
            "_index" : "basketball",
            "_type" : "record",
            "_id" : "2",
            "_score" : 1.0,
            "_source" : {
                "team" : "Chicago Bulls",
                "name" : "Michael Jordan",
                "points" : 20,
                "rebounds" : 5,
                "assists" : 8,
                "submit_date" : "1996-10-11"
            }
        }
    ]
}
}
```  

### search -d(Request BODY) 옵션 조회
```javascript
$ curl -XGET 'localhost:9200/basketball/record/_search -d'
{
    "query" : {
        "term" : {"points" : 30}    
    }
}
```

## 엘라스틱서치 메트릭 어그리게이션(Metric Aggregation)
산술 - AVR, MAX, MIN 값을 구할 때 사용한다.

### 평균값 조회 json
아래는 산술 aggregation에 사용할 json 파일이다.
```javascript
$ cat avg_points_aggs.json
{
    "size" : 0,
    "aggs" : {
        "avg_score" : {
            "avg" : {
                "field" : "points"
            }
        }
    }
}
```  

### 평균값/최대값/최소값/합계 조회
```javascript
$ curl -XGET -H 'Content-Type: application/json' localhost:9200/_search?pretty --data-binary @avg_points_aggs.json
{
    "took" : 2,
    "timed_out" : false,
    "_shards" : {
        "total" : 6,
        "successful" : 6,
        "skipped" : 0,
        "failed" : 0
    },
    "hits" : {
        "total" : {
            "value" : 40,
            "relation" : "eq"
        },
        "max_score" : null,
        "hits" : [ ]
    },
    "aggregations" : {
        "avg_score" : {
            "value" : 25.0
        }
    }
}

$ curl -XGET -H 'Content-Type: application/json' localhost:9200/_search?pretty --data-binary @max_points_aggs.json
{
    "took" : 1,
    "timed_out" : false,
    "_shards" : {
        "total" : 6,
        "successful" : 6,
        "skipped" : 0,
        "failed" : 0
    },
    "hits" : {
        "total" : {
            "value" : 40,
            "relation" : "eq"
        },
        "max_score" : null,
        "hits" : [ ]
    },
    "aggregations" : {
        "max_score" : {
            "value" : 30.0
        }
    }
}

$ curl -XGET -H 'Content-Type: application/json' localhost:9200/_search?pretty --data-binary @min_points_aggs.json
{
    "took" : 1,
    "timed_out" : false,
    "_shards" : {
        "total" : 6,
        "successful" : 6,
        "skipped" : 0,
        "failed" : 0
    },
    "hits" : {
        "total" : {
            "value" : 40,
            "relation" : "eq"
        },
        "max_score" : null,
        "hits" : [ ]
    },
    "aggregations" : {
        "min_score" : {
            "value" : 20.0
        }
    }
}

$ curl -XGET -H 'Content-Type: application/json' localhost:9200/_search?pretty --data-binary @sum_points_aggs.json
{
    "took" : 1,
    "timed_out" : false,
    "_shards" : {
        "total" : 6,
        "successful" : 6,
        "skipped" : 0,
        "failed" : 0
    },
    "hits" : {
        "total" : {
            "value" : 40,
            "relation" : "eq"
        },
        "max_score" : null,
        "hits" : [ ]
    },
    "aggregations" : {
        "sum_score" : {
            "value" : 50.0
        }
    }
}

$ curl -XGET -H 'Content-Type: application/json' localhost:9200/_search?pretty --data-binary @stats_points_aggs.json
{
    "took" : 2,
    "timed_out" : false,
    "_shards" : {
        "total" : 6,
        "successful" : 6,
        "skipped" : 0,
        "failed" : 0
    },
    "hits" : {
        "total" : {
            "value" : 40,
            "relation" : "eq"
        },
        "max_score" : null,
        "hits" : [ ]
    },
    "aggregations" : {
        "stats_score" : {
            "count" : 2,
            "min" : 20.0,
            "max" : 30.0,
            "avg" : 25.0,
            "sum" : 50.0
        }
    }
}
```  

## 엘라스틱서치 버켓 어그리게이션(Bucket Aggregation)  
Relation 데이터베이스의 group by와 같은 기능이라고 생각하면 될 것 같다. 여기서는 curl comman를 사용해서 Bucket Aggregation은 아래 실행 샘플처럼 사용할 수 있다라는 정도만 보고 넘어갈 예정이다.  

### basketball Index 생성
```javascript
$ curl -XPUT -H 'Content-Type: application/json' http://localhost:9200/basketball
{"acknowledged":true,"shards_acknowledged":true,"index":"basketball"}
```  

### basketball Index에 mapping 삽입
```json
{
        "record" : {
                "properties" : {
                        "team" : {
                                "type" : "text",
                                "fielddata" : true
                        },
                        "name" : {
                                "type" : "text",
                                "fielddata" : true
                        },
                        "points" : {
                                "type" : "long"
                        },
                        "rebounds" : {
                                "type" : "long"
                        },
                        "assists" : {
                                "type" : "long"
                        },
                        "blocks" : {
                                "type" : "long"
                        },
                        "submit_date" : {
                                "type" : "date",
                                "format" : "yyyy-MM-dd"
                        }
                }
        }
}
```  

### Bulk로 데이터 삽입
```json
{ "index" : { "_index" : "basketball", "_type" : "record", "_id" : "1" } }
{"team" : "Chicago","name" : "Michael Jordan", "points" : 30,"rebounds" : 3,"assists" : 4, "blocks" : 3, "submit_date" : "1996-10-11"}
{ "index" : { "_index" : "basketball", "_type" : "record", "_id" : "2" } }
{"team" : "Chicago","name" : "Michael Jordan","points" : 20,"rebounds" : 5,"assists" : 8, "blocks" : 4, "submit_date" : "1996-10-13"}
{ "index" : { "_index" : "basketball", "_type" : "record", "_id" : "3" } }
{"team" : "LA","name" : "Kobe Bryant","points" : 30,"rebounds" : 2,"assists" : 8, "blocks" : 5, "submit_date" : "2014-10-13"}
{ "index" : { "_index" : "basketball", "_type" : "record", "_id" : "4" } }
{"team" : "LA","name" : "Kobe Bryant","points" : 40,"rebounds" : 4,"assists" : 8, "blocks" : 6, "submit_date" : "2014-11-13"}

```  
```javascript
$ curl -XPOST -H 'Content-Type: application/json' http://localhost:9200/_bulk?pretty --data-binary @twoteam_basketball.json
```  

### aggregation json 실행
terms_aggs.json
```json
{
	"size" : 0,
	"aggs" : {
		"players" : {
			"terms" : {
				"field" : "team"
			}
		}
	}
}
```
```javascript
$ curl -XPOST -H 'Content-Type: application/json' http://localhost:9200/_search?pretty --data-binary @terms_aggs.json
```

## 참고  
[ELK 스택 (ElasticSearch, Logstash, Kibana) 으로 데이터 분석](https://www.inflearn.com/course/elk-%EC%8A%A4%ED%83%9D-%EB%8D%B0%EC%9D%B4%ED%84%B0-%EB%B6%84%EC%84%9D/lecture/5498?tab=curriculum)
[BigData Git Repository](https://github.com/minsuk-heo/BigData)

