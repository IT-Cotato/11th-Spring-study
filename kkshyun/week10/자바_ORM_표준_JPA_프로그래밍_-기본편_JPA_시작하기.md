[[자바 ORM 표준 JPA 프로그래밍 - 기본편] 섹션2. JPA 소개 / JPA 시작하기](https://velog.io/@sehyun56/%EC%9E%90%EB%B0%94-ORM-%ED%91%9C%EC%A4%80-JPA-%ED%94%84%EB%A1%9C%EA%B7%B8%EB%9E%98%EB%B0%8D-%EA%B8%B0%EB%B3%B8%ED%8E%B8-%EC%84%B9%EC%85%982.-JPA-%EC%86%8C%EA%B0%9C-JPA-%EC%8B%9C%EC%9E%91%ED%95%98%EA%B8%B0)
### 💡 프로젝트 생성

- H2 데이터베이스 설정
    - h2 > bin 에서 터미널 열기
    - 터미널에서 ./h2.sh → 웹 브라우저 열림
    - JDBC URL 에 jdbc:h2:~/test 입력 후 연결 → 빨간색 연결 끊기 눌러 연결 끊기
    - 그 후 부터는 JDBC URL 에 jdbc:h2:tcp://localhost/~/test 로 연결
- maven 사용
    - 자바 라이브러리, 빌드 관리
    - 라이브러리 자동 다운로드 및 의존성 관리

---

### 💡 애플리케이션 개발

- JPA 구동방식
    - Persistence 클래스에서 시작해서 설정 정보(persistence.xml)를 읽어서 EntityManagerFactory 클래스 만들어서 필요할 때마다 EntityManager 생성
- EntityManagerFactory, EntityManager 생성, 종료

    ```java
    EntityManagerFactory emf = Persistence.createEntityManagerFactory("hello");
    EntityManager em = emf.createEntityManager();
    // 코드
    em.close();
    emf.close();
    ```


---

### 💡 트랜잭션 안에서 작업하기

- JPA는 트랜잭션이 있어야만 데이터베이스와의 작업이 정상적으로 반영

    ```java
    EntityTransaction tx = em.getTransaction();
    tx.begin();
    
    Member member = new Member();
    member.setId(1L);
    member.setName("HelloA");
    em.persist(member);
    
    tx.commit();
    ```

    - try-catch 문 안에 넣는게 좋음

    ```java
    try {
        Member member = new Member();
        member.setId(2L);
        member.setName("HelloB");
        em.persist(member);
        tx.commit();
    } catch (Exception e) {
        tx.rollback();
    } finally {
        em.close();
    }
    ```

- 수정할 때

    ```java
    try {
        Member findMember = em.find(Member.class, 1L); // 조회
        findMember.setName("HelloJPA");
        tx.commit();
    } catch (Exception e) {
        tx.rollback();
    } finally {
        em.close();
    }
    ```


---

### 💡 주의할 점

- 엔티티 매니저 팩토리는 하나만 생성해서 애플리케이션 전체에서 공유
- 엔티티 매니저는 쓰레드 간에 공유하면 안됨
- JPA의 모든 데이터 변경은 트랜잭션 안에서 실행해야함

---

### 💡 JPQL

- JPA를 사용하면 엔티티 객체를 중심으로 개발
- 검색을 할 때도 테이블이 아닌 엔티티 객체를 대상으로 개발
- 엔티티 객체를 대상으로 쿼리를 할 수 있는 JPQL

    ```java
    List<Member> result = em.createQuery("select m from Member as m", Member.class)
                        .setFirstResult(5)
                       .setMaxResults(8)
                        .getResultList();
    ```

- JPA는 SQL을 추상화한 JPQL이라는 객체 지향 쿼리 언어 제공 → 특정 데이터베이스 SQL에 의존하지 않음
- JPQL은 엔티티 객체를 대상으로 쿼리
- SQL은 데이터베이스 테이블을 대상으로 쿼리

---

### 💡 새롭게 알게 된 점 및 중요한 점 정리

1. JPA는 항상 Entity Manager Factory를 만들어야 됨 & DB 작업을 해야하면 Entity Manager Factory로 Entity Manager를 만들어서 작업을 해야 함
2. JPA의 모든 데이터 변경은 트랜잭션 안에서 이루어져야 함 & 커밋을 해야 데이터베이스에 반영이 됨
3. 용어 정리
    - ORM : 자바의 객체와 데이터베이스를 연결
    - JPA : 자바 진영의 ORM 기술 표준, 자바에서 관계형 데이터베이스를 사용하는 방식을 정의한 인터페이스
    - 하이버네이트 : JPA 인터페이스를 구현한 구현체, 자바용 ORM 프레임워크, 내부적으로 JDBC API 사용
    - Spring Data JPA : JPA + Spring을 편하게 쓰기 위한 프레임워크, 리포지토리 인터페이스만 작성하면 자동으로 구현 클래스 생성
    - JPQL : 쿼리 관련 도구 중 하나로 JPA가 제공하는 객체 지향 쿼리 언어, 엔티티 객체 기반으로 쿼리 작성
    - Querydsl : 쿼리 관련 도구 중 하나로 자바 코드로 쿼리 작성

   → 단순한 기능은 스프링 데이터 JPA로 정리하고 복잡한 쿼리를 작성해야 할 경우, JPQL(정적)이나 QueryDSL(동적) 중에 고민하면 됨. 동적 쿼리라면 QueryDSL 사용하면 되는데 스프링 데이터 JPA와 함께 사용하려면 커스텀 리포지토리를 작성해야하므로 귀찮을 수 있음