JPA에서는 데이터를 변경하는 모든 작업은 반드시 트랜잭션 안에서 작업해야 한다

**데이터 저장 예제**

```java
public class JpaMain {
    public static void main(String[] args) {
        EntityManagerFactory emf = Persistence.createEntityManagerFactory("hello");

        EntityManager em = emf.createEntityManager();
			
        EntityTransaction tx = em.getTransaction(); // 트랜잭션 얻음
        tx.begin(); // 트랜잭션 시작

        Member member = new Member();
        member.setId(1L);
        member.setName("HelloA");

        em.persist(member);

        tx.commit(); // 트랜잭션 끝

        em.close();

        emf.close();
    }
}
```

**JPA의 역할**

- SQL쿼리를 직접 작성하지 않아도 JPA가 매핑 정보를 보고 SQL 생성
- `@Table(name=”USER”)` 으로 테이블명 변경 가능

### 트랜잭션 에러 처리

중간에 에러 발생 시 em.close() / emf.close()가 실행되지 않는 문제를 try-catch-finally 로 해결

```java
public class JpaMain {
    public static void main(String[] args) {
        EntityManagerFactory emf = Persistence.createEntityManagerFactory("hello");
        EntityManager em = emf.createEntityManager();
        EntityTransaction tx = em.getTransaction();
        tx.begin();

        try{
            Member member = new Member();
            member.setId(1L);
            member.setName("HelloA");
            
            em.persist(member);
            tx.commit();
        } catch (Exception e) {
            tx.rollback(); // 에러 시 롤백
        } finally {
            em.close();
        }

        emf.close();
    }
}
```

### 조회

```java
try{
	  Member member = em.find(Member.class, 1L);
	  System.out.println("findMember.id = " + member.getId());
	  System.out.println("findMember.name = " + member.getName());
	
	  tx.commit();
} catch (Exception e) {
  tx.rollback();
} finally {
	  em.close();
}
```

### 수정

```java
try{
    Member findMember = em.find(Member.class, 1L);
    findMember.setName("HelloJPA");

    // em.persist(findMember); // 안해도 됨!!
    
    tx.commit();
} catch (Exception e) {
    tx.rollback();
} finally {
    em.close();
}
```

**변경 감지**

- JPA를 통해 엔티티를 가져오면 JPA의 관리 대상이 되어서 변경 여부를 JPA가 체크한다
- 트랜잭션 커밋 시 자동으로 UPDATE 쿼리 실행
- em.persist 따로 안해도 됨

> 💡주의
> 
> 엔티티 매니저 팩토리는 하나만 생성해서 애플리케이션 전체에서 공유
> 
> 엔티티 매니저는 쓰레드 간에 공유 X (사용하고 버려야 한다)
> 
> JPA의 모든 데이터 변경은 트랜잭션 안에서 실행

### JPQL(객체 지향 쿼리)

만약 나이가 18살 이상인 회원을 모두 검색하고 싶다면 ?

```java
// 멤버 전체 조회
List<Member> result = em.createQuery("select m from Member as m", Member.class)
                .setFirstResult(5) // 5번부터
                .setMaxResults(8)  // 8개 조회
                .getResultList();
```

JPQL은 엔티티 대상으로 쿼리하며 데이터베이스 방언에 의존하지 않는다

JPQL 특징

- SQL을 추상화한 객체 지향 쿼리 언어
- SELECT, FROM, WHERE, GROUP BY, HAVING, JOIN 지원
- DB 테이블이 아닌 엔티티를 대상으로 쿼리
- 방언이 바뀌더라도 코드 변경 필요 없음

JPA 사용 시 주의사항

- EntityManagerFactory는 애플리케이션 전체에서 하나만 생성해 공유
- EntityManager는 쓰레드 간 공유 금지 → 사용 후 반드시 닫기
- 모든 데이터 변경은 트랜잭션 안에서 실행