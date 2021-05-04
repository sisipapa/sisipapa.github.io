---
layout: post 
title: JPA연관관계-OneToMany
category: [JPA]
tags: [JPA, OneToOne]
redirect_from:

- /2021/04/12/

---

## OneToOne 관계  
OneToOne 관계에서는 반대도 OneToOne 관계가 된다. OneToMany 관계에서는 Many쪽이 항상 외래 키를 가지고 있지만, OneToOne 관계에서는 주 테이블이나 대상 테이블에 외래 키를 둘 수 있어서 개발 시 어느 쪽에 둘지를 선택해야 한다.  

## 주 테이블에 외래키가 있는 단방향 연결
주테이블 : User01  
대상테이블 : Address01  
주테이블 OneToOne 어노테이션 선언하고 JoinColumn으로 대상테이블과 연결하고 User01 객체를 통해 Address01를 조회할 수 있는 구조이다.  
```java  
@Entity
@NoArgsConstructor
@AllArgsConstructor
@Builder
@Getter
@Setter
public class User01 {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "user01_id")
    private Long id;
    private String name;

    @OneToOne
    @JoinColumn(name = "address01_id")
    private Address01 address;
}
```
```java  
@Entity
@NoArgsConstructor
@AllArgsConstructor
@Getter
@Setter
@Builder
public class Address01 {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "address01_id")
    private Long id;
    private String addr;
}
```  
```java  
@Test
public void oneToOneMainOneWay(){
    Address01 address = Address01.builder()
            .addr("서울")
            .build();
    address = address01Repository.save(address);

    User01 user = User01.builder()
            .name("유저1")
            .address(address)
            .build();
    user01Repository.save(user);

    Optional<User01> user01 = user01Repository.findById(1L);
    if(user01.isPresent()) System.out.println(user01.get().getName());
}
```  
  
## 주 테이블에 외래키가 있는 양방향 연결
주테이블 : User02  
대상테이블 : Address02  
주테이블과 대상테이블 사이에 양방향 연관관계를 걸고 주테이블 findById, 대상테이블 findById 조회  
일대일 관계에서 지연 로딩으로 설정을 해도 즉시 로딩이 되는 경우가 있다.  
User02.address02 : 지연로딩    
address02.user02 : 지연로딩 안됨  
프록시의 한계로 인해서 외래 키를 직접 관리하지 않는 일대일 관계에서는 지연 로딩으로 설정을 해도 즉시 로딩이 된다. OneToOne 어노테이션의 기존 fetch 타입은 Eager이다.  
```java  
@Entity
@NoArgsConstructor
@AllArgsConstructor
@Getter
@Setter
@Builder
public class User02 {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "user02_id")
    private Long id;
    private String name;

    @JsonIgnore
    @OneToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "address02_id")
    private Address02 address;
}
```  
```java  
@Entity
@NoArgsConstructor
@AllArgsConstructor
@Getter
@Setter
@Builder
public class Address02 {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "address01_id")
    private Long id;
    private String addr;

    @JsonIgnore
    @OneToOne(mappedBy = "address", fetch = FetchType.LAZY)
    private User02 user;
}
```  
```java  
@Test
    public void oneToOneMainTwoWay(){
        Address02 address = Address02.builder()
                .addr("서울")
                .build();
        address = address02Repository.save(address);

        User02 user = User02.builder()
                .name("유저2")
                .address(address)
                .build();
        address.setUser(user);
        user02Repository.save(user);

        System.out.println("====================================================== 주테이블 조회 시작 ======================================================");
        User02 findUser = user02Repository.findById(1L).get();
        System.out.println("====================================================== 주테이블 조회 종료 ======================================================");
        System.out.println("====================================================== 대상테이블 조회 시작 ======================================================");
        Address02 findAddress = address02Repository.findById(1L).get();
        System.out.println("====================================================== 대상테이블 조회 종료 ======================================================");
    }
```  
### 주 테이블 조회시에는 Lazy Loading이 동작한다.  
```text  
====================================================== 주테이블 조회 시작 ====================================================== 
    select
        user02x0_.user02_id as user1_17_0_,
        user02x0_.address02_id as address3_17_0_,
        user02x0_.name as name2_17_0_ 
    from
        user02 user02x0_ 
    where
        user02x0_.user02_id=?
====================================================== 주테이블 조회 종료 ======================================================

### 주 테이블이 아닌 쪽에서 조회 시 Lazy Loading이 동작하지 않는다.  
====================================================== 대상테이블 조회 시작 ======================================================
2021-05-04 16:10:32.769 DEBUG 35312 --- [           main] org.hibernate.SQL                        : 
    select
        address02x0_.address01_id as address1_1_0_,
        address02x0_.addr as addr2_1_0_ 
    from
        address02 address02x0_ 
    where
        address02x0_.address01_id=?
2021-05-04 16:10:32.771 DEBUG 35312 --- [           main] org.hibernate.SQL                        : 
    select
        user02x0_.user02_id as user1_17_0_,
        user02x0_.address02_id as address3_17_0_,
        user02x0_.name as name2_17_0_ 
    from
        user02 user02x0_ 
    where
        user02x0_.address02_id=?
====================================================== 대상테이블 조회 종료 ======================================================
```  

## Github
<https://github.com/sisipapa/study3.git>  
  
## 참고  
[OneToOne 양방향 매핑과 LazyLoading](https://velog.io/@moonyoung/JPA-OneToOne-%EC%96%91%EB%B0%A9%ED%96%A5-%EB%A7%A4%ED%95%91%EA%B3%BC-LazyLoading)  
[JPA 일대일(1:1) @One-To-One 연관관계](https://blog.advenoh.pe.kr/database/JPA-%EC%9D%BC%EB%8C%80%EC%9D%BC-One-To-One-%EC%97%B0%EA%B4%80%EA%B4%80%EA%B3%84/)  

   