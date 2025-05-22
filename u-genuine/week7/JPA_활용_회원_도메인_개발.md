### MemberRepository

```java
@Repository
public class MemberRepository {

    @PersistenceContext
    private EntityManager em;
    
    public void save(Member member) {
        em.persist(member);
    }
    
    public Member findOne(Long id) {
        return em.find(Member.class, id);
    }
    
    public List<Member> findAll() {\
        return em.createQuery("select m from Member m", Member.class)
            .getResultList();
    }
    
    public List<Member> findByName(String name) {
        return em.createQuery("select m from Member m where m.name =:name", Member.class)
            .setParameter("name", name)
            .getResultList();
    }
}
```

**하나씩 살펴보자!**

```java
@Repository
public class MemberRepository { ... }
```

- `@Repository`: 내부에 `@Component` 가 포함되어 있어, 스프링이 자동으로 빈으로 등록한다

```java
@PersistenceContext
private EntityManager em;
```

- `@PersistenceContext`: JPA의 EntityManager를 주입받기 위한 어노테이션
    - 스프링이 이 어노테이션을 보고 EntityManager를 만들어서 주입한다

```java
public void save(Member member) {
    em.persist(member);
}
```

- `persist(member)`: 영속성 컨텍스트에 Member 엔티티를 저장
    - 트랜잭션 커밋 시점에 DB에 INSERT 쿼리가 나간다

```java
public Member findOne(Long id) {
    return em.find(Member.class, id);
}
```

- 1차 캐시 → DB 순으로 조회한다
- `find(Class<T>, PK)` : 타입은 Member.class, 기본키는 id

```java
public List<Member> findAll() {
    return em.createQuery("select m from Member m", Member.class)
        .getResultList();
}
```

- JPQL은 SQL과 달리 테이블이 아니라 엔티티(Member)를 대상으로 쿼리를 작성한다

---

### MemberService

```java
@Service
@Transactional(readOnly = true)
@RequiredArgsConstructor
public class MemberService {

    private final MemberRepository memberRepository;

    /**
     * 회원 가입
     */
    @Transactional // readOnly = false
    public Long join(Member member) {
        validateDuplicateMember(member); // 중복 회원 검증
        memberRepository.save(member);
        return member.getId();
    }

    private void validateDuplicateMember(Member member) {
        List<Member> findMembers = memberRepository.findByName(member.getName());
        if(!findMembers.isEmpty()) {
            throw new IllegalStateException("이미 존재하는 회원입니다.");
        }
    }

    // 회원 전체 조회
    public List<Member> findMembers() {
        return memberRepository.findAll();
    }

    public Member findOne(Long id) {
        return memberRepository.findOne(id);
    }
}
```

**하나씩 살펴보자!**

```java
@Service
@Transactional(readOnly = true)
@RequiredArgsConstructor
public class MemberService { ... }
```

- `@Service`: 내부에 `@Component` 가 포함되어 컴포넌트 스캔의 대상이 되며 비즈니스 로직을 담당하는 클래스라는 것을 명시한다
- `@Transactional(readOnly=true)` : 클래스 내 모든 public 메서드에 기본으로 읽기 전용 트랜잭선이 적용된다
    - JPA의 모든 데이터 변경 로직은 트랜잭션 안에서 실행되어야 한다
    - 성능 최적화 (dirth checking 생략 등)
    - 쓰기 작업에는 별도로 `@Transactional` 을 메서드에 명시해야 한다 (기본값은 readOnly = false)
- `@RequiredArgsConstructor` : final 필드만을 인자로 받는 생성자를 롬복이 자동으로 생성해준다

**회원가입 메서드**

```java
@Transactional
public Long join(Member member) {
    validateDuplicateMember(member); // 중복 회원 검증
    memberRepository.save(member);
    return member.getId();
}
	
	
private void validateDuplicateMember(Member member) {
    List<Member> findMembers = memberRepository.findByName(member.getName());
    if(!findMembers.isEmpty()) {
        throw new IllegalStateException("이미 존재하는 회원입니다.");
    }
}
```

- `@Transactional`
    - 클래스 레벨의 `readOnly=true` 를 덮어쓰고 쓰기 가능 트랜잭션으로 변경한다

- save(member)에서 em.persist(member)가 호출된다
    - 이 시점에 member 객체가 영속성 컨텍스트에 등록된다 (INSERT 쿼리는 아직 안 나감)
    - 영속성 컨텍스트는 **Map<Key, Value>** 형태
        - Key: 엔티티의 식별자(PK, 즉 id)
        - Value: 실제 엔티티 객체(member)
    - 영속성 컨텍스트에 등록됨과 동시에 JPA가 PK 값을 생성하고 객체에 넣어준다
        - @GeneratedValue(strategy = GenerationType.IDENTITY)를 통해 ID 생성 전략을 미리 지정했기 때문에
        - member.getId()는 persist() 호출 직후에도 null이 아닌 값이 채워져있다
    - 이후 트랜잭션 커밋 시점에는
        - JPA가 영속성 컨텍스트에 등록된 변경 내역을 보고, 필요한 SQL(insert / update / delete)실행
        - 여기서 INSERT 쿼리가 DB로 나감
        - 자동 flush + commit

- 멀티스레드 환경에서 동시에 같은 이름으로 회원 가입이 일어날 수 있다
    - DB 레벨에서도 name필드에 unique 제약 조건을 걸어주는 것이 안전하다

---

### **MemberSericeTest**

```java
@SpringBootTest
@Transactional
class MemberServiceTest {
	
    @Autowired
    MemberService memberService;

    @Autowired
    MemberRepository memberRepository;

    @Autowired
    EntityManager em;

    @Test
    // @Rollback(false)
    public void 회원가입() throws Exception {
        // given
        Member member = new Member();
        member.setName("kim");

        // when
        Long savedId = memberService.join(member);

        // then
        em.flush(); // INSERT 쿼리 확인용
        assertEquals(member, memberRepository.findOne(savedId));
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
        // 예외가 발생해야 테스트 통과
        assertThrows(IllegalStateException.class, () -> memberService.join(member2)); // 예외가 발생해야 한다 !!
    }
}
```

- `@Transactional(테스트용)`
    - 각 테스트 메서드 실행 후 자동으로 롤백된다
    - DB에 반영되지 않기 때문에 다른 테스트에 영향을 주지 않는다

- `@Rollback(false)`
    - 테스트 후 롤백하지 않고 커밋한다
    - DB에 반영된 결과를 볼 수 있다

- `em.flush()`
    - 영속성 컨텍스트의 변경사항을 DB에 즉시 반영 (INSERT 쿼리 확인 가능)
    - 하지만 트랜잭션은 롤백되므로, DB에는 실제로 반영되지 않는다
    - 쿼리는 보고싶고, DB에는 반영하고 싶지 않을 때 @Rollback(false) 대신 사용