[자바 ORM 표준 JPA 프로그래밍 - 기본편](https://www.inflearn.com/course/ORM-JPA-Basic)을 수강하며 간단히 정리한 내용이다.

### JPA 소개

#### JPA?
***
- Java Persistence API. 자바 진영의 ORM 기술 표준
- 애플리케이션과 JDBC 사이에서 동작
- 과거와는 다르게 JPA가 JDBC API를 이용해 SQL을 DB로 넘겨줌

#### ORM?
***
- Object-relational mapping(객체 관계 매핑)
- 객체는 객체대로 설계하고, 관계형 DB는 관계형 DB대로 설계
- ORM 프레임워크가 중간에서 매핑함
- 대중적인 언어는 대부분 ORM 기술 존재
- ORM은 객체와 RDB 기둥 위에 있는 기술

#### JPA는 표준 명세
***
- JPA는 인터페이스의 모음
- JPA 2.1 표준 명세를 구현한 3가지 구현체 -> 하이버네이트, EclipseLink, DataNucleus

#### JPA를 사용해야 하는 이유
- SQL 중심적인 개발에서 객체 중심으로 개발
- 생산성
    * 저장, 조회, 수정, 삭제 모두 메서드가 다 만들어져 있어 사용하기만 하면 됨(ex. 저장: jpa.persist(member))
- 유지보수
    * 과거에는 필드 변경 시 모든 SQL을 수정해야 했지만 JPA 사용 시 필드만 추가하면 SQL은 JPA가 처리해줌
- 패러다임의 불일치 해결
- 성능
    * 1차 캐시와 동일성 보장: 같은 트랜잭션 안에서는 같은 엔티티를 반환해 약간의 조회 성능을 향상시킴
    * 트랜잭션을 지원하는 쓰기 지연: ex.트랜잭션이 커밋할 때까지 INSERT SQL을 모아 커밋하는 순간 한번에 보냄
    * 지연 로딩: JOIN SQL로 한번에 연관된 객체까지 미리 조회하는 즉시 로딩과 달리 객체가 실제 사용될 때 로딩
- 데이터 접근 추상화와 벤더 독립성
- 표준


### JPA 시작

- 이번 강의에서는 H2 Database와 메이븐(Maven)을 사용.
- maven에서는 pom.xml 파일에 라이브러리 추가
- JPA 설정:
    * /META-INF/persistence.xml (경로)
    * persistence-unit name으로 이름을 지정해줌
```
<?xml version="1.0" encoding="UTF-8"?>
<persistence version="2.2"
    xmlns="http://xmlns.jcp.org/xml/ns/persistence" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/persistence http://xmlns.jcp.org/xml/ns/persistence/persistence_2_2.xsd">
    
    <persistence-unit name="hello">
        <properties>
            <!-- 필수 속성 -->
            <property name="javax.persistence.jdbc.driver" value="org.h2.Driver"/>
            <property name="javax.persistence.jdbc.user" value="sa"/>
            <property name="javax.persistence.jdbc.password" value=""/>
            <property name="javax.persistence.jdbc.url" value="jdbc:h2:tcp://localhost/~/test"/>
            <property name="hibernate.dialect" value="org.hibernate.dialect.H2Dialect"/>
            
            <!-- 옵션 -->
            <property name="hibernate.show_sql" value="true"/>
            <property name="hibernate.format_sql" value="true"/>
            <property name="hibernate.use_sql_comments" value="true"/>
            <!--<property name="hibernate.hbm2ddl.auto" value="create" />-->
        </properties>
    </persistence-unit>
</persistence>
```

#### 데이터베이스 방언?
***
- 각각의 데이터베이스가 제공하는 SQL 문법과 함수는 조금씩 다른데 
- SQL 표준을 지키지 않는 특정 데이터 베이스만의 고유한 기능을 방언이라고 함.
- JPA는 특정 데이터베이스에 종속되지 않기 때문에 -> hibernate.dialect 속성에 지정하면 40가지 이상의 데이터베이스 방언을 지원받을 수 있음

#### 애플리케이션 개발
***
- JPA 구동 방식 
    * Persistence 클래스에서 설정 정보(persistence.xml)를 조회한 후
    * -> EntityManagerFactory 클래스를 생성
    * -> 이후 필요할 때맏다 EntityManagerFactory에서 EntityManager 생성하며 구동

- 객체 테이블 생성 및 매핑
    * ```@Entity```: JPA에 관리할 객체
    * ```@Id```: 데이터베이스 PK와 매핑

- EntityManagerFactory는 하나만 생성해서 애플리케이션 전체에서 공유
- EntityManager는 쓰레드 간에 공유 x(사용하고 버림)
- JPA의 모든 데이터 변경은 트랜잭션 안에서 실행

#### JPQL
***
- JPA가 제공하는 객체 지향 쿼리 언어로, SQL을 추상화한 언어
- SQL과 문법이 유사
- SQL은 데이터베이스 테이블을 대상으로 쿼리하는 반면 JPQL은 **엔티티 객체**를 대상으로 쿼리
- -> 테이블이 아닌 객체를 대상으로 검색하는 객체 지향 쿼리
- SQL을 추상화함으로써 특정 데이터베이스에 의존하지 않음
