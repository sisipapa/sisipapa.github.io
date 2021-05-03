---
layout: post 
title: JPA연관관계-OneToMany
category: [JPA]
tags: [JPA, OneToMany]
redirect_from:

- /2021/04/12/

---

## OneToMany 관계  
- OneToMany 관계에서는 One이 연관관계의 주인이다.  
- One쪽에서 외래키를 관리하겠다는 의미가 된다.  

## @JoinTable을 사용한 @OneToMany 단방향 연관관계(Team01과 Member01은 1:N관계)
- Team01.java  

```java  
@Entity
@Getter
@Setter
@NoArgsConstructor
public class Team01 {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "team01_id")
    private Long id;
    private String name;

    @JsonIgnore
    @OneToMany(cascade = CascadeType.ALL, orphanRemoval = true)
    private List<Member01> members = new ArrayList<>();

    public void addMember(final Member01 member){
        members.add(member);
    }

    public Team01(String name){
        this.name = name;
    }

}
```  

- Member01.java  

```java  
@Entity
@Getter
@Setter
@NoArgsConstructor
public class Member01 {

    public Member01(String name){
        this.name = name;
    }

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "member01_id")
    private Long id;
    private String name;

}
```  

- OneToManyRepository.java  

```java  
@Repository
@AllArgsConstructor
public class OneToManyRepository {

    private EntityManager em;

    @Transactional
    public void joinTable() {
        Team01 team = new Team01("team01");

        team.addMember(new Member01("member01"));
        team.addMember(new Member01("member02"));
        team.addMember(new Member01("member03"));
        team.addMember(new Member01("member04"));

        em.persist(team);
    }
}

```  

- OneToManyRepositoryTest.java  

```java  
@SpringBootTest
class OneToManyRepositoryTest {

    @Autowired
    OneToManyRepository repository;

    @Test
    public void joinTable(){
        repository.joinTable();
    }
}
```  

- 결과  

```sql    
insert into team01(name) values(?)  
  
insert into member01(name) values(?)  
insert into member01(name) values(?)  
insert into member01(name) values(?)  
insert into member01(name) values(?)  

insert into team01_members(team01_team01_id, members_member01_id) values (?, ?)
insert into team01_members(team01_team01_id, members_member01_id) values (?, ?)
insert into team01_members(team01_team01_id, members_member01_id) values (?, ?)
insert into team01_members(team01_team01_id, members_member01_id) values (?, ?)
```   

OneToMany에서 JoinTable을 사용하면 Team01과 Member01을 저장한 후 매핑테이블에 한 번 더 저장된다. 의도하지 않은 team01_members 관계 테이블이 생성이 된다.

## @JoinColumn을 사용한 @OneToMany 단방향 연관관계(Team02과 Member02은 1:N관계)
- Team02.java

```java  
@Entity
@Getter
@Setter
@NoArgsConstructor
public class Team02 {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "team02_id")
    private Long id;
    private String name;

    @JsonIgnore
    @OneToMany(cascade = CascadeType.ALL, orphanRemoval = true)
    @JoinColumn(name="team02_id")
    private List<Member02> members = new ArrayList<>();

    public void addMember(final Member02 member){
        members.add(member);
    }

    public Team02(String name){
        this.name = name;
    }

}
```  

- Member02.java

```java  
@Entity
@Getter
@Setter
@NoArgsConstructor
public class Member02 {

    public Member02(String name){
        this.name = name;
    }

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "member01_id")
    private Long id;
    private String name;

}
```  

- OneToManyRepository.java

```java  
@Repository
@AllArgsConstructor
public class OneToManyRepository {

    private EntityManager em;

    @Transactional
    public void joinColumn() {
        Team02 team = new Team02("team01");

        team.addMember(new Member02("member01"));
        team.addMember(new Member02("member02"));
        team.addMember(new Member02("member03"));
        team.addMember(new Member02("member04"));

        em.persist(team);
    }
}

```  

- OneToManyRepositoryTest.java

```java  
@SpringBootTest
class OneToManyRepositoryTest {

    @Autowired
    OneToManyRepository repository;

    @Test
    public void joinColumn(){
        repository.joinColumn();
    }
}
```  

- 결과

```sql    
insert into team01(name) values(?)  
  
insert into member01(name) values(?)  
insert into member01(name) values(?)  
insert into member01(name) values(?)  
insert into member01(name) values(?)  

update member02 set team02_id=? where member01_id=?
update member02 set team02_id=? where member01_id=?
update member02 set team02_id=? where member01_id=?
update member02 set team02_id=? where member01_id=?
```   

@JoinColumn을 사용하면 member를 DB에 저장할 때, team를 모르기 때문에 먼저 저장한 후에 update문을 통해서 team_id를 업데이트한다.  

## @OneToMany 양방향 연관관계(Team02과 Member02은 1:N관계)
- Team03.java

```java  
@Entity
@Getter
@Setter
@NoArgsConstructor
public class Team03 {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "team03_id")
    private Long id;
    private String name;

    @JsonIgnore
    @OneToMany(mappedBy = "team03", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<Member03> members = new ArrayList<>();

    public void addMember(final Member03 member){
        members.add(member);
    }

    public Team03(String name){
        this.name = name;
    }

}
```  

- Member03.java

```java  
@Entity
@Getter
@Setter
@NoArgsConstructor
public class Member03 {

    public Member03(String name){
        this.name = name;
    }

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "member03_id")
    private Long id;
    private String name;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "team03_id")
    private Team03 team03;

}
```  

- OneToManyRepository.java

```java  
@Repository
@AllArgsConstructor
public class OneToManyRepository {

    private EntityManager em;

    @Transactional
    public void twoWay() {
        Team03 team = new Team03("team01");

        team.addMember(new Member03("member01"));
        team.addMember(new Member03("member02"));
        team.addMember(new Member03("member03"));
        team.addMember(new Member03("member04"));

        em.persist(team);
    }
}

```  

- OneToManyRepositoryTest.java

```java  
@SpringBootTest
class OneToManyRepositoryTest {

    @Autowired
    OneToManyRepository repository;

    @Test
    public void twoWay(){
        repository.twoWay();
    }
}
```  

- 결과

```sql    
insert into team01(name) values(?)  
  
insert into member03(name, team03_id) values(?, ?)
insert into member03(name, team03_id) values(?, ?)
insert into member03(name, team03_id) values(?, ?)
insert into member03(name, team03_id) values(?, ?) 
```  

mappedBy 속성으로 관계의 주인이 누구인지 알 수 있다. FK를 갖고 있는 엔티티가 관계의 주인이 되는데, Member가 엔티티가 주인이다. 즉, mappedBy를 갖고 있지 않은 엔티티가 주인이다.  

## Github
<https://github.com/sisipapa/study3.git>  
  
## 참고  
[JPA - One To Many 단방향의 문제점](https://dublin-java.tistory.com/51)  
[@OneToMany, 일대다 관계](https://ict-nroo.tistory.com/125)  

   