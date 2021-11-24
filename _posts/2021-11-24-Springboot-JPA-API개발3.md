---
layout: post
title: Springboot-JPA-API 개발3
category: [jpa]
tags: [jap, Springboot, api]
redirect_from:

- /2021/11/24/

---

오늘은 API 개발 고급 - 컬렉션 조회 최적화 강의를 보면서 내용을 정리해 보려고 한다.   
주문내역에서 추가로 주문한 상품 정보를 Order 기준으로 컬렉션인 OrderItem 와 Item 이 필요하다. 이전 정리에서는 xToOne(OneToOne, ManyToOne) 관계만 있었다. 이번에는 컬렉션인 일대다 관계(OneToMany)를 조회하고, 최적화하는 방법을 단계별로 정리해 볼 예정이다.  

## V1: 엔티티 직접 노출
### OrderApiController  
Entity를 직접 노출하는 방법은 좋지 않다. 절대 쓰지 말것!!!  
```java
/**
 * V1. 엔티티 직접 노출
 * - Hibernate5Module 모듈 등록, LAZY=null 처리
 * - 양방향 관계 문제 발생 -> @JsonIgnore
 */
@GetMapping("/api/v1/orders")
public List<Order> ordersV1() {
    List<Order> all = orderRepository.findAll();
    for (Order order : all) {
        order.getMember().getName(); //Lazy 강제 초기화
        order.getDelivery().getAddress(); //Lazy 강제 초기환
        List<OrderItem> orderItems = order.getOrderItems();
        orderItems.stream().forEach(o -> o.getItem().getName()); //Lazy 강제 초기화
    }
    return all;
}
```  

## V2: 엔티티를 DTO로 변환  
DTO로만 변환을 하고 fetch join을 사용하지 않은 경우이다.  
지연 로딩으로 많은 SQL 수행되어 성능이 좋지 않다.  
Order뿐만 아니라 OrderItem도 DTO 클래스를 만들어 준다.  

### OrderApiController  
```java
@GetMapping("/api/v2/orders")
public List<OrderDto> ordersV2() {
    List<Order> orders = orderRepository.findAll();
    List<OrderDto> result = orders.stream()
    .map(o -> new OrderDto(o))
    .collect(toList());
    return result;
}

@Data
static class OrderDto {
    private Long orderId;
    private String name;
    private LocalDateTime orderDate; //주문시간
    private OrderStatus orderStatus;
    private Address address;
    private List<OrderItemDto> orderItems;
    public OrderDto(Order order) {
        orderId = order.getId();
        name = order.getMember().getName();
        orderDate = order.getOrderDate();
        orderStatus = order.getStatus();
        address = order.getDelivery().getAddress();
        orderItems = order.getOrderItems().stream()
                .map(orderItem -> new OrderItemDto(orderItem))
                .collect(toList());
    }
}
@Data
static class OrderItemDto {
    private String itemName;//상품 명
    private int orderPrice; //주문 가격
    private int count; //주문 수량
    public OrderItemDto(OrderItem orderItem) {
        itemName = orderItem.getItem().getName();
        orderPrice = orderItem.getOrderPrice();
        count = orderItem.getCount();
    }
}
```  

## V3: 엔티티를 DTO로 변환 - 페치 조인 최적화  
fetch join으로 SQL이 한번만 실행된다.  
distinct 를 사용한 이유는 1대다 조인이 있으므로 데이터베이스 row가 증가하고 order Entity 조회 수도 증가한다.  
JPA의 distinct는 SQL에 distinct를 추가하고, 더해서 같은 Entity가 조회되면, 애플리케이션에서 중복을 걸러준다.  

```text
toMany Collection fetch join은 페이징이 불가하다. 하이버에이트는 WARN 로그를 남기면서 모든 데이터를 DB에서 읽어 메모리에서 페이징을 한다. 
OOM 으로 어플리케이션 장애가 올 수 있다. 실제로 findAllWithItem 메소드 쿼리에 setFirstResult(1), setMaxResults(100)를 주고 API를 호출하면 아래와 같은 로그를 확인할 수 있다.  

2021-11-24 14:33:09.875  WARN 3512 --- [nio-8080-exec-4] o.h.h.internal.ast.QueryTranslatorImpl   : HHH000104: firstResult/maxResults specified with collection fetch; applying in memory!
```  

```text
컬렉션 페치 조인은 1개만 사용할 수 있다. 컬렉션 둘 이상에 페치 조인을 사용하면 안된다. 데이터가 부정합하게 조회될 수 있다. 
```  

### OrderApiController  
```java
@GetMapping("/api/v3/orders")
public List<OrderDto> ordersV3() {
    List<Order> orders = orderRepository.findAllWithItem();
    List<OrderDto> result = orders.stream()
    .map(o -> new OrderDto(o))
    .collect(toList());
    return result;
}
```  

### OrderRepository  
```java
public List<Order> findAllWithItem() {
    return em.createQuery(
    "select distinct o from Order o" +
    " join fetch o.member m" +
    " join fetch o.delivery d" +
    " join fetch o.orderItems oi" +
    " join fetch oi.item i", Order.class)
    .getResultList();
}
```  


## 참고  
[실전! 스프링 부트와 JPA 활용2 - API 개발과 성능 최적화](https://www.inflearn.com/course/%EC%8A%A4%ED%94%84%EB%A7%81%EB%B6%80%ED%8A%B8-JPA-API%EA%B0%9C%EB%B0%9C-%EC%84%B1%EB%8A%A5%EC%B5%9C%EC%A0%81%ED%99%94/)  

## Github  
<https://github.com/sisipapa/inflearn-jpa-rest-api.git>  



