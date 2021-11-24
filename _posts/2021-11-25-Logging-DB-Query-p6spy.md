---
layout: post
title: Logging-DB-Query-p6spy
category: [jpa]
tags: [jap, p6spy]
redirect_from:

- /2021/11/25/

---

김영한님의 [실전! 스프링 부트와 JPA 활용2 - API 개발과 성능 최적화](https://www.inflearn.com/course/%EC%8A%A4%ED%94%84%EB%A7%81%EB%B6%80%ED%8A%B8-JPA-API%EA%B0%9C%EB%B0%9C-%EC%84%B1%EB%8A%A5%EC%B5%9C%EC%A0%81%ED%99%94) 강의를 보면서 쿼리 로깅을 위해 p6spy 라이브러리를 알고 사용하게 되었는데 내 콘솔창의 쿼리는 한줄로 표현되는데 강의 내용 중에는 pretty하게 정리된 쿼리가 출력이 되는 것을 보고 어떻게 하면 pretty한 쿼리가 출력되는지 찾아보게 되었다.  

기본적으로 querydsl, jpa를 사용하게 되면 application.yml(application.properties)에 설정을 해서 쿼리를 확인하게 된다.  

## 1. application.yml 설정
```yaml
spring:
 jpa:
  properties:
   hibernate:
    show_sql: true
     format_sql: true
      use_sql_comments: true
```  
- show_sql: sql문을 한줄로 출력  
- format_sql: multiLine으로 pretty sql문을 출력  
- use_sql_comments: sql문 주석을 함께 출력  

```yaml
logging:
 level:
  org:
   hibernate:
    type:
     descriptor:
      sql:
       trace
```  
sql문에 어떤 값이 들어가는지 확인을 할 때는 위와 같은 설정을 추가해 주면 된다.  

## 2. p6spy 설정  
build.gradle p6spy 라이브러리 설정을 추가한다.
```properties
implementation 'com.github.gavlyukovskiy:p6spy-spring-boot-starter:1.7.1'
```  
p6spy 라이브러리를 사용해서 쿼리 로깅을 하려면 1번 application.yml에 설정한 쿼리 로깅 설정을 제거해 준다. 두개 다 설정해도 무방하지만 너무 많은 쿼리 로그가 남게 되서 보기 더 힘들 수 있다.  
spy.properties 설정으로 pretty 쿼리 설정을 할 수도 있는 걸로 알고 있는데 나같은 경우는 MessageFormattingStrategy Interface를 구현한 클래스를 만들어서 하는 방법을 참고해 해결하게 되었다.  
### P6spySqlFormatConfiguration
```java
import com.p6spy.engine.logging.Category;
import com.p6spy.engine.spy.appender.MessageFormattingStrategy;
import org.hibernate.engine.jdbc.internal.FormatStyle;

import java.text.SimpleDateFormat;
import java.util.Date;
import java.util.Locale;

public class P6spySqlFormatConfiguration implements MessageFormattingStrategy {
    @Override
    public String formatMessage(int connectionId, String now, long elapsed, String category, String prepared, String sql, String url) {
        sql = formatSql(category, sql);
        Date _now = new Date();

        SimpleDateFormat format1 = new SimpleDateFormat("yy.MM.dd HH:mm:ss");
//        return now + "|" + elapsed + "ms|" + category + "|connection " + connectionId + "|" + P6Util.singleLine(prepared) + sql;
        return format1.format(_now) + "|" + elapsed + "ms" + sql;
    }

    private String formatSql(String category,String sql) {
        if(sql ==null || sql.trim().equals("")) return sql;

        // Only format Statement, distinguish DDL And DML
        if (Category.STATEMENT.getName().equals(category)) {
            String tmpsql = sql.trim().toLowerCase(Locale.ROOT);
            if(tmpsql.startsWith("create") || tmpsql.startsWith("alter") || tmpsql.startsWith("comment")) {
                sql = FormatStyle.DDL.getFormatter().format(sql);
            }else {
                sql = FormatStyle.BASIC.getFormatter().format(sql);
            }
            sql = "|\nHeFormatSql(P6Spy sql,Hibernate format):"+ sql;
        }

        return sql;
    }
}
```  

### PrettySqlMultiLineFormat
```java
import com.p6spy.engine.spy.P6SpyOptions;
import org.springframework.context.annotation.Configuration;

import javax.annotation.PostConstruct;

@Configuration
public class PrettySqlMultiLineFormat {

    @PostConstruct
    public void setLogMessageFormat() {
        P6SpyOptions.getActiveInstance().setLogMessageFormat(P6spySqlFormatConfiguration.class.getName());
    }
}
```  

P6spySqlFormatConfiguration, PrettySqlMultiLineFormat 두개의 클래스를 만들어주고 실행시켜보니 값이 들어가는 pretty한 쿼리가 나오는 것을 확인 할 수 있었다!!  

## 참고  
[[Querydsl] p6spy를 이용한 Log 남기기 - p6spy/ log/ querydsl/ Spring boot/ JPA/ p6spy pretty](https://winteri-i.tistory.com/25)  




