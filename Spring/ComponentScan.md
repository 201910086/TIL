[스프링 핵심 원리 - 기본편](https://www.inflearn.com/course/%EC%8A%A4%ED%94%84%EB%A7%81-%ED%95%B5%EC%8B%AC-%EC%9B%90%EB%A6%AC-%EA%B8%B0%EB%B3%B8%ED%8E%B8)을 수강하며 정리한 내용이다.


### 컴포넌트 스캔
***
* ```@ComponentScan```을 설정 정보에 붙여주기
* 기존의 AppConfig와 달리 ```@Bean```으로 등록한 클래스가 없음
* ```@ComponentScan``` : 이름 그대로 ```@Component``` 애노테이션이 붙은 클래스를 스캔 후, 스프링 빈으로 등록(@Configuration은 소스코드에 @Component애노테이션이 붙어있어 컴포넌트 스캔의 대상이 됨)


### 탐색 위치와 기본 스캔 대상
***
#### 탐색할 패키지 시작 위치 지정
```
@ComponentScan{
    basePackages = "hello.core",
}
```
* ```basePacages```: 탐색할 패키지의 시작 위치를 지정
* 권장 방법 : 패키지 위치를 따로 지정하지 않고, 설정 정보 클래스의 위치를 프로젝트 최상단에 두기

#### 컴포넌트 스캔 기본 대상
* ```@Component```: 컴포넌트 스캔에서 사용
* ```@Controller```: 스프링 MVC 컨트롤러에서 사용
* ```@Service```: 스프링 비즈니스 로직에서 사용
* ```@Repository```: 스프링 데이터 접근 계층에서 사용
* ```@Configuration```: 스프링 설정 정보에서 사용


### 중복 등록과 충돌
1. 자동 빈 등록 vs 자동 빈 등록
2. 수동 빈 등록 vs 자동 빈 등록
***
* 자동 빈 등록 vs 자동 빈 등록
: ```ConflictingBeanDefinitionExceptoin```예외 발생

* 수동 빈 등록 vs 자동 빈 등록
: 수동 빈 등록이 우선권을 가짐(수동 빈이 자동 빈을 오버라이딩해버림)
