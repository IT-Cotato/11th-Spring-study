[[자바 ORM 표준 JPA 프로그래밍 - 기본편] 섹션 10.값 타입 / 11.객체지향 쿼리 언어1 - 기본 문법](https://velog.io/@sehyun56/%EC%9E%90%EB%B0%94-ORM-%ED%91%9C%EC%A4%80-JPA-%ED%94%84%EB%A1%9C%EA%B7%B8%EB%9E%98%EB%B0%8D-%EA%B8%B0%EB%B3%B8%ED%8E%B8-%EC%84%B9%EC%85%98-10.%EA%B0%92-%ED%83%80%EC%9E%85-11.%EA%B0%9D%EC%B2%B4%EC%A7%80%ED%96%A5-%EC%BF%BC%EB%A6%AC-%EC%96%B8%EC%96%B41-%EA%B8%B0%EB%B3%B8-%EB%AC%B8%EB%B2%95)

### 💡 다양한 쿼리 방법

- JPA는 다양한 쿼리 방법을 지원함
  - JPQL, JPA Criteria, QueryDSL, 네이티브 SQL, JDBC API 직접 사용

---

### 💡 JPQL

- JPA가 제공하는 SQL을 추상한 객체 지향 쿼리 언어
- SQL과 문법 유사
- 엔티티 객체를 대상으로 쿼리(SQL은 데이터베이스 테이블을 대상으로 쿼리)
- SQL을 추상화해서 특정 데이터베이스 SQL에 의존하지 않음
- 동적 쿼리를 만들기 어려움
- JPQL은 SQL로 변환됨

```java
List<Member> result = em.createQuery(
        "select m From Member m where m.username like '%kim%'", // Member 엔티티를 조회하는 JPQL
        Member.class
).getResultList();
```

---

### 💡 JPA Criteria

- 자바 표준에서 제공
- 동적쿼리 짜기 좋음
- 컴파일 오류 잡아줌
- 자바 코드로 JPQL 작성 가능
- 너무 복잡하고 실용성이 없기 때문에 QueryDSL 사용 권장

```java
// Criteria 사용 준비
CriteriaBuilder cb = em.getCriteriaBuilder();
CriteriaQuery<Member> query = cb.createQuery(Member.class);

Root<Member> m = query.from(Member.class); // 조회를 시작할 클래스
CriteriaQuery<Member> cq = query.select(m).where(cb.equal(m.get("username"), "kim")); // 쿼리 생성
List<Member> resultList = em.createQuery(cq)
        .getResultList();
```

---

### 💡 QueryDSL

- 오픈소스로 제공됨
- 자바 코드로 JPQL 작성 가능
- 컴파일 시점에 문법 오류 찾을 수 있음
- 동적쿼리 작성 편리
- 단순하고 쉬워서 실무 사용 권장

---

### 💡 네이티브 SQL

- JPA가 제공하는 SQL을 직접 사용할 수 있음
- JPQL로 해결할 수 없는 특정 데이터베이스에 의존적인 기능

```java
em.createNativeQuery("select MEMBER_ID, city, street, zipcode, USERNAME from MEMBER", Member.class)
                    .getResultList();
```

---

### 💡 JDBC 직접 사용

- JPA를 사용하면서 JDBC 커넥션을 직접 사용하거나, 스프링 JdbcTemplate, Mybatis 등을 함께 사용 가능
- 수동 플러시가 필요함(em.flush())

---

### 💡 JPQL 문법

- 엔티티와 속성은 대소문자 구분함
- JPQL 키워드는 대소문자 구분하지 않음
- 엔티티 이름 사용(테이블 이름 X)
- 별칭은 필수
- TypeQuery, Query
  - TypeQuery : 반환 타입 명확할 때 사용
  - Query : 반환 타입 명확하지 않을 때 사용

    ```java
    TypedQuery<Member> query1 = em.createQuery("select m from Member m", Member.class);
    TypedQuery<String> query2 = em.createQuery("select m.username from Member m", String.class);
    Query query3 = em.createQuery("select m.username, m.age from Member m");
    ```

- 결과 조회 API
  - getResultList() : 결과가 하나 이상일 때 리스트 반환, NullPointException 걱정 안 해도 됨
  - getSingleResult() : 결과가 하나일 때만, 결과가 없거나 둘 이상이면 예외 발생

    ```java
    // getResultList()
    List<Member> resultList = query1.getResultList();
    for (Member m : resultList) {
        System.out.println("m = " + m);
    }
    
    // getSingleResult()
    TypedQuery<Member> query4 = em.createQuery("select m from Member m where m.id = 10", Member.class);
    Member result = query4.getSingleResult();
    // Spring Data JPA -> 예외 반환 안 하게끔 코드 짜여져 있음
    ```

- 파라미터 바인딩
  - 기준 : 이름 기준, 위치 기준
  - 위치 기준은 안 쓰는 게 좋음

    ```java
    // 이름 기준
    // 체이닝으로 많이 씀
    Member singleResult = em.createQuery("select m from Member m where m.username = :username", Member.class)
            .setParameter("username", "member1")
            .getSingleResult();
    System.out.println("singleResult = " + singleResult.getUsername());
    ```


---

### 💡 프로젝션

- SELECT 절에 조회할 대상을 지정하는 것
- 프로젝션 대상 : 엔티티, 임베디드 타입, 스칼라 타입(숫자, 문자 등 기본 데이터 타입)
- 여러 값 조회
  - Query 타입으로 조회, Object[] 타입으로 조회, new 명령어로 조회(DTO 사용, 패키지 명을 포함한 전체 클래스명 입력)

    ```java
    // Object[] 타입으로 조회
    List resultList = em.createQuery("select m.username, m.age from Member m")
            .getResultList();
    Object o = resultList.get(0);
    Object[] result = (Object[]) o;
    System.out.println("username = " + result[0]);
    System.out.println("age = " + result[1]);
    ```


---

### 💡 페이징 API

- setFirstResult(int startPosition) : 조회 시작 위치(0부터 시작)
- setMaxResults(int maxResult) : 조회할 데이터 수

```java
List<Member> result = em.createQuery("select m from Member m order by m.age desc", Member.class)
          .setFirstResult(1)
          .setMaxResults(10)
          .getResultList();
```

---

### 💡 조인

- 내부 조인 : inner join
- 외부 조인 : outer join (ex. 회원은 있고, 팀은 없어도 조인 가능)
- 세타 조인 : cross join (곱하기로 다 불러져서 연관관계가 전혀 없는 것끼리 비교하고 싶을 때 사용)
- on 절 : 조인 대상 필터링, 연관관계 없는 엔티티 외부 조인

```java
String query = "select m from Member m inner join m.team t"; // inner 생략 가능
String query = "select m from Member m left outer join m.team t"; // outer 생략 가능
String query = "select m from Member m, Team t where m.username = t.name"; // cross join
String query = "select m from Member m left join m.team t on t.name = 'teamA'"; // 조인 대상 필터링, on 절 사용
String query = "select m from Member m left join Team t on m.username = t.name"; // 비연관관계 외부 조인, on 절 사용
```

---

### 💡 서브 쿼리

- 서브 쿼리 지원 함수
  - [NOT] EXISTS {ALL | ANY | SOME} (subquery) : 서브쿼리에 결과가 존재하면 참
    - ALL : 모두 만족하면 참
    - ANY, SOME : 조건을 하나라도 만족하면 참
  - [NOT] IN (subquery) : 서브쿼리의 결과 중 하나라도 같은 것이 있으면 참
- JPA 서브 쿼리 한계
  - JPA는 WHERE, HAVING 절에서만 서브 쿼리 사용 가능
  - SELECT 절 가능(하이버네이트에서 지원)
  - FROM 절의 서브쿼리는 현재 JPQL에서 불가능 → 조인으로 풀 수 있으면 풀어서 해결
    - 하이버네이트6부터는 FROM 절의 서브쿼리를 지원함

---

### 💡 JPQL 문법 및 함수 활용 정리

- JPQL 기타
  - SQL과 문법이 같은 식
  - EXISTS, IN
  - AND, OR, NOT
  - =, >, >=, <, <=, <>
  - BETWEEN, LIKE, IS NULL
- 조건식 - CASE 식
  - COALESCE : 하나씩 조회해서 null이 아니면 반환
  - NULLIF : 두 값이 같으면 null 반환, 다르면 첫번째 값 반환

    ```java
    // 기본 CASE 식
    String query = "select " +
            "case when m.age <= 10 then '학생요금' " +
            "     when m.age >= 60 then '경로요금' " +
            "     else '일반요금' " +
            "end " +
            "from Member m";
    // COALESCE
    String query = "select coalesce(m.username, '이름 없는 회원') from Member m";
    // NULLIF
    String query = "select NULLIF(m.username, '관리자') from Member m";
    ```

- JPQL 기본 함수
  - CONCAT, SUBSTRING, TRIM, LOCATE, SIZE, INDEX 등
- 사용자 정의 함수 호출
  - 하이버네이트는 사용전 방언에 추가해야함
  - 사용하는 DB 방언을 상속받고, 사용자 정의 함수 등록

---

### 💡 새롭게 알게 된 점 및 중요한 점 정리

1. JPQL : JPA가 제공하는 SQL을 추상한 객체 지향 쿼리 언어, 엔티티 객체를 대상으로 쿼리, 특정 데이터베이스 SQL에 의존하지 않음
2. QueryDSL : 오픈소스로 제공, 자바 코드로 작성, 컴파일 시점에 문법 오류 찾을 수 있음, 동적쿼리 작성 편리
3. 반환 타입 명확할 때 TypeQuery, 반환 타입 명확하지 않을 때 Query 사용
4. setFirstResult와 setMaxResults 메서드를 사용하여 페이징 처리함