[[[실전! 스프링 부트와 JPA 활용1 - 웹 애플리케이션 개발] 섹션4. 애플리케이션 구현 준비 / 섹션5. 회원 도메인 개발]](https://velog.io/@sehyun56/%EC%8B%A4%EC%A0%84-%EC%8A%A4%ED%94%84%EB%A7%81-%EB%B6%80%ED%8A%B8%EC%99%80-JPA-%ED%99%9C%EC%9A%A91-%EC%9B%B9-%EC%95%A0%ED%94%8C%EB%A6%AC%EC%BC%80%EC%9D%B4%EC%85%98-%EA%B0%9C%EB%B0%9C-%EC%84%B9%EC%85%984.-%EC%95%A0%ED%94%8C%EB%A6%AC%EC%BC%80%EC%9D%B4%EC%85%98-%EA%B5%AC%ED%98%84-%EC%A4%80%EB%B9%84-%EC%84%B9%EC%85%985.-%ED%9A%8C%EC%9B%90-%EB%8F%84%EB%A9%94%EC%9D%B8-%EA%B0%9C%EB%B0%9C)
### 💡 회원 리포지토리 개발

- 스프링이 엔티티 매니저를 만들어서 인젝션 해줌
- @PersistentContext 와 @Autowired 를 사용하지 않고 다른 코드들과의 코드의 통일성을 위해서 @RequiredArgsConstructor를 사용하고 final 키워드를 사용해서 구현

```java
@Repository
@RequiredArgsConstructor
public class MemberRepository {

//    @PersistenceContext
//    @Autowired
    private final EntityManager em; // 스프링이 이 엔티티 매니저를 만들어서 인젝션 해줌

//    public MemberRepository(EntityManager em) {
//        this.em = em;
//    }

    public void save(Member member) {
        em.persist(member); // 영속성 컨텍스트에 넣음
    }

    public Member findOne(Long id) {
        return em.find(Member.class, id); // JPA의 find 메소드 사용
    }

    public List<Member> findAll() {
        return em.createQuery("select m from Member m", Member.class)
                .getResultList();
    }

    public List<Member> findByName(String name) {
        return em.createQuery("select m from Member m where m.name = :name", Member.class)
                .setParameter("name", name)
                .getResultList();
    }
}
```

---

### 💡  회원 서비스 개발

- @Transactional
    - 트랜잭션, 영속성 컨텍스트
    - lazy loading 가능해짐
    - 클래스에 붙은 @Transactional은 public 메소드에 다 적용됨
    - 읽는 기능만 하면 @Transactional(readOnly = true) 를 하면 성능 최적화 가능 그러나 쓰기 기능이 있으면 데이터 변경이 되지 않기 때문에 @Transactional 써야함
- @Autowired를 통해 스프링에 들어있는 리포지토리를 인젝션하는 것, setter 인젝션, 생성자 인젝션도 가능하지만 추천하는 방법은 Lombok의 기능인 @RequiredArgsConstructor를 사용하는 방법
    - @RequiredArgsConstructor 은 final 이 붙은 필드만 가지고 생성자 만들어줌

```java
@Service
@Transactional(readOnly = true)
// @AllArgsConstructor
@RequiredArgsConstructor // Lombok - final 있는 필드만 가지고 생성자 만들어줌
// @Transactional // lazy loading 가능해짐
// public 메소드는 클래스에 붙은 Transactional 어노테이션에 다 적용됨
public class MemberService {

    private final MemberRepository memberRepository; // final 넣으면 생성자 주입 안 할 걸 알 수 있음

    // 생성자 인젝션
//    private MemberRepository memberRepository;
//    @Autowired // 생략 가능
//    public MemberService(MemberRepository memberRepository) {
//        this.memberRepository = memberRepository;
//    }

    // @Autowired // 스프링에 들어있는 리포지토리를 인젝션해줌
    // private MemberRepository memberRepository;

    // setter 인젝션
//    private MemberRepository memberRepository;
//    @Autowired
//    private void setMemberRepository(MemberRepository memberRepository) {
//        this.memberRepository = memberRepository;
//    }

    // 회원 가입
    @Transactional
    public Long join(Member member) {
        validateDuplicateMember(member); // 중복 회원 검증
        memberRepository.save(member);
        return member.getId();
    }

    private void validateDuplicateMember(Member member) {
        // EXCEPTION
        List<Member> findMembers = memberRepository.findByName(member.getName());
        if(!findMembers.isEmpty()) {
            throw new IllegalStateException("이미 존재하는 회원입니다.");
        }
    }

    // 회원 전체 조회
    // @Transactional(readOnly = true) // 읽기에는 readOnly 쓰면 좋고 성능이 최적화되고 쓰기에는 readOnly 붙이면 데이터 변경이 안됨
    public List<Member> findMembers() {
        return memberRepository.findAll();
    }

    // @Transactional(readOnly = true)
    public Member findOne(Long memberId) {
        return memberRepository.findOne(memberId);
    }
    
}

```

---

### 💡 회원 기능 테스트

- Test에 @Transactional 을 붙이면 각각의 테스트를 실행할 때마다 트랜잭션을 시작하고 테스트가 끝나면 커밋을 하지 않고 롤백을 함
    - 롤백을 원치 않는 경우 @Rollback(false) 붙이면 됨

```java
@SpringBootTest
@Transactional // 이 어노테이션이 테스트에 있으면 커밋을 안 하고 롤백을 함
public class MemberServiceTest {

    @Autowired MemberService memberService;
    @Autowired MemberRepository memberRepository;
    @Autowired
    EntityManager em;

    @Test
    // @Rollback(false) // 롤백을 안 하고 커밋을 함
    public void 회원가입() throws Exception {
        // given
        Member member = new Member();
        member.setName("kim");

        // when
        Long saveId = memberService.join(member);

        // then
        em.flush();
        Assertions.assertEquals(member, memberRepository.findOne(saveId));
    }

    @Test
    public void 중복_회원_예외() throws Exception {
        // given
        Member member1 = new Member();
        member1.setName("kim");

        Member member2 = new Member();
        member2.setName("kim");
        // when
        memberService.join(member1);

        // then
        // fail("예외가 발생해야 한다.");
        assertThrows(IllegalStateException.class, () ->
                memberService.join(member2));
    }

}
```

- 테스트를 할 때 DB를 분리시키는게 좋음
    - 방법 1 ) test에 resources 파일을 만들어서 application.yml 을 *url: jdbc:h2:mem:test 이렇게 설정하기*
    - 방법 2 ) 따로 데이터베이스 설정을 안 하면 메모리 DB 사용함

---

### 💡 새롭게 알게 된 점 및 중요한 점 정리

1. @Transactional
    - 클래스에 붙은 @Transactional은 public 메소드에 다 적용됨
    - 읽기 기능에는 @Transactional(readOnly = true), 쓰기 기능에는 @Transactional
    - Test에 @Transactional 붙이면 커밋이 되지 않고 롤백이 됨 → 원치 않으면 @Transactional(false)
2. 따로 DB 설정을 하지 않으면 메모리 DB를 사용함