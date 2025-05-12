**Member 엔티티**

```java
@Entity
@Getter
@Setter
public class Member {

	@Id
	@GeneratedValue
	private Long id;
	private String username;
}
```

**Member 리포지토리**

```java
@Repository
public class MemberRepository {

	@PersistenceContext
	private EntityManager em;

	public Long save(Member member) {
		em.persist(member);
		return member.getId();
	}

	public Member find(Long id) {
		return em.find(Member.class, id);
	}
}
```

- @Repository
    - Component Scan의 대상이 되는 어노테이션 (내부에 @Component를 가지고 있음)
- EntityManager
    - 스프링 부트가 @PersistenceContext 어노테이션을 읽어서 엔티티 매니저 주입해줌(알아서 생성되어있음 가져다 쓰기만 하면 됨)


>💡
> **영속성 컨텍스트(Persistence Context)란?**
>
> - JPA가 엔티티 객체를 관리하는 공간(캐시)
> - DB 대신 엔티티를 임시로 보관하는 1차 캐시
> - EntityManager가 내부적으로 이 영속성 컨텍스트를 갖고 있다
> - 이 컨텍스트에 들어간 객체는 “영속 상태”라고 부른다


> 💡
> **em.persist(member)**
>
> JPA의 EntityManager를 사용해서 엔티티를 영속화하는 방식
>
> 1. 해당 엔티티(member)를 JPA의 영속성 컨텍스트에 저장한다
> 2. 트랜잭션이 커밋될 때 실제 데이터베이스에 INSERT SQL이 실행된다
> 3. 엔티티가 영속 상태가 되면서 JPA의 다양한 기능(변경 감지, 지연 로딩 등)을 사용할 수 있게 된다

Spring Data JPA로만 개발을 해왔기 때문에 순수 JPA 방식이 낯설었다

JpaRepository의 메서드들은 내부적으로 EntityManager의 메서드를 호출하도록 구현되어 있고, 나는 그동안 JpaRepository의 추상화된 메서드를 사용했던 것

---

### **Member 리포지토리 테스트**

```java
@SpringBootTest
public class MemberRepositoryTest {

	@Autowired MemberRepository memberRepository;

	@Test
	@Transactional
	@Rollback(false)
	public void testMember() throws Exception {
		// given
		Member member = new Member();
		member.setUsername("memberA");

		// when
		Long savedId = memberRepository.save(member);
		Member findMember = memberRepository.find(savedId);

		// then
		assertThat(findMember.getId()).isEqualTo(member.getId());
		assertThat(findMember.getUsername()).isEqualTo(member.getUsername());
		assertThat(findMember).isEqualTo(member); // findMember == member
	}
}
```

**트랜잭션 관련 주의사항**

- `@Transactional`을 붙이지 않으면 다음과 같은 에러가 발생한다

  > No EntityManager with actual transaction available for current thread - cannot reliably process 'persist' call

- EntityManager를 통한 모든 데이터 변경은 트랜잭션 안에서 이루어져야 하기 때문이다 (기본편)
- 테스트에서는 기본적으로 트랜잭션 종료 후 자동 롤백
    - 실제 DB 반영을 원할 경우 `@Rollback(false)` 추가하면 된다

---

### **JPA의 동일성 비교**

`assertThat(findMember).isEqualTo(member); // findMember == member`

- 같은 트랜잭션 안에서 저장하고 조회하면 영속성 컨텍스트가 같다
- 같은 영속성 컨텍스트에서는 ID가 같으면 동일한 엔티티
- 이미 영속성 컨텍스트에 존재하면 SELECT 쿼리 없이 1차 캐시에서 반환된다

> 💡
> **같은 영속성 컨텍스트란?**
> 
> - 같은 EntityManager 인스턴스를 공유하고 있다는 의미
> - 같은 트랜잭션 내에서 같은 EntityManager를 사용 중이라는 의미
> 
> 예를 들어, 같은 테스트 메서드에서 find()를 두 번 호출하면
> 
> ```java
> Member m1 = em.find(Member.class, 1L);
> Member m2 = em.find(Member.class, 1L);
> System.out.println(m1 == m2); // true
> ```
> 
> → m2는 DB에 쿼리를 날리는 게 아니라 1차 캐시에서 m1을 꺼내온 것

---

### 쿼리 파라미터 로그 남기기 (P6Spy)

```java
// P6Spy 의존성 추가
implementation 'com.github.gavlyukovskiy:p6spy-spring-boot-starter:1.10.0'
```

**로그 출력 예시**

```java
    insert 
    into
        member
        (username, id) 
    values
        (?, ?)
2025-05-12T10:30:22.878+09:00  INFO 35905 --- [Test worker] p6spy                                   
: #1747013422878 | took 0ms | statement | connection 4| url jdbc:h2:tcp://localhost/~/jpashop
insert into member (username,id) values (?,?)
insert into member (username,id) values ('memberA',1);
```

- 실제 바인딩된 파라미터 값을 포함한 SQL 문이 로그에 출력되어 디버깅에 유리하다