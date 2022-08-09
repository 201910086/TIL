[스프링 MVC - 1편](https://www.inflearn.com/course/%EC%8A%A4%ED%94%84%EB%A7%81-mvc-1)을 수강하며 정리한 내용이다.


## 스프링 MVC


### 스프링 MVC 전체 구조
![image](https://user-images.githubusercontent.com/70851874/183660391-5cf3700a-b854-4cda-842e-3e97c9f6115c.png)

**동작 순서**
- 핸들러 조회: 핸들러 매핑을 통해 요청 URL에 매핑된 핸들러(컨트롤러)를 조회
- 핸들러 어댑터 조회: 핸들러를 실행할 수 있는 핸들러 어댑터 조회
- 핸들러 어댑터 실행
- 핸들러 실행
- ModelAndView 반환: 핸들러 어댑터는 핸들러가 반환하는 정보를 ModelAndView로 변환해서 반환
- viewResolver 호출: 뷰 리졸버를 찾고 실행
- View 반환: 뷰 리졸버는 뷰 논리 이름을 물리 이름으로 바꾸고 렌더링 역할을 담당하는 뷰 객체를 반환
- 뷰 렌더링

**스프링 MVC의 큰 강점**은 ```DispatcherServlet``` 코드의 변경 없이 원하는 기능을 변경하거나 확장할 수 있다는 것이다.

#### @RequestMapping
```
@Controller
@RequestMapping("/springmvc/v3/members")
public class SpringMemberControllerV3 {
    private MemberRepository memberRepository = MemberRepository.getInstance();

    @GetMapping("/new-form")
    public String newForm() {
        return "new-form";
    }

    @PostMapping("/save")
    public String save(
        @RequestParam("username") String username,
        @RequestParam("age") int age,
        Model model) {

        Member member = new Member(username, age);
        memberRepository.save(member);

        model.addAttribute("member", member);
        return "save-result";
    }

    @GetMapping
    public String members(Model model) {
        List<Member> members = memberRepository.findAll();
        model.addAttribute("members", members);
        return "members";
    }
}
```

```@RequestMapping(value = "/new-form", method = RequestMethod.GET)```
=
```@GetMapping("/new-form")```
