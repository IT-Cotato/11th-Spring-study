## 📌 경로 표현식

---

객체 그래프를 점(.)을 찍어 탐색하는 방식

→ 상태 필드, 단일 값 연관 필드, 컬렉션 값 연관 필드로 나뉜다

```sql
SELECT m.username -- 상태 필드
FROM  Member m
        JOIN m.team t -- 단일 값 연관 필드
        JOIN m.orders o -- 컬렉션 값 연관 필드
WHERE t.name = '팀A'
```

### 1. 상태 필드

- 단순 값을 저장하는 필드
- 탐색의 끝이며 더 이상 경로 탐색이 불가능함

```sql
-- JPQL, SQL 동일함
SELECT m.username, m.age FROM Member m
```

### 2. 단일 값 연관 경로

- 대상이 엔티티인 경우(@ManyToOne, @OneToOne)
- **묵시적인 내부 조인 발생**

  → join을 명시하지 않아도 실제 쿼리에 내부 조인 발생


⚠️ 실무에서는 **명시적 조인 권장**

```sql
// JPQL
SELECT t.member FROM Team t

// SQL (묵시적 내부 조인 발생!!)
SELECT m.* FROM Team t INNER JOIN Member m ON t.member_id = m.id
```

### 3. 컬렉션 값 연관 경로

- 대상이 컬렉션인 경우 (@OneToMany, @OneToMany)
- **묵시적 내부 조인 발생**
- 탐색의 끝 → 더 이상 경로 탐색 불가 (.size 정도만 가능)

```java
String query = "SELECT t.members FROM Team t"; 
Collection result = em.createQuery(query, Collection.class).getResultList();
```

✅ 명시적 조인으로 별칭을 부여해 사용해야 함

```sql
SELECT m.username FROM Team t JOIN t.members m
```


> ❗ **묵시적 조인은 지양하자**
> 
> - 항상 내부 조인(INNER JOIN)으로 처리됨
> - 쿼리 구조를 파악하기 어렵고, SQL 튜닝에 불리
> - 명시적으로 조인하여 별칭을 부여해야 관리가 쉽다

**예제 정리**

```sql
SELECT o.member.team FROM Orders o // 성공
SELECT t.members FROM Team // 성공 (탐색의 끝)
SELECT t.members.username FROM Team t // 실패 (컬렉션에서 직접 탐색 불가)
SELECT m.username FROM Team t JOIN t.members m // 성공 (명시적 조인 후 탐색)
```

> 💡 **실무 조언**
> 
> - 가급적 **명시적 조인**을 사용하자
> - 조인은 SQL 튜닝에 중요 포인트이다
> - 묵시적 조인은 조인이 일어나는 상황을 한 눈에 파악하기 어렵다

## 📌 Fetch join

---

JPQL에서 성능 최적화를 위해 제공하는 기능

연관된 엔티티나 컬렉션을 SQL 한 번에 함께 조회한다

- SQL의 조인 종류가 아니라 JPQL 전용 기능
- JOIN FETCH 키워드를 사용한다

### 1. 엔티티 Fetch Join

회원과 연관된 팀을 **한 번의 쿼리로 함께 조회**하고 싶은 경우

```sql
-- JPQL
SELECT m FROM Member m JOIN FETCH m.team

-- SQL
SELECT M.*, T.*
FROM MEMBER M 
INNER JOIN TEAM T ON M.TEAM_ID = T.ID
```

- JPQL에는 m만 명시했지만, 실제 SQL은 m.*, T.* 모두 조회한다

**❌ Fetch Join 없이 조회하는 경우**

```java
String query = "SELECT m FROM Member m";
List<Member> result = em.createQuery(query, Member.class).getResultList();

for(Member member : result) {
	System.out.println("member.teamName = " + member.getTeam().getName());
}
```

- member.getTeam().getName()을 호출할 때마다 **팀 엔티티를 지연 로딩**(프록시)
- 회원 수만큼 팀을 다시 조회하게 되어 **N+1 문제 발생**
  - N+1 문제는 즉시 로딩, 지연 로딩 상관 없이 발생 가능

동일한 팀이 존재한다면 이후 조회는 1차 캐시에서 해결되지만

최악의 경우 (조회한 회원들의 팀이 모두 다른 경우) N+1번의 쿼리가 발생한다

**✅ Fetch Join 사용 시**

```java
String query = "SELECT m FROM Member m JOIN FETCH m.team";
List<Member> result = em.createQuery(query, Member.class).getResultList();

for(Member member : result) {
	System.out.println("member.teamName = " + member.getTeam().getName());
}
```

- 회원을 조회하는 순간 팀도 함께 로딩됨
- 팀은 프록시가 아닌 실제 객체
- 지연 로딩X, 즉시 로딩O

### 2. 컬렉션 Fetch Join

일대다 관계에서 연관된 컬렉션을 한 번의 SQL로 조회하고 싶을 때 사용한다

```sql
-- JPQL
SELECT t FROM Team t JOIN FETCH t.members WHERE t.name = '팀A'

-- SQL
SELECT T.*, M.* 
FROM TEAM T 
INNER JOIN MEMBER M ON T.ID = M.TEAM_ID
WHERE T.NAME = '팀A'
```

**⚠️ 문제: 데이터 뻥튀기 발생**

팀A가 회원1, 회원2 두 명의 멤버를 가지고 있는 경우

조인 결과는 아래처럼 두 줄이 된다

| **TEAM.ID** | **TEAM.NAME** | **MEMBER.ID** | **MEMBER.NAME** |
| --- | --- | --- | --- |
| 1 | 팀A | 1 | 회원1 |
| 1 | 팀A | 2 | 회원2 |
- 팀A는 1개지만, 멤버 수 만큼 줄이 생김 (2줄)
- JPA는 이 데이터를 그대로 매핑하면서 Team 객체를 두 번 리스트에 넣을 수 있다
- 반복문을 돌면 중복 출력 발생

```
teamname = 팀A
-> username = 회원1
-> username = 회원2
teamname = 팀A
-> username = 회원1
-> username = 회원2
```

**✅ 해결 방법: DISTINCT**

JPA는 DISTINCT를 단순히 SQL에만 붙이지 않고

애플리케이션 레벨에서 식별자 중복 제거도 함께 수행한다

```sql
-- JPQL
SELECT DISTINCT t 
FROM Team t 
JOIN FETCH t.members 
WHERE t.name = '팀A';
```

> 📌 **DISTINCT가 하는 일**
> 
> 1. SQL 쿼리에 DISTINCT 붙여 중복 레코드 (모든 컬럼이 동일한) 제거
> 2. JPA가 같은 식별자(PK 기준)의 엔티티는 한 번만 수집함

## 📌  Fetch Join vs 일반 Join

---

### **1. 일반 조인 (JOIN)**

```sql
-- JPQL
SELECT t FROM Team t JOIN t.members m WHERE t.name = '팀A'

-- SQL
SELECT T.* 
FROM Team T 
INNER JOIN MEMBER M ON T.ID = M.TEAM_ID 
WHERE T.NAME = '팀A'
```

- 연관된 엔티티는 조회 대상이 아니다
- SELECT 절에 지정한 Team 엔티티만 조회
- members는 아직 로딩되지 않은 상태 → 이후 접근 시 지연 로딩 발생
- 데이터 뻥튀기 발생 가능 (조인 자체는 실행되므로 줄 수는 늘어남)

**출력 시**

```java
for (Team team : result) {
    // members 컬렉션 접근 시 그때 쿼리 발생
    for (Member member : team.getMembers()) {
        System.out.println(member.getUsername());
    }
}
```

- N+1 문제 발생 가능성 존재

### 2. Fetch 조인 (JOIN FETCH)

```sql
-- JPQL
SELECT t FROM Team t JOIN FETCH t.members WHERE t.name = '팀A'

-- SQL
SELECT T.*, M.*
FROM TEAM T
INNER JOIN MEMBER M ON T.ID = M.TEAM_ID
WHERE T.NAME = '팀A'
```

- 연관된 members 컬렉션까지 한 번에 조회
- 즉시 로딩이 일어나고, 프록시 아님
- JPA가 객체 그래프를 한 번에 구성
- N+1 문제 해결 가능

**출력 시**

```java
for (Team team : result) {
    for (Member member : team.getMembers()) {
        System.out.println(member.getUsername());
    }
}
```

- 별도 쿼리 없이, 이미 함께 로딩된 컬렉션 사용
- 단, 컬렉션 조인 시 뻥튀기 가능 → DISTINCT 함께 사용 권장

## 📌 Fetch Join 특징과 한계

---

### 1. 페치 조인 대상에 별칭을 줄 수 없음

```sql
-- 잘못된 JPQL
SELECT t 
FROM Team t 
JOIN FETCH t.members m 
WHERE m.username = '회원1'
```

- m에 별칭을 부여해 조건을 거는 건 불가능
- Fetch join은 **“연관된 모든 데이터를 가져온다”는 개념**이므로
- 멤버 일부만 필터링하면 객체 그래프가 깨질 위험이 있다

📌  **대안**

- 해당 멤버만 필요한 경우, 멤버만 별도의 쿼리로 조회해야 함

### 2. 둘 이상의 컬렉션은 페치 조인 불가

```sql
// 예: 팀 -> 멤버, 팀 -> 프로젝트
SELECT t
FROM Team t
JOIN FETCH t.members
JOIN Fetch t.projects -- 불가
```

- 일대다 + 일대다는 곱집합처럼 작용해서 조인 결과 뻥튀기가 심각해진다
- 심각한 성능 저하 유발

📌  **대안**

- 한 컬렉션만 페치 조인하고, 나머지는 배치 사이즈 or 별도 조회

### 3. 컬렉션 페치 조인 시 페이징 불가

컬렉션 페치 조인과 페이징은 함께 쓸 수 없다

```sql
SELECT t FROM Team t
JOIN FETCH t.members
ORDER BY t.name
-- setFirstResult(), setMaxResults() 등 페이징 메서드 사용 시 예외가 발생함
```

**❗️ 이유: 데이터 뻥튀기**

- 컬렉션 조인은 조인된 행 수만큼 결과가 늘어남
- JPA는 페이징 기준이 되는 Team의 행 수를 정확히 계산할 수 없기 때문에 예외 발생

**예시**

- 팀A(회원 2명), 팀B(회원 3명), 팀C(회원 1명)
- 조인 후 결과는 총 6행
- setMaxResults(2) 사용하면 의도(팀 2개)와 다르게 행 기준으로 2개만 자름 → 데이터 잘림 발생

✅ **해결 방법: 배치 사이즈 (@BatchSize)**

페이징 + 컬렉션 조회를 동시에 하고 싶다면?

- JOIN FETCH 제거
- 대신 @BatchSize로 지연 로딩 최적화

```java
@Entity
public class Team {
	@OneToMany(mappedBy = "team", fetch = FetchType.LAZY)
	@Bacthsize(size = 100) // 배치 사이즈 설정
  private List<Member> members = new ArrayList<>();
}
```

**🧠 배치 사이즈 작동 방식**

- 팀 10개를 조회하고 → 그 다음 컬렉션 members로 접근 시

```sql
SELECT * FROM member WHERE team_id IN(?, ?, ..., ?) -- 최대 100개
```

- 10개의 지연 로딩을 1 개의 쿼리로 대체한다

  → 즉, N+1 쿼리를 1+1 쿼리로 줄여 성능 개선


### ✅  정리

- 컬렉션 페치 조인은 성능 최적화에 유용하지만, 페이징과는 양립할 수 없다
- 실무에서는 보통
  - @OneToMany(fetch = LAZY) 설정으로 기본 지연 로딩을 유지하고
  - 컬렉션 페이징 필요 시 @BatchSize로 쿼리를 최적화한다
- 필요한 정보만 가공해서 쓰는 경우라면 DTO 조회 전략이 가장 효과적이다

## 📌 벌크 연산

---

여러 데이터를 한 번에 업데이트하는 연산

**성능 최적화에 유리**하다 (반복문 + 개별 쿼리보다 빠르다)

### 동작 방식

- 벌크 연산은 **영속성 컨텍스트를 무시**하고 데이터베이스에 직접 쿼리를 실행한다
- 따라서 영속성 컨텍스트와 데이터베이스 상태가 달라질 수 있다 → 데이터 불일치 가능

### 실행 후 처리 방법 (데이터 불일치 방지)

1. 벌크 연산을 먼저 실행

   이후 조회 시, 항상 DB에서 최신 상태를 가져옴


1. 벌크 연산 후 영속성 컨텍스트를 초기화

   초기화 후 조회해야 데이터베이스에서 다시 데이터를 가져옴


> 📌 **참고**
> 
> - flush()는 commit 시점이나 JPQL 실행 시 자동 호출됨
> - 따라서 벌크 연산 시에도 실행 전 flush가 자동으로 이뤄짐

### **예시**

```java
int count = em.createQuery(
			"UPDATE Member m SET m.age = 20 WHERE m.age < 20"
			).executeUpdate();
			
em.clear(); // 벌크 연산 후 초기화

Member member = em.find(Member.class, memberId); // 변경된 값으로 재조회
```
