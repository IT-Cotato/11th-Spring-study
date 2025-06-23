[[자바 ORM 표준 JPA 프로그래밍 - 기본편] 섹션2. JPA 소개 / JPA 시작하기](https://velog.io/@sehyun56/%EC%9E%90%EB%B0%94-ORM-%ED%91%9C%EC%A4%80-JPA-%ED%94%84%EB%A1%9C%EA%B7%B8%EB%9E%98%EB%B0%8D-%EA%B8%B0%EB%B3%B8%ED%8E%B8-%EC%84%B9%EC%85%982.-JPA-%EC%86%8C%EA%B0%9C-JPA-%EC%8B%9C%EC%9E%91%ED%95%98%EA%B8%B0)
### 💡 객체와 관계형 데이터베이스의 차이

1. 상속 : 데이터베이스에는 Table 슈퍼타입 서브타입 관계로 상속을 구현함
2. 연관관계 : 객체는 참조를 사용하고 테이블은 외래 키를 사용

   → 객체다운 모델링 : 참조로 연관관계를 맺음 그러나 insert 쿼리가 까다로워짐

    ```java
    class Member {
    	String id;
    	Team team; //참조로 연관관계를 맺는다.
    	String username;
    }
    ```

3. 데이터 타입
4. 데이터 식별 방법

---

### 💡 SQL 중심적인 개발의 문제점

- 개발자는 SQL 매퍼 역할을 해야하는데 객체답게 모델링 할수록 매핑 작업만 늘어남
- 객체를 자바 컬렉션에 저장 하듯이 DB에 저장하는 방법 → 해결방법으로 JPA 나옴

---

### 💡 JPA(Java Persistence API)

- 자바 진영의 ORM 기술 표준
    - ORM 이란?
        - Object-relational mapping(객체 관계 매핑)
        - 객체는 객체대로 설계하고 관계형 데이터베이스는 관계형 데이터베이스대로 설계
        - ORM 프레임워크가 중간에서 매핑해줌
- JPA는 애플리케이션과 JDBC 사이에서 동작
    - 원래 개발자가 직접 JDBC API 사용했었는데 JPA가 대신해줌
- JPA는 표준명세
    - JPA는 인터페이스의 모음
    - 보통 구현체로 하이버네이트가 많이 쓰임
- 개발자가 유지보수해야하는 영역이 줄어듦
    - 개발자가 한 줄 치면 나머지는 JPA가 처리해줌
- 동일한 트랜잭션에서 조회한 엔티티는 같음을 보장해줌

---

### 💡 JPA와 CRUD

- 저장 : jpa.persist(member)
- 조회 : Member member = jpa.find(memberId)
- 수정 : member.setName(”변경할 이름”)
- 삭제 : jpa.remove(member)

---

### 💡 JPA의 성능 최적화 지원

- 1차 캐시와 동일성 보장
    - 같은 트랜잭션 안에서는 같은 엔티티 반환 - 약간의 조회 성능 향상
- 트랜잭션을 지원하는 쓰기 지연
    - 트랜잭션을 커밋할 때까지 INSERT SQL을 모음
    - JDBC BATCH SQL 기능을 사용해서 한번에 SQL 전송

    ```java
    transaction.begin(); // 트랜잭션 시작
    em.persist(memberA);
    em.persist(memberB);
    em.persist(memberC); // 커밋하는 순간 데이터베이스에 INSERT SQL을 모아서 보냄
    transaction.commit(); // 트랜잭션 커밋
    ```

- 지연 로딩



---

### 💡 지연 로딩과 즉시 로딩

- 지연 로딩 : 객체가 실제 사용될 때 로딩
- 즉시 로딩 : JOIN SQL로 한번에 연관된 객체까지 미리 조회