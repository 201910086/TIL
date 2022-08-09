[스프링 MVC - 1편](https://www.inflearn.com/course/%EC%8A%A4%ED%94%84%EB%A7%81-mvc-1)을 수강하며 정리한 내용이다.


### MVC 프레임워크
(단계별로 점진적으로 추가)

#### 1. 프론트 컨트롤러 패턴
- 프론트 컨트롤러 서블릿 하나로 클라이언트 요청을 받음
- 프론트 컨트롤러가 요청에 맞는 컨트롤러 찾아서 호출해줌
- 이전에는 중복되었던 것들을 공통 처리가 가능
- 프론트 컨트롤러를 제외한 나머지 컨트롤러는 서블릿 사용하지 않아도 됨

**스프링 웹 MVC의 핵심이 바로 FrontController**
**스프링 웹 MVC의 DispatchServlet이 FrontController 패턴으로 구현되어 있음**

```
@WebServlet(name = "frontControllerServletV1", urlPatterns = "/frontcontroller/v1/*")
public class FrontControllerServletV1 extends HttpServlet {
    private Map<String, ControllerV1> controllerMap = new HashMap<>();

    public FrontControllerServletV1() {
        controllerMap.put("/front-controller/v1/members/new-form", new MemberFormControllerV1());
        controllerMap.put("/front-controller/v1/members/save", new MemberSaveControllerV1());
        controllerMap.put("/front-controller/v1/members", new MemberListControllerV1());
    }

    @Override
    protected void service(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
        System.out.println("FrontControllerServletV1.service");
        String requestURI = request.getRequestURI();

        ControllerV1 controller = controllerMap.get(requestURI);
        if (controller == null) {
            response.setStatus(HttpServletResponse.SC_NOT_FOUND);
            return;
        }

        controller.process(request, response);
    }
}
```


#### 2. View 분리
- 생략되었지만 v1 구현 시 모든 컨트롤러에서 뷰로 이동하는 부분에 중복이 있어 깔끔하지 않다.
```
String viewPath = "/WEB-INF/views/new-form.jsp";
RequestDispatcher dispatcher = request.getRequestDispatcher(viewPath);
dispatcher.forward(request, response);
```
- 이를 해결하기 위해 별도로 뷰를 처리하는 객체를 만든다.

```
public class MyView {

    private String viewPath;

    public MyView(String viewPath) {
        this.viewPath = viewPath;
    }

    public void render(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
        RequestDispatcher dispatcher = request.getRequestDispatcher(viewPath);
        dispatcher.forward(request, response);
    }
}
```
이 MyView만 보면 감이 안잡힐 수도 있으나 같이 구현되는 컨트롤러 인터페이스에서 반환을 MyView로 하게 된다.

```
public interface ControllerV2 {
    MyView process(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException;
}
```

그렇게 함으로써 각 컨트롤러는 일일이 ```dispatcher.forward()```를 직접 생성, 호출하지 않아도 되고, 단순히 MyView 객체를 생성하고 거기에 뷰 이름만 넣고 반환하기만 하면 된다.
```
return new MyView("/WEB-INF/views/members.jsp");
```


#### 3. Model 추가
요청 파라미터의 정보는 HttpServletRequest, HttpServletResponse가 아닌 자바의 Map으로 대신 넘기도록 하면 컨트롤러가 서블릿 기술을 몰라도 동작할 수 있다. 그리고 request 객체는 별도의 Model 객체를 만들어 반환하면 구현하는 컨트롤러가 서블릿 기술을 전혀 사용하지 않도록 구현할 수 있다.
-> 구현 코드가 매우 단순해지고 테스트 코드 작성이 쉽다

```
public class ModelView {
    private String viewName;
    private Map<String, Object> model = new HashMap<>();

    public ModelView(String viewName) {
        this.viewName = viewName;
    }

    public String getViewName() {
        return viewName;
    }

    public void setViewName(String viewName) {
        this.viewName = viewName;
    }

    public Map<String, Object> getModel() {
        return model;
    }

    public void setModel(Map<String, Object> model) {
        this.model = model;
    }
}
```

model은 단순히 map으로 되어 있어 컨트롤러에서 뷰에 필요한 데이터를 key, value로 넘겨주면 된다.

```
public interface ControllerV3 {
    ModelView process(Map<String, String> paramMap);
}
```

**뷰 리졸버**
컨트롤러가 반환한 논리 뷰 이름을 실제 물리 뷰 경로로 변경한다. 그리고 실제 물리 경로가 있는 MyView객체를 반환한다.


#### 4. 좀 더 단순하고 실용적인 컨트롤러
ModelView가 없이 model 객체를 파라미터로 전달하여 결과로 뷰 이름만 반환하게 한다.
```
public interface ControllerV4 {

    String process(Map<String, String> paramMap, Map<String, Object> model);
}
```
```return "new-form";```과 같이 논리 뷰 이름만 반환.


#### 5. 유연한 컨트롤러
ModelView가 있는 버전과 없는 버전 두 방식 모두로 개발하고 싶을 때는 어댑터 패턴을 이용해 프론트 컨트롤러가 다양한 방식의 컨트롤러를 처리할 수 있도록 할 수 있다.

**핸들러 어댑터**: 다양한 종류의 컨트롤러를 호출할 수 있게 하는 중간의 어댑터 역할을 하는 것
**핸들러**: 이전의 컨트롤러 -> 핸들러

```
public interface MyHandlerAdapter {
    boolean supports(Object handler);
    ModelView handle(HttpServletRequest request, HttpServletResponse response, Object handler) throws ServletException, IOException;
}
```
handler: 컨트롤러를 말함. 어댑터가 해당 컨트롤러 처리 가능한지 판단하는 메서드
어댑터는 실제 컨트롤러를 호출하고 그 결과로 ModelView를 반환


각각 ModelView가 있는 버전, 없는 버전에 해당하는 어댑터를 구현한다.
