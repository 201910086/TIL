[스프링 핵심 원리 - 기본편](https://www.inflearn.com/course/%EC%8A%A4%ED%94%84%EB%A7%81-%ED%95%B5%EC%8B%AC-%EC%9B%90%EB%A6%AC-%EA%B8%B0%EB%B3%B8%ED%8E%B8/dashboard)을 수강하며 정리한 내용이다.


### 스프링 컨테이너
***
* ```ApplicationContext```(인터페이스)를 스프링 컨테이너라고 한다.
* 기존은 개발자가 ```AppConfig```로 직접 객체 생성과 DI를 했지만, 이제부터는 스프링 컨테이너를 통해 사용
* 스프링 컨테이너는 ```@Configuration```이 붙은 ```AppConfig```를 설정 정보로 사용
* 이때 ```@Bean```이라 적힌 메서드 모두 호출하여 반환된 객체를 스프링 컨테이너에 등록하고 이렇게 등록된 객체들을 스프링 빈이라고 한다.
* ```@Bean```이 붙은 메서드 명 = 스프링 빈의 이름(따로 지정도 할 수는 있으나 보통은 메서드 명과 통일해서 사용)
* **빈 이름은 항상 다른 이름을 부여**해야 한다. 같은 이름 부여 시, 다른 빈이 무시되거나 기존 빈을 덮어버리는 등 오류가 발생


#### 스프링 빈 조회
***
**스프링 빈 조회 시 유의 사항**
동일한 타입이 둘 이상일 경우, 빈 이름을 지정하거나 모두 조회하고 싶을 때는 특정 타입 모두 조회하는 방식으로 해야 한다.
```
@Test
@DisplayName("빈 이름을 지정")
void findBeanByName() {
    MemberRepository memberRepository = ac.getBean("memberRepository1", MemberRepository.class);
    assertThat(memberRepository).isInstanceOf(MemberRepository.class);
}
@Test
@DisplayName("특정 타입을 모두 조회")
void findAllBeanByType() {
    Map<String, MemberRepository> beansOfType = ac.getBeansOfType(MemberRepository.class);
    for (String key : beansOfType.keySet()) {
        System.out.println("key = " + key + " value = " + beansOfType.get(key));
    }
    System.out.println("beansOfType = " + beansOfType);
    assertThat(beansOfType.size()).isEqualTo(2);
}
```


### 스프링 빈 조회 - 상속 관계
***
* 부모 타입으로 조회 시, 자식도 함께 조회됨
* 따라서 Object 타입으로 조회 시, 모든 스프링 빈이 조회됨
