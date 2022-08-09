[스프링 MVC - 1편](https://www.inflearn.com/course/%EC%8A%A4%ED%94%84%EB%A7%81-mvc-1)을 수강하며 정리한 내용이다.


### 서블릿


```@WebServlet``` 서블릿 애노테이션
- name: 서블릿 이름
- urlPatterns: URL 매핑

HTTP 요청을 통해 매핑된 URL이 호출되면 서블릿 컨테이너는 다음 메서드를 실행
```
protected void service(HttpRequest request, HttpResponse response)
```

#### HttpServletRequest
**HttpServletRequest 역할**
HTTP 요청 메시지를 편리하게 사용할 수 있도록 개발자 대신 HTTP 요청 메시지를 파싱하고 그 결과를 ```HttpServletRequest``` 객체에 담아서 제공.

**HTTP 요청 메시지 예시**
```
POST /save HTTP/1.1
Host: localhost:8080
Content-Type: application/x-www-form-urlencoded

username=kim&age=20
```
* START LINE
    - HTTP 메서드, URL, 쿼리 스트링, 스키마/프로토콜
* 헤더
    - 헤더 조회
* 바디
    - form 파라미터 형식 조회
    - message body 데이터 직접 조회

* ++ 추가로 임시 저장소 기능(setAttribute 등), 세션 관리 기능(getSession) 등이 있음


#### HTTP 요청 데이터
HTTP 요청 메시지를 통해 클라이언트에서 서버로 데이터를 전달하는 방법 알아보기

1. GET - 쿼리 파라미터
    - 메시지 바디 없이 URL의 쿼리 파라미터를 사용해 데이터를 전달
    - 검색, 필터, 페이징 등에서 많이 사용
    - ```HttpServletRequest```를 통해 조회 가능
```
String username = request.getParameter("username"); //단일 파라미터 조회
Enumeration<String> parameterNames = request.getParameterNames(); //파라미터 이름들 모두 조회
Map<String, String[]> parameterMap = request.getParameterMap(); //파라미터를 Map으로 조회
String[] usernames = request.getParameterValues("username"); //복수 파라미터 조회
```

2. POST - HTMLForm
    - content-type은 ```application/x-www-form-urlencoded``` 꼭 지정해야 함
    - 메시지 바디에 쿼리 파라미터 형식으로 데이터 전달
    - GET쿼리파라미터 방식과 마찬가지로 ```HttpServletRequest```를 통해 쿼리 파라미터 조회 메서드 그대로 사용하여 조회 가능

3. HTTP message body
    - HTTP message body에 데이터 직접 담아서 요청
    - HTTP API에서 주로 사용하며 JSON, XML, TEXT
    - 위 두 가지와 달리 POST, PUT, PATCH도 가능
- (3.1.) JSON 형식으로 데이터 전달 시:
content-type: **application/json**
JSON 결과 파싱해서 사용할 수 있는 자바 객체로 변환하려면 JSON 변환 라이브러리를 추가해서 사용해야 하지만, 스프링 부트로 Spring MVC를 선택할 시, 기본으로 Jackson 라이브러리(ObjectMapper)를 제공한다.


#### HttpServletResponse
**역할**
- HTTP 응답 메시지 생성
    - HTTP 응답코드 지정, 헤더 생성, 바디 생성
- 편의기능 제공
    - content-type, 쿠키, redirect
```
response.setStatus(HttpServletResponse.SC_OK); //200
response.setHeader("Content-Type", "text/plain;charset=utf-8");
response.setHeader("Pragma", "no-cache");
response.setHeader("my-header","hello");

content(response);
cookie(response);
response.sendRedirect("/basic/hello-form.html);

PrintWriter writer = response.getWriter();
writer.println("ok);
```

