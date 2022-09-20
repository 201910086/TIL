[실전 Querydsl](https://www.inflearn.com/course/Querydsl-%EC%8B%A4%EC%A0%84)을 수강하며 간단히 정리한 내용이다.

### 스프링 데이터 JPA 리포지토리
***
```
public interface MemberRepository extends JpaRepository<Member, Long> {
    List<Member> findByUsername(String username);
    // 나머지 기본제공을 제외하고 username으로 찾는 것만 메소드 통해 생성을 이용
}
```

### 스프링 데이터 JPA 테스트
***
```
@SpringBootTest
@Transactional
class MemberRepositoryTest {

    @Autowired EntityManager em;
    @Autowired MemberRepository memberRepository;
    
    @Test
    public void basicTest() {
        Member member = new Member("member1", 10);
        memberRepository.save(member);
        
        Member findMember = memberRepository.findById(member.getId()).get();
        assertThat(findMember).isEqualTo(member);
        
        List<Member> result1 = memberRepository.findAll();
        assertThat(result1).containsExactly(member);
        
        List<Member> result2 = memberRepository.findByUsername("member1");
        assertThat(result2).containsExactly(member);
    }`
}
```
- Querydsl 전용 기능인 회원 search를 작성할 수 없음 -> **사용자 정의 리포지토리 필요**

### 사용자 정의 리포지토리
***
#### 사용자 정의 인터페이스
```
public interface MemberRepositoryCustom {
    List<MemberTeamDto> search(MemberSearchCondition condition);
}
```

#### 사용자 정의 인터페이스 구현체
```
public class MemberRepositoryImpl implements MemberRepositoryCustom {
    private final JPAQueryFactory queryFactory;
    
    public MemberRepositoryImpl(EntityManager em) {
        this.queryFactory = new JPAQueryFactory(em);
    }
    
    @Override
    //회원명, 팀명, 나이(ageGoe, ageLoe)
    public List<MemberTeamDto> search(MemberSearchCondition condition) {
        return queryFactory
            .select(new QMemberTeamDto(
                member.id,
                member.username,
                member.age,
                team.id,
                team.name))
            .from(member)
            .leftJoin(member.team, team)
            .where(usernameEq(condition.getUsername()),
                teamNameEq(condition.getTeamName()),
                ageGoe(condition.getAgeGoe()),
                ageLoe(condition.getAgeLoe()))
            .fetch();
        }
    
    private BooleanExpression usernameEq(String username) {
        return isEmpty(username) ? null : member.username.eq(username);
    }
    
    private BooleanExpression teamNameEq(String teamName) {
        return isEmpty(teamName) ? null : team.name.eq(teamName);
    }
    
    private BooleanExpression ageGoe(Integer ageGoe) {
        return ageGoe == null ? null : member.age.goe(ageGoe);
    }
    
    private BooleanExpression ageLoe(Integer ageLoe) {
        return ageLoe == null ? null : member.age.loe(ageLoe);
    }
}
```

#### 스프링 데이터 리포지토리에 사용자 정의 인터페이스 상속
```
public interface MemberRepository extends JpaRepository<Member, Long>, MemberRepositoryCustom {
    List<Member> findByUsername(String username);
}
```

#### 커스텀 리포지토리 동작 테스트 추가
```
@Test
public void searchTest() {
    ...
    
    MemberSearchCondition condition = new MemberSearchCondition();
    condition.setAgeGoe(35);
    condition.setAgeLoe(40);
    condition.setTeamName("teamB");
    
    List<MemberTeamDto> result = memberRepository.search(condition);
    
    assertThat(result).extracting("username").containsExactly("member4");
}
```

### 스프링 데이터 페이징 활용1 - Querydsl 페이징 연동
***
#### 사용자 정의 인터페이스에 페이징 2가지 추가
```
public interface MemberRepositoryCustom {
    List<MemberTeamDto> search(MemberSearchCondition condition);
    
    
    Page<MemberTeamDto> searchPageSimple(MemberSearchCondition condition, Pageable pageable);
    Page<MemberTeamDto> searchPageComplex(MemberSearchCondition condition, Pageable pageable);
}
```

#### 데이터 내용과 전체 카운트를 별도로 조회
* 데이터 조회 쿼리와 전체 카운트 쿼리 분리
```
@Override
public Page<MemberTeamDto> searchPageComplex(MemberSearchCondition condition, Pageable pageable) {
    List<MemberTeamDto> content = queryFactory
        .select(new QMemberTeamDto(
            member.id,
            member.username,
            member.age,
            team.id,
            team.name))
        .from(member)
        .leftJoin(member.team, team)
        .where(usernameEq(condition.getUsername()),
            teamNameEq(condition.getTeamName()),
            ageGoe(condition.getAgeGoe()),
            ageLoe(condition.getAgeLoe()))
        .offset(pageable.getOffset())
        .limit(pageable.getPageSize())
        .fetch();
    
    JPAQuery<Long> countQuery = queryFactory
        .select(member)
        .from(member)
        .leftJoin(member.team, team)
        .where(
            usernameEq(condition.getUsername()),
            teamNameEq(condition.getTeamName()),
            ageGoe(condition.getAgeGoe()),
            ageLoe(condition.getAgeLoe())
        );
        // .fetchCount(); -> Querydsl 5.0 부터는 지원x

    // return new PageImpl<>(content, pageable, total);
    return PageableExecutionUtils.getPage(content, pageable, countQuery::fetchOne); // 스프링 데이터 페이징 활용2 - CountQuery 최적화
}
```