[[자바 ORM 표준 JPA 프로그래밍 - 기본편] 섹션 10.값 타입 / 11.객체지향 쿼리 언어2 - 중급 문법](https://velog.io/@sehyun56/%EC%9E%90%EB%B0%94-ORM-%ED%91%9C%EC%A4%80-JPA-%ED%94%84%EB%A1%9C%EA%B7%B8%EB%9E%98%EB%B0%8D-%EA%B8%B0%EB%B3%B8%ED%8E%B8-%EC%84%B9%EC%85%98-12.%EA%B0%9D%EC%B2%B4%EC%A7%80%ED%96%A5-%EC%BF%BC%EB%A6%AC-%EC%96%B8%EC%96%B42-%EC%A4%91%EA%B8%89-%EB%AC%B8%EB%B2%95)
### 💡 경로 표현식

- .(점)을 찍어 객체 그래프를 탐색하는 것
- 상태 필드, 단일 값 연관 필드, 컬렉션 값 연관 필드를 구분해서 이해하는 것이 중요
- 상태 필드
  - 단순히 값을 저장하기 위한 필드 ex) m.username
  - 경로 탐색 끝, 더 이상 탐색하지 않음
- 단일 값 연관 필드
  - @ManyToOne, @OneToOne, 대상이 엔티티 ex) m.team
  - 묵시적 내부 조인(inner join) 발생, 탐색함
- 컬렉션 값 연관 경로
  - @OneToMany, @ManyToMany, 대상이 컬렉션 ex)m.orders
  - 묵시적 내부 조인 발생, 탐색하지 않음
  - ex) select t.members.username from Team t -> 실패
  - 쓸 수 있는게 .size 정도 ex) String query = "select t.members.size From Team t";
  - From 절에서 명시적 조인을 통해 별칭을 얻으면 별칭을 통해 탐색 가능 ex) String query = "select m.username From Team t join t.members m";
- 최대한 묵시적 조인을 쓰지 말고, 명시적 조인(join 키워드 직접 사용)을 써야함

---

### 💡 페치 조인(fetch join)

- 실무에서 엄청 중요함
- SQL 조인 종류가 아님, JPQL에서 성능 최적화를 위해 제공하는 기능
- 연관된 엔티티나 컬렉션을 SQL 한번에 함께 조회하는 기능
- SELECT 절에 지정한 엔티티만 조회하는 일반 조인과 달리 JPQL은 결과를 반환할 때 연관관계 고려 안 함
- join fetch 명령어 사용
- 지연로딩 설정 해놔도 페치 조인이 우선
- 페치 조인을 사용할 때만 연관된 엔티티도 함께 조회(즉시 로딩)

```java
String query = "select m From Member m join fetch m.team";
List<Member> members = em.createQuery(query, Member.class)
	.getResultList();

for(Member member : result) {
		// 이때 Team은 프록시 아님
    System.out.println("member = " + member.getUsername() + ", team = " + member.getTeam().getName());
}

=>
Hibernate: 
    /* select
        m 
    From
        Member m 
    join
        
    fetch
        m.team */ 
select
    m1_0.id,
    m1_0.age,
    t1_0.id,
    t1_0.name,
    m1_0.type,
    m1_0.username 
from
    Member m1_0 
join
    Team t1_0 
        on t1_0.id=m1_0.TEAM_ID
member = 회원1, team = 팀A
member = 회원2, team = 팀A
member = 회원3, team = 팀B
```

---

### 💡 페치 조인과 DISTINCT

- 하이버네이트6부터는 DISTINCT 명령어를 사용하지 않아도 애플리케이션에서 중복 제거가 자동으로 적용됨
- DISTINCT : 중복된 결과를 제거하는 명령어
- JPQL의 DISTINCT 2가지 기능 제공
  - SQL에 DISTINCT를 추가
  - 애플리케이션에서 엔티티 중복 제거
- DISTINCT가 추가로 애플리케이션에서 중복 제거 시도
- 같은 식별자를 가진 Team 엔티티 제거

```java
String query = "select distinct t From Team t join fetch t.members";
```

---

### 💡 페치 조인의 특징과 한계

- 페치 조인 대상에는 별칭을 줄 수 없음 → 페치 조인은 따로 조회해야함(별칭은 안 쓰는게 가급적 맞음)
- 둘 이상의 컬렉션은 페치 조인 할 수 없음
- 컬렉션을 페치 조인하면 페이징 API 사용할 수 없음
  - 일대일, 다대일 같은 단일 값 연관 필드들은 페이징 가능
  - 방향을 뒤집어서 다대일로 변경하는 등 방법이 있음
  - 배치 사이즈 설정 → N+1 문제 해결 가능
- 연관된 엔티티들을 SQL 한번으로 조회하므로 성능이 최적화됨
  - N+1 문제 대부분 해결 가능

---

### 💡 다형성 쿼리

- TYPE : 조회 대상을 특정 자식으로 한정
- TREAT : 상속 구조에서 부모 타입을 특정 자식 타입으로 다룰 때 사용

---

### 💡 JPQL - 엔티티 직접 사용

- JPQL에서 엔티티를 직접 사용하면 SQL에서 해당 엔티티의 기본 키 값을 사용
  - 엔티티의 아이디를 사용하거나 엔티티를 직접 사용해도 같은 SQL 실행

---

### 💡 Named 쿼리 - 정적 쿼리

- 미리 정의해서 이름을 부여해두고 사용하는 JPQL
- 애플리케이션 로딩 시점에 쿼리를 검증
- Spring Data JPA는 Named Query 방식으로 작동함

```java
// Member
@Entity
@NamedQuery(
        name = "Member.findByUsername",
        query = "SELECT m FROM Member m WHERE m.username = :username"
)
public class Member{}

// Named 쿼리
List<Member> resultList = em.createNamedQuery("Member.findByUsername", Member.class)
                .setParameter("username", "회원1")
                .getResultList();
```

---

### 💡 벌크 연산

- 쿼리 한번으로 여러 테이블 row 변경(엔티티)
- executeUpdate() 결과는 영향받은 엔티티 수 반환
- 벌크 연산은 영속성 컨텍스트 무시하고 데이터베이스에 직접 쿼리
- 벌크 연산 수행 후 영속성 컨텍스트 초기화

```java
// FLUSH 자동 호출
int resultCount = em.createQuery("update Member m set m.age = 20")
        .executeUpdate();
em.clear(); // DB에서 새로 가져와짐

System.out.println("resultCount = " + resultCount);
```

---

### 💡 새롭게 알게 된 점 및 중요한 점 정리

1. 단일 값 연관 필드는 묵시적 내부 조인(inner join) 발생하고, 컬렉션 값 연관 필드는 명시적 조인을 통해 별칭을 통해 탐색 가능
2. 페치 조인 : JPQL에서 성능 최적화를 위해 제공하는 기능, 연관된 엔티티나 컬렉션을 SQL 한번에 함께 조회하는 기능(즉시 로딩), N+1 문제 대부분 해결 가능
3. 벌크 연산은 데이터베이스에 직접 쿼리하기 때문에 영속성 컨텍스트를 우회함, 따라서 벌크 연산 후 데이터를 조회하면 영속성 컨텍스트에 있는 기존 데이터가 반환되므로 최신 데이터를 가져오려면 벌크 연산 수행 후 영속성 컨텍스트를 초기화해야함