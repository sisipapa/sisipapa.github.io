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

## 1. V1: 엔티티 직접 노출
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

## 2. V2: 엔티티를 DTO로 변환  
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

## 3-0. V3: 엔티티를 DTO로 변환 - 페치 조인 최적화  
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

## 3-1. V3.1: 엔티티를 DTO로 변환 - 페이징과 한계 돌파  
Collection fetch join을 하게 되면 페이징이 불가능하다. toMany Collection Join이 있는 Entity의 조회를 위해서는 다음과 같은 순서로 진행한다.  
1. ToOne(OneToOnne, ManyToOnne) 관계를 모두 fetch join한다. ToOne관계의 fetch join의 경우 ROW수 증가가 없으므로 페이징에 영향을 주지 않는다.
2. Collection은 LAZY 로딩으로 조회한다.
3. 지연로딩 성능 최적화를 위해 default_batch_fetch_size, @BatchSize 설정을 추가한다.
- default_batch_fetch_size : 전역설정
- @BatchSize : 개별 최적화설정
- 위 옵션을 적용하면 Collection 또는 Proxy 객체를 한꺼번에 설정한 size 만큼 IN 쿼리로 조회한다.

### OrderApiController  
```java
@GetMapping("/api/v3.1/orders")
public List<OrderDto> orderV3_page(@RequestParam(value = "offset", defaultValue = "0") int offset,
                                   @RequestParam(value = "limit", defaultValue = "100") int limit){
    
    // 1. toOne관계는 fetch join을 하고 페이징처리
    // 2. application.yml default_batch_fetch_size 설정
    // 3. toMany관계는 LAZY 조회! => default_batch_fetch_size 설정으로 Entity마다 1회씩 쿼리가 더 수행
    List<Order> orders = orderRepository.findAllWithMemberDelivery(offset, limit);

    List<OrderDto> result = orders.stream()
            .map(OrderDto::new)
            .collect(Collectors.toList());

    return result;
}
```  

### OrderReposiroty
```java
// Order 기준 ToOne 관계인 Member,Delivery는 fetch join으로 한번에 조회
public List<Order> findAllWithMemberDelivery(int offset, int limit) {
    return em.createQuery(
            "select o from Order o" +
                    " join fetch o.member m" +
                    " join fetch o.delivery d", Order.class)
            .setFirstResult(offset)
            .setMaxResults(limit)
            .getResultList();
}
```  

### application.yml
```yaml
spring:
  jpa:
    properties:
      hibernate:
        default_batch_fetch_size: 1000 
```  
```text
default_batch_fetch_size 의 크기는 적당한 사이즈를 골라야 하는데, 100~1000 사이를 선택하는 것을 권장한다.  
이 전략을 SQL IN 절을 사용하는데, 데이터베이스에 따라 IN 절 파라미터를 1000으로 제한하기도 한다.  
1000으로 잡으면 한번에 1000개를 DB에서 애플리케이션에 불러오므로 DB에 순간 부하가 증가할 수 있다.  
하지만 애플리케이션은 100이든 1000이든 결국 전체 데이터를 로딩해야 하므로 메모리 사용량이 같다.  
1000으로 설정하는 것이 성능상 가장 좋지만, 결국 DB든 애플리케이션이든 순간 부하를 어디까지 견딜 수 있는지로 결정하면 된다.  
```  

## 4. V4: JPA에서 DTO 직접 조회  
- Query: Root Entity 1회, Collection Entity N번 수행
- ToOne(OneToOne, ManyToOne) 관계들을 먼저 조회하고, ToMany(1:N) 관계는 각각 별도로 처리
- Row수 증가가 없는 ToOne관계는 fetch join으로 최적화 하고 ToMany 관계는 별도의 최적화가 힘들기 때문에 LAZY로 조회하는 로직을 구현한다.

### OrderApiController  
```java
@GetMapping("/api/v4/orders")
public List<OrderQueryDto> orderV4(){
    return orderQueryRepository.findOrderQueryDtos();
}
```  

### OrderQueryRepository
```java
@Repository
@RequiredArgsConstructor
public class OrderQueryRepository {

    private final EntityManager em;

    /**
     * 컬렉션은 별도로 조회
     * Query: 루트 1번, 컬렉션 N 번
     * 단건 조회에서 많이 사용하는 방식
     */
    public List<OrderQueryDto> findOrderQueryDtos(){
        List<OrderQueryDto> result = findOrders();

        result.forEach(o -> {
            List<OrderItemQueryDto> orderItems = findOrderItems(o.getOrderId());
            o.setOrderItems(orderItems);
        });

        return result;
    }

    /**
     * 1:N 관계인 orderItems 조회
     */
    private List<OrderItemQueryDto> findOrderItems(Long orderId) {
        return em.createQuery("select " +
                        "new com.example.inflearnjparestapi.repository.order.query.OrderItemQueryDto(oi.order.id, i.name, oi.orderPrice, oi.count) " +
                        "from OrderItem oi " +
                        "join oi.item i " +
                        "where oi.order.id = :orderId", OrderItemQueryDto.class)
                .setParameter("orderId", orderId)
                .getResultList();
    }

    /**
     * 1:N 관계(컬렉션)를 제외한 나머지를 한번에 조회
     */
    private List<OrderQueryDto> findOrders() {
        return em.createQuery("select " +
                        "new com.example.inflearnjparestapi.repository.order.query.OrderQueryDto(o.id, m.name, o.orderDate, o.status, d.address) " +
                        "from Order o " +
                        "join o.member m " +
                        "join o.delivery d ", OrderQueryDto.class)
                .getResultList();
    }
}
```  

## 5. V5: JPA에서 DTO 직접 조회 - 컬렉션 조회 최적화    
- Query: 루트 1번, 컬렉션 1번  
- ToOne 관계들을 먼저 조회하고, 식별자 orderId를 List로 만들어 In 쿼리로 ToMany 관계인 OrderItem 을 한꺼번에 조회  

### OrderApiController
```java
@GetMapping("/api/v5/orders")
public List<OrderQueryDto> orderV5(){
    return orderQueryRepository.findAllByDto_optimization();
}
```  

### OrderQueryRepository
```java
/**
 * Query : 루트1번, 컬렉션 1번
 *
 * @return
 */
public List<OrderQueryDto> findAllByDto_optimization() {
    
    // toOne 1번 쿼리로 조회 
    List<OrderQueryDto> result = findOrders();
    
    // toMany N+1 쿼리가 아닌 in 한번 쿼리로 처리를 위한 id 목록조회
    List<Long> orderIds = findOrderIds(result);
    
    // toMany 컬렉션 IN쿼리로 한번에 조회
    List<OrderItemQueryDto> orderItmes = findOrderItems(orderIds);
    
    // Collectors.groupingBy를 통한 Grouping
    Map<Long, List<OrderItemQueryDto>> orderItemMap = orderItmes.stream()
            .collect(Collectors.groupingBy(orderItemQueryDto -> orderItemQueryDto.getOrderId()));

    result.forEach(o -> o.setOrderItems(orderItemMap.get(o.getOrderId())));

    return result;
}

private List<OrderItemQueryDto> findOrderItems(List<Long> orderIds) {
    return em.createQuery("select " +
                    "new com.example.inflearnjparestapi.repository.order.query.OrderItemQueryDto(oi.order.id, i.name, oi.orderPrice, oi.count) " +
                    "from OrderItem oi " +
                    "join oi.item i " +
                    "where oi.order.id in :orderIds", OrderItemQueryDto.class)
            .setParameter("orderIds", orderIds)
            .getResultList();
}

private List<Long> findOrderIds(List<OrderQueryDto> result) {
    return result.stream()
            .map(o -> o.getOrderId())
            .collect(Collectors.toList());
}
```  

## 6. V6: JPA에서 DTO로 직접 조회, 플랫 데이터 최적화  
- 쿼리가 한번만 수행된다.    
- 쿼리가 한번이지만 조인으로 인해 DB에서 어플리케이션으로 전달하는 데이터가 중복이 있어서 V5 보다 느릴 수 있다.  
- DB에서 flat하게 데이터를 조회해서 어플리케이션에서 grouping을 해주기 때문에 어플리케이션에서 추가 작업이 발생한다.  
- 페이징이 불가능하다.  

### OrderApiController  
```java
@GetMapping("/api/v6/orders")
public List<OrderQueryDto> orderV6(){
    List<OrderFlatDto> flats = orderQueryRepository.findAllBy_flat();

    return flats.stream()
            .collect(groupingBy(o -> new OrderQueryDto(o.getOrderId(), o.getName(), o.getOrderDate(), o.getOrderStatus(), o.getAddress()),
                    mapping(o -> new OrderItemQueryDto(o.getOrderId(), o.getItemName(), o.getOrderPrice(), o.getCount()), toList())
            )).entrySet().stream()
            .map(e -> new OrderQueryDto(e.getKey().getOrderId(),
                                    e.getKey().getName(),
                                    e.getKey().getOrderDate(),
                                    e.getKey().getOrderStatus(),
                                    e.getKey().getAddress(),
                                    e.getValue()))
            .collect(toList());
}
```  

### OrderQueryRepository  
```java
public List<OrderFlatDto> findAllBy_flat() {
    return em.createQuery(
            "select " +
                    "new com.example.inflearnjparestapi.repository.order.query.OrderFlatDto(o.id, m.name, o.orderDate, o.status, d.address, i.name, oi.orderPrice, oi.count) " +
                    "from Order o " +
                    "join o.member m " +
                    "join o.delivery d " +
                    "join o.orderItems oi " +
                    "join oi.item i", OrderFlatDto.class
    ).getResultList();
}
```  

### OrderQueryDto 생성자 추가  
```java
public OrderQueryDto(Long orderId, String name, LocalDateTime orderDate, OrderStatus orderStatus, Address address, List<OrderItemQueryDto> orderItems){
    this.orderId = orderId;
    this.name = name;
    this.orderDate = orderDate;
    this.orderStatus = orderStatus;
    this.address = address;
    this.orderItems = orderItems;
}
```  

### OrderFlatDto
```java
@Data
@AllArgsConstructor
public class OrderFlatDto {

    private Long orderId;
    private String name;
    private LocalDateTime orderDate;
    private OrderStatus orderStatus;
    private Address address;

    private String itemName;
    private int orderPrice;
    private int count;
}
```  

## 권장순서
1. fetch join으로 쿼리수 최적화
   1. 페치조인으로 쿼리 수를 최적화
   2. 컬렉션 최적화
       1. 페이징 있는 경우 -  hibernate.default_batch_fetch_size, @BatchSize 로 최적화
       2. 페이징 없는 경우 - fetch join 사용
2. Entity 조회 방식으로 해결이 안되면 DTO 조회 방식 사용한다.
3. DTO 조회방식으로 해결이 안되면 NativeSQL or 스프링 JdbcTemplate  



## 참고  
[실전! 스프링 부트와 JPA 활용2 - API 개발과 성능 최적화](https://www.inflearn.com/course/%EC%8A%A4%ED%94%84%EB%A7%81%EB%B6%80%ED%8A%B8-JPA-API%EA%B0%9C%EB%B0%9C-%EC%84%B1%EB%8A%A5%EC%B5%9C%EC%A0%81%ED%99%94/)  

## Github  
<https://github.com/sisipapa/inflearn-jpa-rest-api.git>  



