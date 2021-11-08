---
layout: post 
title: Springboot Querydsl정리
category: [QueryDsl]
tags: [QueryDsl, springboot]
redirect_from:

- /2021/08/10/

---

현재 운영중인 시스템에서는 JPA, Querydsl을 사용하지 않고 있다. 언제 사용하게 될 지 모르지만 언제든 사용할 수 있게 Querydsl 사용과 관련된 정리를 해보려고 한다. [lelecoder님 블로그](https://lelecoder.com/145)의 내용이 정리가 잘 되어 있어 참고해서 진행할 예정이다. 

## PreSetting  
### docker mysql 설치
```shell
docker run -d --name test_mysql -p 3306:3306 -e MYSQL_ROOT_PASSWORD=rootpwd mysql:5.7 --character-set-server=utf8mb4 --collation-server=utf8mb4_unicode_ci
```  


### docker mysql 접속 및 database 생성
```shell
C:\Users\user>docker ps -a
CONTAINER ID   IMAGE       COMMAND                  CREATED          STATUS          PORTS                                                  NAMES
5b591d0d3972   mysql:5.7   "docker-entrypoint.s…"   15 seconds ago   Up 13 seconds   0.0.0.0:3306->3306/tcp, :::3306->3306/tcp, 33060/tcp   test_mysql

C:\Users\user>docker exec -ti 5b591d0d3972 bash
root@5b591d0d3972:/# mysql -u root -p
Enter password:
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 2
Server version: 5.7.35 MySQL Community Server (GPL)

Copyright (c) 2000, 2021, Oracle and/or its affiliates.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
4 rows in set (0.00 sec)

mysql> create database testdb;
Query OK, 1 row affected (0.00 sec)
```  

## JPQL과 Querydsl 비교
JPQL과 Querydsl은 동일 sql을 실행한다. JPQL 실행시점에 오류파악이 가능하지만 Querydsl은 컴파일 시점에 오류 파악이 가능하다.  

```java
@Test
@SpringBootTest
public class QuerydslExam1Test {

    @Autowired
    EntityManager entityManager;

    @Autowired
    JPAQueryFactory jpaQueryFactory;

    @Test
    void selectJpql(){
        String queryString = "select m from Member m where m.username = :username";
        String username = "member1";
        Member findMember = entityManager.createQuery(queryString, Member.class)
                .setParameter("username", username)
                .getSingleResult();

        assertThat(findMember.getUsername()).isEqualTo(username);
    }

    @Test
    void selectQuerydsl(){
        String username = "member1";
        QMember m = new QMember("m");
        Member findMember = jpaQueryFactory
                .select(m)
                .from(m)
                .where(m.username.eq(username))
                .fetchOne();

        assertThat(findMember.getUsername()).isEqualTo(username);
    }
}
```  

```sql
select
        member0_.member_id as member_i1_0_,
        member0_.age as age2_0_,
        member0_.team_id as team_id4_0_,
        member0_.username as username3_0_ 
    from
        member member0_ 
    where
        member0_.username=?
```  

## fetch, fetchOne, fetFirst, fetchResults, fetchCount  
- fetch() : 리스트를 조회한다. 만약 데이터가 없으면 빈 리스트 반환한다.   
- fetchOne() : 단 건 조회할 때 사용한다. 결과가 없으면 null을 반환하고, 결과 값이 두 개 이상이면 에러가 발생한다.  
- fetchFirst() : limit 구문을 붙여서 쿼리가 실행되고 단 건 조회할 때 사용한다.  
- fetchResults() : 결과값으로 반환되는 QueryResults에는 페이징 정보와 목록을 가지고 있다.  
- fetchCount() : count 쿼리로 변경되어 실행되어서 결과값으로는 count를 반환한다.  
```java
@SpringBootTest
public class QuerydslExam2Test {

    Logger log = (Logger) LoggerFactory.getLogger(QuerydslExam2Test.class);

    @Autowired
    JPAQueryFactory jpaQueryFactory;

    @Test
    void fetch(){
        List<Member> members = jpaQueryFactory
                .selectFrom(member)
                .fetch();

        assertThat(members.stream().count()).isEqualTo(100);
        log.info("### list size : {}", members.stream().count());
    }

    @Test
    void fetchOne(){
        Member findMember = jpaQueryFactory
                .selectFrom(member)
                .where(member.age.eq(99), member.username.eq("member99"))
                .fetchOne();

        log.info("### age : {}", findMember.getAge());
        log.info("### username : {}", findMember.getUsername());
        assertThat(findMember.getAge()).isEqualTo(99);
        assertThat(findMember.getUsername()).isEqualTo("member99");
    }

    @Test
    void fetchFirst(){
        Member findMember = jpaQueryFactory
                .selectFrom(member)
                .where(member.team.id.eq(2l))
                .fetchFirst();

        log.info("username : {}", findMember.getUsername());
        assertThat(findMember.getUsername()).isEqualTo("member1");

    }

    @Test
    void fetchResults(){
        QueryResults<Member> results = jpaQueryFactory
                .selectFrom(member)
                .fetchResults();

        results.getResults()
                .stream()
                .forEach(eachMember -> {
                        log.info("### username : {}", eachMember.getUsername());
                });

        log.info("total count : {}", results.getTotal());
        assertThat(results.getTotal()).isEqualTo(100);
    }

    @Test
    void fetchCount(){
        long totalCount = jpaQueryFactory
                .selectFrom(member)
                .fetchCount();

        log.info("total count : {}", totalCount);
        assertThat(totalCount).isEqualTo(100);
    }
}
```  

## orderBy, groupBy, paging  
- orderBy : orderBy 메서드 인자에는 여러 인자를 넘긴다. 
- paging : offset, limit를 이용해서 페이징 기능 지원한다. (offset:시작번호, limit:개수)
- groupBy : having도 사용가능하다. Tuple 타입은 Querydsl에서 지원하는 서로 다른 타입을 처리할 수 있는 반환값이다.  

```java
@SpringBootTest
public class QuerydslExam3Test {

    Logger log = (Logger)LoggerFactory.getLogger(QuerydslExam3Test.class);

    @Autowired
    JPAQueryFactory jpaQueryFactory;

    @Test
    void orderBy(){
        List<Member> findMembers = jpaQueryFactory
                .selectFrom(member)
                .where(member.team.id.eq(2l))
                .orderBy(member.age.desc().nullsLast())
                .fetch();

        findMembers.stream().forEach(findMember -> {
            log.info("username : {}, age : {}", findMember.getUsername(), findMember.getAge());
        });
    }

    @Test
    void paging(){
        List<Member> findMembers = jpaQueryFactory
                .selectFrom(member)
                .orderBy(member.id.desc())
                .offset(0)
                .limit(10)
                .fetch();

        var membersUsername = findMembers.stream()
                .map(findMember -> findMember.getUsername())
                .collect(Collectors.toList());

        log.info("member username : {}", membersUsername.toString());
    }

    @Test
    void groupBy(){
        List<Tuple> findMembers = jpaQueryFactory
                .select(team.name, member.age.max())
                .from(member)
                .join(member.team, team)
                .groupBy(team.name)
                .fetch();

        Tuple teamA = findMembers.get(0);
        log.info("team name : {}", teamA.get(0, String.class));
        log.info("team age max : {}", teamA.get(1, Long.class));

        assertThat(teamA.get(team.name)).isEqualTo("Team A");
        assertThat(teamA.get(member.age.max())).isEqualTo(98);

    }

}
```  

## BooleanBuilder 활용  
where절의 처리를 BooleanBuilder를 활용해 처리하고 Projection DTO를 생성해 쿼리 결과를 원하는 필드만 조회하는 예제이다.  

```java
@SpringBootTest
public class QuerydslExam4Test {

    Logger log = (Logger)LoggerFactory.getLogger(QuerydslExam4Test.class);

    @Autowired
    JPAQueryFactory jpaQueryFactory;

    @Test
    void projection(){

        MemberSearchDto searchDto = MemberSearchDto.builder()
                .username("member99")
                .ageLoe(100)
                .ageGoe(90)
                .teamName("Team B")
                .build();
        BooleanBuilder builder = getBooleanBuilder(searchDto);


        List<MemberTeamDto> list = jpaQueryFactory
                .select(new QMemberTeamDto(
                        member.id.as("memberId"),
                        member.username,
                        member.age,
                        team.id.as("teamId"),
                        team.name.as("teamName")))
                .from(member)
                .leftJoin(member.team, team)
                .where(builder)
                .fetch();

        var usernameList = list.stream()
                .map(memberTeamDto -> memberTeamDto.getUsername())
                .collect(Collectors.toList());

        log.info("usernameList : {}", usernameList.toString());

        assertThat(usernameList.size()).isEqualTo(1);

        /*
        select
            member0_.member_id as col_0_0_,
            member0_.username as col_1_0_,
            member0_.age as col_2_0_,
            team1_.team_id as col_3_0_,
            team1_.name as col_4_0_
        from
            member member0_
        left outer join
            team team1_
                on member0_.team_id=team1_.team_id
        where
            member0_.username=?
            and team1_.name=?
            and member0_.age>=?
            and member0_.age<=?
        */
    }

    @Test
    void paging(){

        MemberSearchDto searchDto = MemberSearchDto.builder()
                .ageLoe(100)
                .ageGoe(0)
                .teamName("Team B")
                .build();
        BooleanBuilder builder = getBooleanBuilder(searchDto);
        Pageable pageable = PageRequest.of(0, 10);

        QueryResults<MemberTeamDto> results = jpaQueryFactory
                .select(new QMemberTeamDto(
                        member.id.as("memberId"),
                        member.username,
                        member.age,
                        team.id.as("teamId"),
                        team.name.as("teamName")))
                .from(member)
                .leftJoin(member.team, team)
                .where(builder)
                .orderBy(member.id.desc())
                .offset(pageable.getOffset())
                .limit(pageable.getPageSize())
                .fetchResults();

        var usernameList = results.getResults().stream()
                .map(dto -> dto.getUsername())
                .collect(Collectors.toList());
        long total = results.getTotal();

        log.info("usernameList : {}", usernameList.toString());
        log.info("total : {}", total);

        assertThat(total).isEqualTo(50);

        /**
         * select
         *         member0_.member_id as col_0_0_,
         *         member0_.username as col_1_0_,
         *         member0_.age as col_2_0_,
         *         team1_.team_id as col_3_0_,
         *         team1_.name as col_4_0_
         *     from
         *         member member0_
         *     left outer join
         *         team team1_
         *             on member0_.team_id=team1_.team_id
         *     where
         *         team1_.name=?
         *         and member0_.age>=?
         *         and member0_.age<=?
         *     order by
         *         member0_.member_id desc limit ?
         */
    }

    BooleanBuilder getBooleanBuilder(MemberSearchDto condition){

        BooleanBuilder builder = new BooleanBuilder();

        if (hasText(condition.getUsername())) {
            builder.and(member.username.eq(condition.getUsername()));
        }

        if (hasText(condition.getTeamName())) {
            builder.and(team.name.eq(condition.getTeamName()));
        }

        if (condition.getAgeGoe() != null) {
            builder.and(member.age.goe(condition.getAgeGoe()));
        }

        if (condition.getAgeLoe() != null) {
            builder.and(member.age.loe(condition.getAgeLoe()));
        }

        return builder;

    }
}
```  

## Projection 활용의 다양한 방법  
QueryDsl에서 Projection을 활용해서 필요한 필드만을 query하는 다양한 방법이다.  
```java
@SpringBootTest
public class QuerydslExam5Test {

    Logger log = (Logger)LoggerFactory.getLogger(QuerydslExam5Test.class);

    @Autowired
    JPAQueryFactory jpaQueryFactory;

    /**
     * select 사용
     */
    @Test
    void projection1(){
        List<String> result = jpaQueryFactory
                .select(member.username)
                .from(member)
                .fetch();

        log.info("result : {}", result.toString());
    }

    /**
     * Tuple
     */
    @Test
    void projection2(){
        List<Tuple> result = jpaQueryFactory
                .select(member.username, member.age)
                .from(member)
                .orderBy(member.age.asc())
                .fetch();

        var resultList = result.stream()
                .map(tuple -> tuple.get(0, String.class))
                .collect(Collectors.toList());

        log.info("resultList : {}", resultList);
    }

    /**
     * 프로퍼티 접근 방법
     */
    @Test
    void projection3(){
        List<MemberDto> result = jpaQueryFactory
                .select(Projections.bean(MemberDto.class,
                        member.username,
                        member.age))
                .from(member)
                .fetch();

        var resultList = result.stream()
                .map(dto -> dto.getUsername())
                .collect(Collectors.toList());

        log.info("resultList : {}", resultList);
    }

    /**
     * 필드 접근 방법
     */
    @Test
    void projection4(){
        List<MemberDto> result = jpaQueryFactory
                .select(Projections.fields(MemberDto.class,
                        member.username,
                        member.age))
                .from(member)
                .fetch();

        var resultList = result.stream()
                .map(dto -> dto.getUsername())
                .collect(Collectors.toList());

        log.info("resultList : {}", resultList);
    }

    /**
     * 생성자 접근 방법
     */
    @Test
    void projection5(){
        List<MemberDto> result = jpaQueryFactory
                .select(Projections.constructor(MemberDto.class,
                        member.username,
                        member.age))
                .from(member)
                .fetch();

        var resultList = result.stream()
                .map(dto -> dto.getUsername())
                .collect(Collectors.toList());

        log.info("resultList : {}", resultList);
    }

    /**
     * QueryProjection Annotion
     */
    @Test
    void projection6(){
        List<MemberDto> result = jpaQueryFactory
                .select(new QMemberDto(member.username, member.age))
                .from(member)
                .fetch();

        var resultList = result.stream()
                .map(dto -> dto.getUsername())
                .collect(Collectors.toList());

        log.info("resultList : {}", resultList);
    }
}
```  

## subquery, case활용, 상수 query, concat  
```java
@SpringBootTest
public class QuerydslExam6Test {

    Logger log = (Logger)LoggerFactory.getLogger(QuerydslExam6Test.class);

    @Autowired
    JPAQueryFactory jpaQueryFactory;

    @Test
    void subquery(){
        QMember subQueryMember = new QMember("test");
        Member findMember = jpaQueryFactory
                .selectFrom(member)
                .where(member.age.eq( JPAExpressions.select(subQueryMember.age.max()) .from(subQueryMember) ))
                .fetchOne();

        log.info("username : {}" ,findMember.getUsername());
        assertThat(findMember.getUsername()).isEqualTo("member99");


        /**
         * select
         *         member0_.member_id as member_i1_0_,
         *         member0_.age as age2_0_,
         *         member0_.team_id as team_id4_0_,
         *         member0_.username as username3_0_
         *     from
         *         member member0_
         *     where
         *         member0_.age=(
         *             select
         *                 max(member1_.age)
         *             from
         *                 member member1_
         *         )
         */
    }

    @Test
    void caseSelect(){
        List<String> result = jpaQueryFactory
                .select(member.age .when(10).then("TEN") .when(20).then("TWENTY") .otherwise("ETC"))
                .from(member)
                .fetch();

        log.info("result : {}", result.toString());
    }

    @Test
    void constant(){
        List<Tuple> result = jpaQueryFactory
                .select(member.username, Expressions.constant("TEST"))
                .from(member)
                .fetch();

        log.info("result : {}", result.toString());
    }

    @Test
    void concat(){
        String result = jpaQueryFactory
                .select(member.username.concat("_").concat(member.age.stringValue()))
                .from(member)
                .where(member.username.eq("member99"))
                .fetchOne();

        log.info("result : {}", result);
    }
}
```  

## 참고
[스프링 데이터 JPA와 Querydsl 인프런 강의 정리 - lelecoder](https://lelecoder.com/145)

## Github
<https://github.com/sisipapa/study-Springboot-querydsl.git>   
