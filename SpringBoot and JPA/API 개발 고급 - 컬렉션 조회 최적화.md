[실전 스프링 부트와 JPA 활용 2 - API 개발과 성능 최적화](https://www.inflearn.com/course/%EC%8A%A4%ED%94%84%EB%A7%81%EB%B6%80%ED%8A%B8-JPA-API%EA%B0%9C%EB%B0%9C-%EC%84%B1%EB%8A%A5%EC%B5%9C%EC%A0%81%ED%99%94)을 수강하며 간단히 정리한 내용이다.

- 주문내역에서 추가로 주문한 상품 정보를 추가로 조회하기
- Order 기준으로 컬렉션인 orderItem과 Item이 필요
- 이번에는 컬렉션인 일대다 관계를 조회하고 최적화하는 방법 알아보기
### 주문 조회 V1: 엔티티 직접 노출
***
```
@RestController
@RequiredArgsConstructor
public class OrderApiController {
    
    private final OrderRepository orderRepository;
    
    @GetMapping("/api/v1/orders")
    public List<Order> ordersV1() {
        List<Order> all = orderRepository.findAll();
        for (Order order : all) {
            order.getMember().getName(); //Lazy 강제 초기화
            order.getDelivery().getAddress(); //Lazy 강제 초기화
            List<OrderItem> orderItems = order.getOrderItems();
            orderItems.stream().forEach(o -> o.getItem().getName()); //Lazy 강제 초기화
        }
        return all;
    }
}
```
- Hibernate5Module 모듈 등록, LAZY=null 처리
- 엔티티가 변하면 API 스펙이 변함
- 트랜잭션 안에서 지연 로딩 필요
- 양방향 연관관계 문제 -> @JsonIgnore
- 엔티티 직접 노출로 좋은 방법 아님

### 주문 조회 V2: 엔티티를 DTO로 변환(fetch join 사용X)
```
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
    private LocalDateTime orderDate;
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
    private String itemName;
    private int orderPrice;
    private int count;
    
    public OrderItemDto(OrderItem orderItem) {
        itemName = orderItem.getItem().getName();
        orderPrice = orderItem.getOrderPrice();
        count = orderItem.getCount();
    }
}
```
- 트랜잭션 안에서 지연 로딩 필요
- 지연 로딩으로 너무 많은 SQL 실행

### 주문 조회 V3: 엔티티를 DTo로 변환 - 패치 조인 최적화(fetch join 사용O)
***
```
@GetMapping("/api/v3/orders")
public List<OrderDto> ordersV3() {
    List<Order> orders = orderRepository.findAllWithItem();
    List<OrderDto> result = orders.stream()
        .map(o -> new OrderDto(o))
        .collect(toList());
    return result;
}
```
```
//OrderRepository에 추가
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
- 패치 조인으로 SQL 1번만 실행됨
- distinct 를 사용한 이유는 1대다 조인이 있으므로 데이터베이스 row가 증가. 그 결과 같은 order
엔티티의 조회 수도 증가하게 됨. JPA의 distinct는 SQL에 distinct를 추가하고, 더해서 같은 엔티티가
조회되면, 애플리케이션에서 중복을 걸러줌. 이 예에서 order가 컬렉션 페치 조인 때문에 중복 조회 되는
것을 막아줌
- 단점: 페이징 불가능
- 참고: 컬렉션 패치 조인은 1개만 사용 가능. 컬렉션 둘 이상에 패치 조인 사용x

### 주문 조회 V3.1: 엔티티를 DTO로 변환 - 페이징과 한계 돌파
***
- 컬렉션을 패치 조인하면 페이징 불가능
    * 컬렉션 패치조인하면 일대다 조인이 발생하므로 데이터가 예측할 수 없이 증가함
    * 일대다에서 일을 기준으로 페이징을 하는 것이 목적이지만 데이터는 다(N)을 기준으로 row가 생성됨
    * Order를 기준으로 페이징하고 싶은데 다(N)인 OrderItem을 조인하면 OrderItem이 기준이 되어버린느 것
- -> 페이징 컬렉션 엔티티 함께 조회하려면?
    * 먼저 ToOne(OneToOne, ManyToOne) 관계는 모두 패치조인. ToOne관계는 페이징 쿼리에 영황 주지 않음
    * 컬렉션은 지연로딩으로 조회
    * 지연로딩 최적화를 위해 hibernate.default_batch_fetch_size , @BatchSize 를 적용.
    * hibernate.default_batch_fetch_size: 글로벌 설정 (```application.yml```)
    * @BatchSize: 개별 최적화 
    * 크기는 적당한 사이즈를 골라야 하는데, 100~1000 사이를 선택하는 것을 권장
    * 이 옵션을 사용하면 컬렉션이나, 프록시 객체를 한꺼번에 설정한 size 만큼 IN 쿼리로 조회

```
//OrderRepository에 추가
public List<Order> findAllWithMemberDelivery(int offset, int limit) {
    return em.createQuery(
        "select o from Order o" +
        " join fetch o.member m" +
        " join fetch o.delivery d", Order.class)
        .setFirstResult(offset)
        .setMaxResults(limit)
        .getResultList();
}
//" join fetch o.member m join fetch o.delivery d", betchsize 설정했다면 없어도 가능
//근데 패치조인으로 하는게 좋음

```
```
//OrderApiController에 추가
@GetMapping("/api/v3.1/orders")
public List<OrderDto> ordersV3_page(
        @RequestParam(value = "offset", defaultValue = "0") int offset,
        @RequestParam(value = "limit", defaultValue = "100") int limit) {
    
    List<Order> orders = orderRepository.findAllWithMemberDelivery(offset, limit);
    List<OrderDto> result = orders.stream()
        .map(o -> new OrderDto(o))
        .collect(toList());
    return result;
}
```
- 장점:
    * 쿼리 호출 수가 1+N에서 1+1로 최적화
    * 조인보다 DB 데이터 전송량이 최적화
    * 패치 조인 방식과 비교해서 쿼리 호출 수가 약간 증가하지만 DB 데이터 전송량이 감소
    * 컬렉션 패치 조인은 페이징이 불가능하지만 이 방법은 가능
- 결론:
    * ToOne관계는 패치조인으로 쿼리 수 줄이고 나머지는 hibernate.default_batch_fetch_size 로 최적화

### 주문 조회 V4: JPA에서 DTO 직접 조회
***
```
private final OrderQueryRepository orderQueryRepository;

@GetMapping("/api/v4/orders")
public List<OrderQueryDto> ordersV4() {
    return orderQueryRepository.findOrderQueryDtos();
}
```
```
@Repository
@RequiredArgsConstructor
public class OrderQueryRepository {

    private final EntityManager em;
    
    public List<OrderQueryDto> findOrderQueryDtos() {
        //루트 조회(toOne 코드를 모두 한번에 조회)
        List<OrderQueryDto> result = findOrders();
        
        //루프를 돌면서 컬렉션 추가(추가 쿼리 실행)
        result.forEach(o -> {
            List<OrderItemQueryDto> orderItems = findOrderItems(o.getOrderId());
            o.setOrderItems(orderItems);
        });
        return result;
    }
    
    /**
    * 1:N 관계(컬렉션)를 제외한 나머지를 한번에 조회
    */
    private List<OrderQueryDto> findOrders() {
        return em.createQuery(
            "select new jpabook.jpashop.repository.order.query.OrderQueryDto(o.id, m.name, o.orderDate, o.status, d.address)" +
            " from Order o" +
            " join o.member m" +
            " join o.delivery d", OrderQueryDto.class)
            .getResultList();
    }
    
    /**
    * 1:N 관계인 orderItems 조회
    */
    private List<OrderItemQueryDto> findOrderItems(Long orderId) {
        return em.createQuery(
            "select new jpabook.jpashop.repository.order.query.OrderItemQueryDto(oi.order.id, i.name, oi.orderPrice, oi.count)" +
            " from OrderItem oi" +
            " join oi.item i" +
            " where oi.order.id = : orderId",
            OrderItemQueryDto.class)
            .setParameter("orderId", orderId)
            .getResultList();
        }
    }
```
```
//OrderQueryDto
@Data
@EqualsAndHashCode(of = "orderId")
public class OrderQueryDto {

    private Long orderId;
    private String name;
    private LocalDateTime orderDate; //주문시간
    private OrderStatus orderStatus;
    private Address address;
    private List<OrderItemQueryDto> orderItems;
    
    public OrderQueryDto(Long orderId, String name, LocalDateTime orderDate, OrderStatus orderStatus, Address address) {
        this.orderId = orderId;
        this.name = name;
        this.orderDate = orderDate;
        this.orderStatus = orderStatus;
        this.address = address;
    }
}
```
```
//OrderItemQueryDto
@Data
public class OrderItemQueryDto {

    @JsonIgnore
    private Long orderId; //주문번호
    private String itemName;//상품 명
    private int orderPrice; //주문 가격
    private int count; //주문 수량
    
    public OrderItemQueryDto(Long orderId, String itemName, int orderPrice, int count) {
        this.orderId = orderId;
        this.itemName = itemName;
        this.orderPrice = orderPrice;
        this.count = count;
    }
}
```
- 단건 조회할 때 많이 사용하는 방식
- 쿼리: 루트1번, 컬렉션N번 실행
- ToOne관계들을 먼저 조회하고 ToMany관계는 각각 별도로 처리
- row 수가 증가하지 않는 ToOne 관계는 조인으로 최적화하기 쉬우므로 한 번에 조회하고 ToMany 관계를 최적화하기 어려우므로 findOrderItems()같은 별도의 메서드로 조회

### 주문 조회 V5: JPA에서 DTO 직접 조회 - 컬렉션 조회 최적화
***
```
@GetMapping("/api/v5/orders")
public List<OrderQueryDto> ordersV5() {
    return orderQueryRepository.findAllByDto_optimization();
}
```
```
//OrderQueryRepository에 추가

public List<OrderQueryDto> findAllByDto_optimization() {
    //루트 조회(toOne 코드를 모두 한번에 조회)
    List<OrderQueryDto> result = findOrders();
    
    //orderItem 컬렉션을 MAP 한방에 조회
    Map<Long, List<OrderItemQueryDto>> orderItemMap = findOrderItemMap(toOrderIds(result));
    
    //루프를 돌면서 컬렉션 추가(추가 쿼리 실행X)
    result.forEach(o -> o.setOrderItems(orderItemMap.get(o.getOrderId())));
    
    return result;
}

private List<Long> toOrderIds(List<OrderQueryDto> result) {
    return result.stream()
        .map(o -> o.getOrderId())
        .collect(Collectors.toList());
}

private Map<Long, List<OrderItemQueryDto>> findOrderItemMap(List<Long> orderIds) {
    List<OrderItemQueryDto> orderItems = em.createQuery(
        "select new jpabook.jpashop.repository.order.query.OrderItemQueryDto(oi.order.id, i.name, oi.orderPrice, oi.count)" +
        " from OrderItem oi" +
        " join oi.item i" +
        " where oi.order.id in :orderIds", OrderItemQueryDto.class)
        .setParameter("orderIds", orderIds)
        .getResultList();
    
    return orderItems.stream()
       .collect(Collectors.groupingBy(OrderItemQueryDto::getOrderId));
}
```
- 데이터를 한꺼번에 처리할 때 많이 사용하는 방식
- 쿼리: 루트1번, 컬렉션1번
- ToOne 관계들 먼저 조회하고 여기서 얻은 식별자 orderId로 ToMany 관계인 orderItem을 한꺼번에 조회
- MAP 사용해서 매칭 성능 향상

### 주문 조회 V6: JPA에서 DTO로 직접 조회, 플랫 데이터 최적화
***
```
@GetMapping("/api/v6/orders")
public List<OrderQueryDto> ordersV6() {
    List<OrderFlatDto> flats = orderQueryRepository.findAllByDto_flat();
    
    return flats.stream()
        .collect(groupingBy(o -> new OrderQueryDto(o.getOrderId(), o.getName(), o.getOrderDate(), o.getOrderStatus(), o.getAddress()),
        mapping(o -> new OrderItemQueryDto(o.getOrderId(), o.getItemName(), o.getOrderPrice(), o.getCount()), toList())
        .map(e -> new OrderQueryDto(e.getKey().getOrderId(), e.getKey().getName(), e.getKey().getOrderDate(), e.getKey().getOrderStatus(), e.getKey().getAddress(), e.getValue()))
        .collect(toList());
}
```
```
//OrderQueryDto에 생성자 추가
public OrderQueryDto(Long orderId, String name, LocalDateTime orderDate, OrderStatus orderStatus, Address address, List<OrderItemQueryDto> orderItems) {
    this.orderId = orderId;
    this.name = name;
    this.orderDate = orderDate;
    this.orderStatus = orderStatus;
    this.address = address;
    this.orderItems = orderItems;
}
```
```
//OrderQueryRepository에 추가
public List<OrderFlatDto> findAllByDto_flat() {
    return em.createQuery(
        "select new jpabook.jpashop.repository.order.query.OrderFlatDto(o.id, m.name, o.orderDate, o.status, d.address, i.name, oi.orderPrice, oi.count)" +
        " from Order o" +
        " join o.member m" +
        " join o.delivery d" +
        " join o.orderItems oi" +
        " join oi.item i", OrderFlatDto.class)
        .getResultList();
}
```
```
//OrderFlatDto
@Data
public class OrderFlatDto {

    private Long orderId;
    private String name;
    private LocalDateTime orderDate; 
    private Address address;
    private OrderStatus orderStatus;
    private String itemName;
    private int orderPrice;
    private int count; 
    
    public OrderFlatDto(Long orderId, String name, LocalDateTime orderDate, OrderStatus orderStatus, Address address, String itemName, int orderPrice, int count) {
        this.orderId = orderId;
        this.name = name;
        this.orderDate = orderDate;
        this.orderStatus = orderStatus;
        this.address = address;
        this.itemName = itemName;
        this.orderPrice = orderPrice;
        this.count = count;
    }
}
```
- 쿼리: 1번
- 단점:
    * 쿼리는 한번이지만 조인으로 인해 DB -> 애플리케이션 전달하는 데이터에 중복 데이터 추가되므로 V5보다 더 느릴 수도 있음
    * 애플리케이션에서 추가 작업이 큼
    * **페이징 불가능**