[자바 ORM 표준 JPA 프로그래밍 - 기본편](https://www.inflearn.com/course/ORM-JPA-Basic)을 수강하며 간단히 정리한 내용이다.

### 객체지향 쿼리 언어 소개
***
- JPA는 다양한 쿼리 방법을 지원한다: JPQL, JPA criteria, QueryDSL, 네이티브 SQL, JDBC API 직접 사용 마이바티스 등

### JPQL
#### JPQL 소개
***
- 가장 단순한 조회 방법 - EntityManager.find(), 객체 그래프 탐색(a.getB().getC())

#### JPQL
***
- JPA를 사용하면 엔티티 객체를 중심으로 개발
- 문제는 검색 쿼리인데, 검색할 때도 테이블이 아닌 엔티티 객체를 대상으로 검색
- 모든 DB 데이터를 객체로 변환해서 검색하는 것은 불가능함
- 애플리케이션이 필요한 데이터만 DB에서 불러오려면 결국 검색 조건이 포함된 SQL이 필요하다
- 이때 JPA는 SQL을 추상화한  JPQL을 제공
- SQL과 문법이 유사함

### JPA Criteria?
***
- 문자가 아닌 자바코드로 JPQL 작성 가능
- JPQL 빌더 역할을 하며  JPA 공식 기능이다
- 그러나 너무 복잡하고 실용성이 없어 대신 QueryDSL 사용을 권장한다

### QueryDSL
***
- 문자가 아닌 자바코드로 JPQL을 작성할 수 있음
- JPQL 빌더 역할을 하며 컴파일 시점에 문법 오류를 찾을 수 있다
- 동적쿼리 작성이 편리하며 단순하고 쉬워 실무에서 사용하기를 권장

### 네이티브 SQL
***
- JPA가 제공하는 SQL을 직접 사용하는 기능
- JPQL로 해결할 수 없는 특정 데이터베이스에 의존적인 기능을 한다

### JPQL - 기본 문법과 기능
#### JPQL 소개
***
- JPQL은 객체지향 쿼리 언어로 엔티티 객체를 대상으로 쿼리한다
- SQL을 추상화해서 특정 데이터베이스 SQL에 의존하지 않지만 결국 SQL로 변환된다

#### JPQL 문법
***
```
select_문 :: =
    select_절
    from_절
    [where_절]
    [groupby_절]
    [having_절]
    [orderby_절]

update_문 :: = update_절 [where_절]
delete_문 :: = delete_절 [where_절]
```
- 엔티티와 속성은 대소문자를 구분하지만 JPQL 키워드는 구분하지 않음
- 엔티티 객체를 대상으로 하기에 테이블 이름이 아닌 엔티티 이름을 사용함
- 별칭은 필수이나 as는 생략이 가능함

#### TypeQuery, Query
***
- TypeQuery: 반환 타입이 명확할 때 사용
```
TypedQuery<Member> query =
    em.createQuery("SELECT m FROM Member m", Member.class);
```
- Query: 반환 타입이 명확하지 않을 때 사용
```
Query query =
    em.createQuery("SELECT m.username, m.age from Member m");
```

#### 결과 조회 API
***
- ```query.getResultList()```: 결과가 하나 이상일 때, 리스트 반환, 결과가 없으면 빈 리스트 반환
- ```query.getSingleResult()```: 결과가 정확히 하나, 단일 객체 반환
    * 결과가 없으면: javax.persistence.NoResultException
    * 둘 이상이면: javax.persistence.NonUniqueResultException
- singleResult는 두 경우 모두 에러가 발생함에 유의

#### 파라미터 바인딩 - 이름 기준, 위치 기준
***
```
SELECT m FROM Member m where m.username=:username

query.setParameter("username", usernameParam);
```
- 위치 기준보다는 이름 기준으로 사용 권장

#### 프로젝션
***
- SELECT 절에 조회할 대상을 지정하는 것
- 프로젝션 대상: 엔티티, 임베디드 타입, 스칼라 타입(숫자, 문자등 기본 데이터 타입)
- 엔티티 프로젝션, 임베디드 타입 프로젝션, 스칼라 타입 프로젝션이 있는데 세가지 모두 내부에서 동작하는 과정이 다르므로 구별해서 알아둬야 함
- DISTINCT로 중복 제거

#### 페이징 API
***
- JPA는 페이징을 다음 두 API로 추상화
- setFirstResult(int startPosition) : 조회 시작 위치(0부터 시작)
- setMaxResults(int maxResult) : 조회할 데이터 수

#### 조인
***
- 내부 조인: SELECT m FROM Member m [INNER] JOIN m.team t
- 외부 조인: SELECT m FROM Member m LEFT [OUTER] JOIN m.team t
- 세타 조인: SELECT count(m) FROM Member m, Team t where m.username = t.name

####
- 조인 - ON절
    * 조인 대상 필터링: ```SELECT m, t FROM Member m LEFT JOIN m.team t on t.name = 'A'```
    * 연관관계 없는 엔티티 외부 조인: ```SELECT m, t FROM Member m LEFT JOIN Team t on m.username = t.name```

#### 서브 쿼리
***
- ex. 나이가 평균보다 많은 회원
```
select m from Member m
    where m.age > (select avg(m2.age) from Member m2)
```

#### JPA 서브 쿼리 한계
***
- JPA는 WHERE, HAVING 절에서만 서브 쿼리 사용 가능하고 SELECT절도 가능(하이버네이트 지원)하나,
- FROM 절의 서브 쿼리는 현재 JPQL에서 불가능하다 -> 조인으로 풀 수 있으면 풀어서 해결 가능

#### JPQL 타입 표현
***
- 문자: ‘HELLO’, ‘She’’s’
- 숫자: 10L(Long), 10D(Double), 10F(Float)
- Boolean: TRUE, FALSE
- ENUM: jpabook.MemberType.Admin (패키지명 포함)
- 엔티티 타입: TYPE(m) = Member (상속 관계에서 사용)

#### 조건식 - CASE 식
***
- COALESCE: 하나씩 조회해서 null이 아니면 반환
- NULLIF: 두 값이 같으면 null 반환, 다르면 첫번째 값 반환

#### JPQL 기본 함수
***
- CONCAT
- SUBSTRING
- TRIM
- LOWER, UPPER
- LENGTH
- LOCATE
- ABS, SQRT, MOD
- SIZE, INDEX(JPA 용도)

#### 사용자 정의 함수 호출
***
하이버네이트는 사용전 방언에 추가해하 하며 사용하는 DB 방언을 상속받고, 사용자 정의 함수를 등록함

### JPQL - 경로 표현식
#### 경로 표현식
***
- 점을 찍어 객체 그래프를 탐색하는 것
- ex. ```select m.username from Member as m```

#### 경로 표현식 용어 정리
***
- 상태 필드(state field): 단순히 값을 저장하기 위한 필드 (ex: m.username)
    * 경로 탐색의 끝이라 더 이상 탐색 안됨. username 다음으로 탐색할 것이 더 이상 없음
- 연관 필드(association field): 연관관계를 위한 필드
    * 단일 값 연관 필드:
  @ManyToOne, @OneToOne, 대상이 엔티티(ex: m.team)
    * 묵시적 내부 조인(inner join) 발생. 탐색이 가능 -> m.team.teamname
    * 컬렉션 값 연관 필드:
  @OneToMany, @ManyToMany, 대상이 컬렉션(ex: m.orders)
    * 묵시적 내부 조인 발생하지만 탐색은 더 이상 안됨
    * -> FROM 절에서 명시적 조인을 통해 별칭을 얻으면 별칭을 통해 탐색 가능

#### 명시적 조인, 묵시적 조인
***
- 명시적 조인: join 키워드 직접 사용 -> ```select m from Member m join m.team t```
- 묵시적 조인: 경로 표현식에 의해 묵시적으로 SQL 조인 발생(내부 조인만 가능) -> ```select m.team from Member m```

#### 경로 표현식 - 예제
***
- select t.members.username from Team t -> 실패
- members는 컬렉션 값이기에 어떤 멤버의 username인지 알 수 없으므로 가져 올 수 없다

#### 경로 탐색을 사용한 묵시적 조인 시 주의사항
***
- 항상 내부 조인
- 컬렉션은 경로 탐색의 끝, 명시적 조인을 통해 별칭을 얻어야 함
- 경로 탐색은 주로 SELECT, WHERE 절에서 사용하지만 묵시적 조인으로 인해 SQL의 FROM (JOIN) 절에 영향을 줌
- 가급적 묵시적 조인 대신 명시적 조인 사용

### JPQL - 패치 조인(fetch join)
#### 패치 조인(fetch join)
***
- SQL 조인 종류가 아니며 JPQL에서 성능 최적화를 위해 제공하는 기능이다
- 연관된 엔티티나 컬렉션을 SQL 한 번에 함께 조회하는 기능이다
- join fetch 명령어 사용

#### 엔티티 패치 조인
***
- 회원을 조회하면서 연관된 팀도 함께 조회함
- [JPQL] select m from Member m join fetch m.team
- 즉시로딩과 같음. 그러나 쿼리로 내가 어떤 객체 그래프를 한번에 조회할 거야 라는 거를 명시적으로 동적인 타이밍에 정할 수 있는 것이 패치조인

#### 패치 조인 사용 코드
***
```
String jpql = "select m from Member m join fetch m.team";
List<Member> members = em.createQuery(jpql, Member.class)
    .getResultList();
for (Member member : members) {
    //페치 조인으로 회원과 팀을 함께 조회해서 지연 로딩X
    System.out.println("username = " + member.getUsername() + ", " +
                        "teamName = " + member.getTeam().name());
}
```

#### 컬렉션 패치 조인
***
- 일대다 관계의 컬렉션 패치 조인
```
[JPQL]
select t
from Team t join fetch t.members
where t.name = ‘팀A'
```
- 문제는 일대다인데 조인으로 하게 되면 뻥튀기가 될 수 있다.
- 팀 입장에서 멤버는 팀A가 두명있음. 따라서 팀A를 두 개 가져와버리게 됨
```
teamname = 팀A, team = Team@0x100
-> username = 회원1, member = Member@0x200
-> username = 회원2, member = Member@0x300
teamname = 팀A, team = Team@0x100
-> username = 회원1, member = Member@0x200
-> username = 회원2, member = Member@0x300
```
- -> DISTINCT로 해결: JPQL의 DISTINCT 2가지 기능 제공 - SQL에 DISTINCT를 추가 + 애플리케이션에서 엔티티 중복 제거

#### 패치 조인과 일반 조인의 차이
***
- 일반 조인 실행시 연관된 엔티티를 함께 조회하지 않음
- 단지 SELECT 절에 지정한 엔티티만 조회할 뿐. 
- 패치 조인을 사용할 때만 연관된 엔티티도 함께 조회(즉시 로딩)
- 패치 조인은 객체 그래프를 SQL 한번에 조회하는 개념

#### 패치 조인의 특징과 한계
***
- 패치 조인 대상에는 별칭을 줄 수 없음(하이버네이트는 가능하지만 사용 안하는 것이 좋음)
- 둘 이상의 컬렉션은 패치 조인 할 수 없다
    * 다대일로 방향 뒤집어서 해결하는 방법 1
    * 방법2.
    ```
    String query = "select f From Team t";
    
    List<Team> result = em.createQuery(query, Team.class)
        .setFirstResult(0)
        .setMaxResults(2)
        .getResultList();
    ```
    * 이거는 쿼리가 루프 돌면서 계속 나와서 성능 안나옴. 그래서 이때 Team 클래스에서 members 에 @BatchSize(size=100) 어노테이션 추가. -> select해서 멤버 가져오는데 한번에 팀A, 팀B 다 가져옴(팀 가져올 때 지연로딩을 끌고 올 때 한번에 100개씩 들고와줘) N+1문제 해결
    * 근데 보통 BatchSize를 글로벌세팅으로 세팅함
    ```
    persistence.xml
    <property name="hibernate.default_batch_fetch_size" value="100"/>
    ```
    * 방법3으로 dto로 따로 만드는 것도 있긴 함
- 컬렉션을 페치 조인하면 페이징 API(setFirstResult, setMaxResults)를 사용할 수 없다.
- 엔티티에 직접 적용하는 글로벌 로딩 전략보다 우선함. 실무에서 글로벌 로딩 전략은 모두 지연로딩. 최적화가 필요한 곳은 패치 조인 적용

#### 패치 조인 - 정리
***
- 모든 것을 패치 조인으로 해결할 수는 없으나 객체 그래프를 유지할 때 사용하면 효과적임
- 여러 테이블을 조인해서 엔티티가 가진 모양이 아닌 전혀 다른 결과를 내야 하면, 페치 조인 보다는 일반 조인을 사용하고 필요한 데이터들만 조회해서 DTO로 반환하는 것이 효과적

### JPQL - 다형성 쿼리
#### TYPE
***
- 조회 대상을 특정 자식으로 한정
- 예) Item 중에 Book, Movie를 조회해라
```
select i from Item i
where type(i) IN (Book, Movie)
```

#### TREAT
***
- 자바의 타입 캐스팅과 유사
- 상속 구조에서 부모 타입을 특정 자식 타입으로 다룰 때 사용
- FROM, WHERE, SELECT(하이버네이트 지원) 사용
- 예) 부모인 Item과 자식 Book이 있다.
```
select i from Item i
where treat(i as Book).auther = ‘kim’
```

### JPQL - 엔티티 직접 사용
#### 엔티티 직접 사용 - 기본 키 값
***
- JPQL에서 엔티티를 직접 사용하면 SQL에서 해당 엔티티의 기본 키 값을 사용
- ```select count(m.id) from Member m``` / ```select count(m) from Member m```
- 이 두 경우 모두 같은 SQL이 실행 됨 -> ```select count(m.id) as cnt from Member m```

### JPQL - Named 쿼리
#### Named 쿼리 - 정적 쿼리
***
- 미리 정의해서 이름을 부여해두고 사용하는 JPQL
- 어노테이션, XML에 정의
- 애플리케이션 로딩 시점에 초기화 후 재사용
- 애플리케이션 로딩 시점에 쿼리를 검증
```
@Entity
@NamedQuery(
    name = "Member.findByUsername",
    query="select m from Member m where m.username = :username")
public class Member {
    ...
}

List<Member> resultList =
    em.createNamedQuery("Member.findByUsername", Member.class)
        .setParameter("username", "회원1")
        .getResultList();
```
- xml에 정의도 가능함
- named 쿼리 환경에 따른 설정: xml이 항상 우선권 가짐. 애플리케이션 운영 환경에 따라 다른 xml 배포 할 수 있음

### JPQL - 벌크 연산
#### 벌크 연산
***
- 재고가 10개 미만인 모든 상품의 가격을 10% 상승하려면?
- JPA 변경 감지 기능으로 실행하려면 너무 많은 SQL이 실행됨
- -> 벌크 연산으로 해결 가능
- 쿼리 한 번으로 여러 테이블 로우 변경(엔티티)
- executeUpdate()의 결과는 영향받은 엔티티 수 반환
- UPDATE, DELETE 지원
- INSERT(insert into .. select, 하이버네이트 지원)
```
String qlString = "update Product p " +
                "set p.price = p.price * 1.1 " +
                "where p.stockAmount < :stockAmount";

int resultCount = em.createQuery(qlString)
                    .setParameter("stockAmount", 10)
                    .executeUpdate();
```

#### 벌크 연산 주의
***
- 벌크 연산은 영속성 컨텍스트를 무시하고 데이터베이스에 직접
쿼리
- 벌크 연산을 먼저 실행
- 벌크 연산 수행 후 영속성 컨텍스트 초기화