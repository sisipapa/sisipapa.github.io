---
layout: post
title: Springboot-JPA-API 개발2
category: [jpa]
tags: [jap, Springboot, api]
redirect_from:

- /2021/11/23/

---

오늘은 API 개발 고급 - 지연 로딩과 조회 성능 최적화 강의를 보면서 내용을 정리해 보려고 한다.   
주문(Order)를 기준으로 oneToOne 관계인 배송정보(Delivery), ManyToOne 관계인 Member Entity를 조회하는 API를 단계적으로 성능 최적화 나가는 과정을 정리할 예정이다.   

## V1: 엔티티를 직접 노출  
- Entity를 조회해서 API의 응답결과로 Entity를 직접 반환해주는 API이다.   
- order -> member 와 order -> address fetchType이 LAZY로 실제 엔티티가 존재하지 않고 프록시가 존재하기 때문에 API 호출 시 오류가 발생한다.  
- Hibernate5Module 을 스프링 빈으로 등록하면 해결은 가능하지만 많이 사용할 일은 없어 보여 따로 정리를 하지는 않았다.  

### OrderSimpleApiController  
```java
/**
 * V1. 엔티티 직접 노출
 * - Hibernate5Module 모듈 등록, LAZY=null 처리
 * - 양방향 관계 문제 발생 -> @JsonIgnore
 */
@GetMapping("/api/v1/simple-orders")
public List<Order> ordersV1() {
    List<Order> all = orderRepository.findAllByString(new OrderSearch());
    for (Order order : all) {
        order.getMember().getName(); //Lazy 강제 초기화
        order.getDelivery().getAddress(); //Lazy 강제 초기환
    }
    return all;
}
```  

## V2: 엔티티를 DTO로 변환  
- V1 API에서 응답결과를 Entity를 DTO로 변환하는 방법이다.  
- 쿼리가 총 1 + N + N번 실행된다.  
order 조회 1번(order 조회 결과 수가 N이 된다.)  
order -> member 지연 로딩 조회 N 번  
order -> delivery 지연 로딩 조회 N 번   

### OrderSimpleApiController    


## V3: 엔티티를 DTO로 변환 - 페치 조인 최적화  
- 엔티티를 fetch join을 사용해서 쿼리 1번에 조회  
- fetch join으로 order -> member , order -> delivery 는 LAZY로 설정을 해도 이미 조회 된 상태이다.  

### OrderSimpleApiController  
```java
/**
 * V3. 엔티티를 조회해서 DTO로 변환(fetch join 사용) - fetch join으로 쿼리 1번 호출
 */
@GetMapping("/api/v3/simple-orders")
public List<SimpleOrderDto> ordersV3() {
    List<Order> orders = orderRepository.findAllWithMemberDelivery();
    List<SimpleOrderDto> result = orders.stream()
    .map(o -> new SimpleOrderDto(o))
    .collect(toList());
    return result;
}
```  

### OrderRepository  
```java
public List<Order> findAllWithMemberDelivery() {
    return em.createQuery
    (
        "select o from Order o" +
        " join fetch o.member m" +
        " join fetch o.delivery d", Order.class
    ).getResultList();
}
```  

## V4: JPA에서 DTO로 바로 조회  
- 일반적인 SQL을 사용할 때 처럼 원하는 컬럼을 선택가 가능하다.  
- new 명령어를 사용해서 JPQL의 결과를 DTO로 즉시 변환
- SELECT 절에서 원하는 데이터를 직접 선택하므로 DB 애플리케이션 네트웍 용량 최적화를 할 수 있지만 생각보다는 효과가 미비하다.
- 리포지토리 재사용성 떨어짐, API 스펙에 맞춘 코드가 리포지토리에 들어가는 단점

### OrderSimpleApiController  
```java
/**
 * V4. JPA에서 DTO로 바로 조회
 * - 쿼리 1번 호출
 * - select 절에서 원하는 데이터만 선택해서 조회
 */
@GetMapping("/api/v4/simple-orders")
public List<OrderSimpleQueryDto> ordersV4() {
    return orderSimpleQueryRepository.findOrderDtos();
}
```  

### OrderRepository  
아래와 같은 화면에 종속된 확장성이 낮은 쿼리는 별도의 Repository로 분리해서 관리해 주면 유지보수 운영상 편리하다.    
```java
public List<OrderSimpleQueryDto> findOrderDtos() {
    return em.createQuery
    (
        "select " +
        "new jpabook.jpashop.repository.order.simplequery.OrderSimpleQueryDto(o.id, m.name, o.orderDate, o.status, d.address)" +
        " from Order o" +
        " join o.member m" +
        " join o.delivery d", OrderSimpleQueryDto.class
    ).getResultList();
}
```  

### OrderSimpleQueryDto 
inner class로 사용하던 SimpleOrderDto와 동일하지만 Controller 외부에서 사용할 때 import가 불편하고 jpql 내에서 생성자에 Order Entity를 바로 사용할 수가 없어 개별 변수로 변경해서 새로 생성하게 되었다.   
```java
@Data
public class OrderSimpleQueryDto {
    private Long orderId;
    private String name;
    private LocalDateTime orderDate; 
    private OrderStatus orderStatus;
    private Address address;
    
    public OrderSimpleQueryDto(Long orderId, String name, LocalDateTime orderDate, OrderStatus orderStatus, Address address) {
        this.orderId = orderId;
        this.name = name;
        this.orderDate = orderDate;
        this.orderStatus = orderStatus;
        this.address = address;
    }
}
```  

## 정리  
엔티티를 DTO로 변환하거나, DTO로 바로 조회하는 두가지 방법은 각각 장단점이 있다. 둘중 상황에 따라서 더 나은 방법을 선택하면 된다. 엔티티로 조회하면 리포지토리 재사용성도 좋고, 개발도 단순해진다. 따라서 강의에서 권장하는 순서는 아래와 같다.  
   
### 쿼리 방식 선택 권장 순서
1. 우선 엔티티를 DTO로 변환하는 방법을 선택한다.  
2. 필요하면 페치 조인으로 성능을 최적화 한다. 대부분의 성능 이슈가 해결된다.  
3. 그래도 안되면 DTO로 직접 조회하는 방법을 사용한다.  
4. 최후의 방법은 JPA가 제공하는 네이티브 SQL이나 스프링 JDBC Template을 사용해서 SQL을 직접 사용한다.  

## 참고  
[실전! 스프링 부트와 JPA 활용2 - API 개발과 성능 최적화](https://www.inflearn.com/course/%EC%8A%A4%ED%94%84%EB%A7%81%EB%B6%80%ED%8A%B8-JPA-API%EA%B0%9C%EB%B0%9C-%EC%84%B1%EB%8A%A5%EC%B5%9C%EC%A0%81%ED%99%94/)  

## Github  
<https://github.com/sisipapa/inflearn-jpa-rest-api.git>  



