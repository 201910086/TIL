[실전 스프링 부트와 JPA 활용 2 - API 개발과 성능 최적화](https://www.inflearn.com/course/%EC%8A%A4%ED%94%84%EB%A7%81%EB%B6%80%ED%8A%B8-JPA-API%EA%B0%9C%EB%B0%9C-%EC%84%B1%EB%8A%A5%EC%B5%9C%EC%A0%81%ED%99%94)을 수강하며 간단히 정리한 내용이다.

### 회원 등록 API
***
#### V1: 엔티티를 RequestBody에 직접 매핑
```
@RestController
@RequiredArgsConstructor
public class MemberApiController {

    private final MemberService memberService;

    @PostMapping("/api/v1/members")
    public CreateMemberResponse saveMemberV1(@RequestBody @Valid Member member){...}
```
- 엔티티에 프레젠테이션 계층을 위한 로직이 추가됨
- 엔티티에 API 검증을 위한 로직이 들어감(ex. @NotEmpty)
- 실무에서는 회원 엔티티를 위한 API가 다양하게 만들어지는데, 한 엔티티에 각각의 API를 위한 모든 요청 요구사항을 담기는 어려움(요즘에는 소셜로그인 등 다양해짐)
- 엔티티가 변경되면 API 스펙이 변경됨
- -> API 요청 스펙에 맞추어 별도의 DTO를 파라미터로 받기(엔티티를 파라미터로 받지 않기!)

#### V2: 엔티티 대신 DTO를 RequestBody에 매핑
```
@PostMapping("/api/v2/members")
public CreateMemberResponse saveMemberV2(@RequestBody @Valid CreateMemberRequest request) {
    ...
    return new CreateMemberResponse(id);
}
```
- 엔티티와 프레젠테이션 계층을 위한 로직 분리 가능
- 엔티티와 API 스펙을 명확하게 분리할 수 있음
- 엔티티가 변해도 API 스펙이 변하지 않음

### 회원 수정 API
***
```
@PostMapping("/api/v2/members/{id}")
public UpdateMemberResponse updateMemberV2(@PathVariable("id") Long id, @RequestBody @Valid UpdateMemberRequest request) {
    memberService.update(id, request.getName());
    Member findMember = memberService.findOne(id);
    return new UpdateMemberResponse(findMember.getId(), findMember.getName());
}

@Data
static class UpdateMemberRequest {
    private String name;
}

@Data
@AllArgsConstructor
static class UpdateMemberResponse {
    private Long id;
    private String name;
}
```
- 회원 수정도 DTO를 요청 파라미터에 매핑

### 회원 조회 API
***
#### V1: 응답 값으로 엔티티를 직접 외부에 노출
```
@RestController
@RequiredArgsConstructor
public class MemberApiController {

    private final MemberService memberService;
    
    @GetMapping("/api/v1/members")
    public List<Member> membersV1() {
    return memberService.findMembers();
    }
}
```
- 엔티티에 프레젠테이션 계층을 위한 로직이 추가됨
- 기본적으로 엔티티의 모든 값이 노출
- 응답 스펙을 맞추기 위해 로직이 추가됨(@JsonResponse 등)
- 한 엔티티에 각가의 API를 위한 프레젠테이션 응답 로직을 담기는 어려움
- 엔티티 변경 시 API 스펙도 같이 변함
- 컬렉션을 직접 반환하면 향후 API 스펙을 변경하기 어려움(별도의 Result 클래스 생성으로 해결해야 함)
- -> API 응답 스펙에 맞추어 별도의 DTO를 반환

#### V2: 응답 값으로 엔티티가 아닌 별도의 DTO 사용
```
@GetMapping("/api/v2/members")
public Result membersV2() {

    List<Member> findMembers = memberService.findMembers();
    //엔티티 -> DTO 변환
    List<MemberDto> collect = findMembers.stream()
        .map(m -> new MemberDto(m.getName()))
        .collect(Collectors.toList());
    return new Result(collect);
}

@Data
@AllArgsConstructor
static class Result<T> {
    private T data;
}

@Data
@AllArgsConstructor
static class MemberDto {
    private String name;
}
```
- Result클래스로 컬렉션을 감싸서 향후 필요한 필드를 추가할 수 있음