[스프링 핵심 원리 - 기본편](https://www.inflearn.com/course/%EC%8A%A4%ED%94%84%EB%A7%81-%ED%95%B5%EC%8B%AC-%EC%9B%90%EB%A6%AC-%EA%B8%B0%EB%B3%B8%ED%8E%B8)을 수강하며 정리한 내용이다.


### 빈 스코프

스코프: 번역 그대로 빈이 존재할 수 있는 범위
스프링 빈은 기본적으로 싱글톤 스코프로 생성됨

스프링이 지원하는 다양한 스코프 종류
* 싱글톤: 기본 스코프. 스프링 컨테이너의 시작과 종료까지 유지되는 가장 넓은 범위의 스코프
* 프로토타입: 스프링 컨테이너는 프로토타입 빈의 생성과 의존관계 주입까지만 관여하고 더는 관리하지 않는 매우 짧은 범위의 스코프
* 웹 관련 스코프
    - request: 웹 요청이 들어오고 나갈 때까지 유지되는 스코프
    - session: 웹 세션이 생성되고 종료될 때까지 유지되는 스코프
    - application: 웹의 서블릿 컨텍스트와 같은 범위로 유지되는 스코프


### 프로토 타입 스코프
싱글톤 스코프는 항상 같은 인스턴스의 스프링 빈을 반환,
반면 프로토타입 스코프는 스프링 컨테이너에 조회하면 스프링 컨테이너는 항상 새로운 인스턴스를 생성해서 반환.

* 핵심은 **스프링 컨테이너는 프로토타입 빈의 생성, 의존관계 주입, 초기화까지만 처리**한다는 것.
* 클라이언트에 빈을 반환한 후, 프로토타입 빈을 관리하지 않는다. 따라서 ```@PreDestroy```같은 종료 메서드가 호출되지 않는다.

* 스프링 컨테이너에 요청할 때마다 새로 생성
* 스프링 컨테이너는 프로토타입 빈의 생성, 의존관계 주입 그리고 초기화까지만 관여
* 종료 메서드 호출 x
* 프로토타입 빈은 프로토타입 빈을 조회한 클라이언트가 관리해야 함. 종료 메서드에 대한 호출도 클라이언트가 직접 해야함.


### 프로토타입 스코프 - 싱글톤 빈과 함께 사용시 문제점
싱글톤 빈이 의존관계 주입을 통해 프로토타입 빈을 주입받아서 사용하는 경우 ->
**싱글톤 빈이 내부에 가지고 있는 프로토타입 빈은 이미 과거에 주입이 끝난 빝으로, 주입 시점에 스프링 컨테이너에 요청해서 프로토타입 빈이 새로 생성된 것이지, 사용할 때마다 새로 생성되는 것이 아님**

굳이 해결하고자 한다면 ```Provider```로 해결이 가능하다.
**ObjectFactory, ObjectProvider** 스프링 제공
```
@Autowired
private ObjectProvider<PrototypeBean> prototypeBeanProvider;

public int logic() {
    PrototypeBean prototypeBean = prototypeBeanProvider.getObject();
    prototypeBean.addCount();
    int count = prototypeBean.getCount();
    return count;
}
```


**JSR-330 Provider** 자바 표준
```
//implementation 'javax.inject:javax.inject:1' gradle 추가 필수
@Autowired
private Provider<PrototypeBean> provider;

public int logic() {
    PrototypeBean prototypeBean = provider.get();
    prototypeBean.addCount();
    int count = prototypeBean.getCount();
    return count;
}
```

* 프로토타입 빈은 매번 사용할 때마다 의존관계 주입이 완료된 새로운 객체가 필요하면 사용하면 된다.
* 사실 실무에서는 프로토타입 빈을 직접적으로 사용하는 일은 드물다.


### 웹 스코프

* 웹 스코프는 웹 환경에서만 동장
* 웹 스코프는 프로토타입과 다르게 스프링이 해당 스코프의 종료 시점까지 관리 -> 종료 메서드 호출

