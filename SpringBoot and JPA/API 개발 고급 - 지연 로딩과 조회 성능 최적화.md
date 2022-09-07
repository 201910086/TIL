[실전 스프링 부트와 JPA 활용 2 - API 개발과 성능 최적화](https://www.inflearn.com/course/%EC%8A%A4%ED%94%84%EB%A7%81%EB%B6%80%ED%8A%B8-JPA-API%EA%B0%9C%EB%B0%9C-%EC%84%B1%EB%8A%A5%EC%B5%9C%EC%A0%81%ED%99%94)을 수강하며 간단히 정리한 내용이다.

- 주문 + 배송정보 + 회원을 조회하는 API 만들기
- 지연 로딩으로 인해 발생하는 성능 문제 단계적으로 해결하기
### 간단한 주문 조회 V1: 엔티티를 직접 노출(먼저 다대일 관계)
***
```
@RestController
@RequiredArgsConstructor
public class OrderSimpleApiController {

    private final OrderRepository orderRepository;
    
    @GetMapping("/api/v1/simple-orders")
    public List<Order> ordersV1() {
        List<Order> all = orderRepository.findAllByString(new OrderSearch());
        for (Order order : all) {
            order.getMember().getName(); //Lazy 강제 초기화
            order.getDelivery().getAddress(); //Lazy 강제 초기화
        }
        return all;
    }
}
```
1. return all만 한다면 무한루프 발생.
(member로 가면 orders가 또 있어서 orders를 또 호출하고 등등 무한으로 돎)
-> 양방향 연관관계가 걸린 곳은 꼭! 한곳을 @JsonIgnore 달아주기
2. LAZY로 되어 있어 진짜 DB에서 안가져옴. 따라서 컨트롤러에서 LAZY 강제 초기화해야 함
- 엔티티를 직접 노출하는 것은 좋지 않음
- order member 와 order address 는 지연 로딩. 따라서 실제 엔티티 대신 프록시 존재
- Hibernate5Module 을 스프링 빈으로 등록하면 해결되긴 함
```
//JpashopApplication에 추가
@Bean
Hibernate5Module hibernate5Module() {
return new Hibernate5Module();
}
//기본적으로 초기화 된 프록시 객체만 노출, 초기화 되지 않은 프록시 객체는 노출 안함

//build.gradle에 다음과 같이 추가해야 함
implementation 'com.fasterxml.jackson.datatype:jackson-datatype-hibernate5'
```
- 주의: 지연 로딩(LAZY)을 피하기 위해 즉시 로딩(EARGR)으로 설정하면 안됨. 즉시 로딩 때문에
연관관계가 필요 없는 경우에도 데이터를 항상 조회해서 성능 문제가 발생 -> 항상 지연 로딩을 기본으로 하고, 성능 최적화가 필요한 경우에는 페치 조인(fetch join)을 사용

### 간단한 주문 조회 V2: 엔티티를 DTO로 변환(fetch join 사용x)
```
@GetMapping("/api/v2/simple-orders")
public List<SimpleOrderDto> ordersV2() {
    List<Order> orders = orderRepository.findAll();
    List<SimpleOrderDto> result = orders.stream()
        .map(o -> new SimpleOrderDto(o))
        .collect(toList());
    return result;
}

@Data
static class SimpleOrderDto {

    private Long orderId;
    private String name;
    private LocalDateTime orderDate; //주문시간
    private OrderStatus orderStatus;
    private Address address;

    public SimpleOrderDto(Order order) {
        orderId = order.getId();
        name = order.getMember().getName();
        orderDate = order.getOrderDate();
        orderStatus = order.getStatus();
        address = order.getDelivery().getAddress();
    }
}
```
- 엔티티를 DTO로 변환하는 일반적인 방법
- 쿼리가 총 1 + N + N번 실행됨
    * order 조회 1번
    * order -> member 지연로딩 조회 N번
    * order -> delivery 지연로딩 조회 N번

### 간단한 주문 조회 V3: 엔티티를 DTO로 변환 - 패치 조인 최적화(fetch join 사용o)
***
```
@GetMapping("/api/v3/simple-orders")
public List<SimpleOrderDto> ordersV3() {
    List<Order> orders = orderRepository.findAllWithMemberDelivery();
    List<SimpleOrderDto> result = orders.stream()
        .map(o -> new SimpleOrderDto(o))
        .collect(toList());
    return result;
}

//orderRepository에 코드 추가
public List<Order> findAllWithMemberDelivery() {
    return em.createQuery("select o from Order o" +
        " join fetch o.member m" +
        " join fetch o.delivery d", Order.class)
        .getResultList();
}
```
- 엔티티를 패치 조인을 사용해서 쿼리 1번에 조회
- 패치 조인으로 order -> member, order -> delivery는 이미 조회된 상태이므로 지연로딩x

### 간단한 주문 조회 V4: JPA에서 DTO로 바로 조회
***
```
private final OrderSimpleQueryRepository orderSimpleQueryRepository; //의존관계 주입

@GetMapping("/api/v4/simple-orders")
public List<OrderSimpleQueryDto> ordersV4() {
    return orderSimpleQueryRepository.findOrderDtos();
}
```
```
//OrderSimpleQueryRepository 조회 전용 리포지토리
@Repository
@RequiredArgsConstructor
public class OrderSimpleQueryRepository {

    private final EntityManager em;
    
    public List<OrderSimpleQueryDto> findOrderDtos() {
        return em.createQuery(
            "select new jpabook.jpashop.repository.order.simplequery.OrderSimpleQueryDto(o.id, m.name, o.orderDate, o.status, d.address)" +
            " from Order o" +
            " join o.member m" +
            " join o.delivery d", OrderSimpleQueryDto.class)
            .getResultList();
    }
}
```
```
//OrderSimpleQueryDto 리포지토리에서 DTO 직접 조회
@Data
public class OrderSimpleQueryDto {
    private Long orderId;
    private String name;
    private LocalDateTime orderDate; //주문시간
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
- 일반적인 SQL을 사용할 때처럼 원하는 값을 선택해서 조회함
- new 명령어로 JPQL의 결과를 DTO로 즉시 변환
- SELECT 절에서 원하는 데이터 직접 선택 DB -> 애플리케이션 네트웍 용량 최적화(그러나 생각보다 미비함)
- 단점: 리포지토리 재사용성 V3에 비해 떨어짐. API 스펙에 맞춘 코드가 리포지토리에 들어가는 단점

### 정리
***
- 엔티티를 DTO로 변환하거나 DTO로 바로 조회하는 두 가지 방법은 각각 장단점 가짐.
- 권장 방법:
1. 우선 엔티티를 DTO로 변환하는 방법 선택
2. 필요하면 패치 조인으로 성능을 최적화 -> 대부분의 성능 이슈 해결
3. 그래도 안되면 DTO로 직접 조회하는 방법 사용
4. 최후의 방법은 JPA가 제공하는 네이티브 SQL이나 스프링 JDBC Template 사용해서 SQL 직접 사용