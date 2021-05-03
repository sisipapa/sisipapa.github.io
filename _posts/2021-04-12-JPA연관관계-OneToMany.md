---
layout: post 
title: JPA연관관계-OneToMany
category: [JPA]
tags: [JPA, OneToMany]
redirect_from:

- /2021/04/12/

---

## OneToMany 관계  
- 1대N 관계에서는 1이 연관관계의 주인이다.  
- 1 쪽에서 외래키를 관리하겠다는 의미가 된다.  

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



  

## Github
<https://github.com/sisipapa/study3.git>  
  
## 참고  
[JPA - One To Many 단방향의 문제점](https://dublin-java.tistory.com/51)
[@OneToMany, 일대다 관계](https://ict-nroo.tistory.com/125)  