[실전 스프링 데이터 JPA](https://www.inflearn.com/course/%EC%8A%A4%ED%94%84%EB%A7%81-%EB%8D%B0%EC%9D%B4%ED%84%B0-JPA-%EC%8B%A4%EC%A0%84)을 수강하며 간단히 정리한 내용이다.

### 스프링 데이터 JPA 구현체 분석
***
* 스프링 데이터 JPA가 제공하는 공통 인터페이스의 구현체
* ```org.springframework.data.jpa.repository.support.SimpleJpaRepository```
* ```@Repository``` 적용: JPA 예외를 스프링이 추상화한 예외로 변환
* ```@Transactional``` 트랜잭션 적용
* **```save()``` 메서드**: 새로운 엔티티면 저장(persist), 새로운 엔티티가 아니면 병합(merge)

### 새로운 엔티티 구별하는 방법
***
- 새로운 엔티티를 판단하는 기본 전략
    * 식별자가 객체일 때 null 로 판단 
    * 식별자가 자바 기본 타입일 때 0 으로 판단 
    * ```Persistable``` 인터페이스를 구현해서 판단 로직 변경 가능
#####
- JPA 식별자 생성 전략이 ```@GenerateValue``` 면 ```save()``` 호출 시점에 식별자가 없으므로 새로운 엔티티로 인식해서 정상 동작
- 그런데 JPA 식별자 생성 전략이 ```@Id``` 만 사용해서 직접 할당이면 이미 식별자 값이 있는 상태로 ```save()```를 호출
- 따라서 이 경우 ```merge()```가 호출
- ```merge()```는 우선 DB를 호출해서 값을 확인하고, DB에 값이 없으면 새로운 엔티티로 인지하므로 매우 비효율적
- 따라서 ```Persistable``` 를 사용해서 새로운 엔티티 확인 여부를 직접 구현하게는 효과적
- 등록시간(```@CreatedDate```)을 조합해서 사용하면 이 필드로 새로운 엔티티 여부를 편리하게 확인 가능(@CreatedDate에 값이 없으면 새로운 엔티티로 판단)
- 예제
```
@Entity
@EntityListeners(AuditingEntityListener.class)
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class Item implements Persistable<String> {
    @Id
    private String id;
    
    @CreatedDate
    private LocalDateTime createdDate;
    
    public Item(String id) {
        this.id = id;
    }
    
    @Override
    public String getId() {
        return id;
    }
    
    @Override
    public boolean isNew() {
        return createdDate == null;
    }
}
```