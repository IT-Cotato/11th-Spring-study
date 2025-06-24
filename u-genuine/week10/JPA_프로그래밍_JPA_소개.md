## **1. 객체와 RDB의 차이**

### 구조적 차이점

객체는 참조를 통해 관계를 맺고 자유롭게 탐색 가능

RDB는 외래 키로 관계를 표현하고 JOIN으로 탐색

### **상속 관계 매핑의 문제**

```java
// 객체 모델
class Album extends Item {
    String artist;
    String genre;
}
```

객체의 상속 관계를 RDB에 저장하려면

- Item 테이블과 Album 테이블에 각각 INSERT하고, 조회할 때도 항상 JOIN이 필요함
- SQL 작성과 관리가 복잡하고 성능에도 부담

---

## 2. 전통적인 개발 방식의 한계

### **객체다운 모델링**

```java
class Member {
	String id;
	Team team; // Team 객체를 참조. Long teamId; 는 참조가 아님
	String username;
}
```

### 신뢰할 수 없는 엔티티

```java
class MemberService {
	public void process() {
		Member member = memberRepository.find(memberId);
		member.getTeam(); // 이게 DB에서 조회되었는지 불안
		member.getOrder().getDelivery(); // 이것도 불안
	}
}
```

Repository의 SQL 구현을 확인하지 않으면 안전한 코드 작성이 불가하다

> 💡**문제점**
>
> Service 계층에서 getTeam()이나 getOrder().getDelivery()를 호출할 때마다 “과연 이 데이터가 DB에서 조회되었을까? 를 걱정해야 함”
>
> 이는 계층 간의 신뢰를 깨뜨리는 문제
>

### 객체 그래프 탐색의 한계

```sql
SELECT M.*, T.* 
FROM member M JOIN team T ON M.team_id = T.team_id
```

- `member.getTeam()` 가능
- `member.getOrder()` 불가능 (JOIN하지 않았음)

이를 해결하려면 상황별로 조회 메서드가 필요하다

```sql
memberRepository.getMember();
memberRepository.getMemberWithTeam();
memberRepository.getMemberWithOrderWithDelivery();
// 조합이 폭발적으로 증가...
```

### 객체 동일성 보장 불가

```java
// 컬렉션
String memberId = "100";
Member member1 = list.get(memberId);
Member member2 = list.get(memberId);

member1 == member2; // true

// DB
String memberId = "100";
Member member1 = memberRepository.getMember(memberId);
Member member2 = memberRepository.getMember(memberId);

member1 == member2; // false (항상 새 객체 생성)
```

결국, 객체지향적으로 모델링할수록 매핑 작업이 늘어나고 SQL 변환 비용이 증가하게 되는 문제

---

## 3. ORM과 JPA

### ORM이란?

- 객체와 관계형 DB 사이의 불일치를 해결하는 개념 또는 기술
- 객체와 RDB의 불일치를 ORM이 매핑으로 해결
- SQL 작성/수행, 매핑, 상태 관리 자동화

→ ORM은 개념 or 방법론

→ ORM을 구현한 여러 프레임워크와 라이브러리가 존재한다 = JPA

### JPA란?

- 자바에서 ORM을 표준화한 API
- JPA는 직접 DB와 통신하지 않고 JPA 구현체(Hibernate 등)가 실무에서 동작


→ JPA는 ORM을 자바에서 쓰기 위한 표준 인터페이스

→ Hibernate는 JPA를 구현한 대표적인 ORM 프레임워크

---

## 4. JPA의 장점

### 생산성 향상

```java
jpa.persist(member);  // 저장
Member member = jpa.find(Member.class, memberId); // 조회
member.setName("변경할 이름");  // 수정
jpa.remove(member);   // 삭제
```

→ SQL작성, 수동 매핑이 필요 없음

### **유지보수성 개선**

**필드 추가/변경 시**

- JPA: 필드만 추가하면 되고, SQL을 수동으로 수정하지 않아도 된다
- 기존 방식: 모든 SQL을 수동으로 수정

### 상속 관계 매핑

```java
// 저장
jpa.persist(album);
// -> JPA가 자동으로 Item 테이블과 Album 테이블에 각각 INSERT

// 조회
Album album = jpa.find(Album.class, albumId);
// → JPA가 자동으로 필요한 JOIN 쿼리 생성
```

### 연관관계와 객체 그래프 탐색

```java
// 연관관계 저장
member.setTeam(team);
jpa.persist(member);

// 자유로운 객체 그래프 탐색
Member member = jpa.find(Member.class, memberId);
Team team = member.getTeam();
Order order = member.getOrder();
Delivery delivery = order.getDelivery(); 
```

마치 자바 컬렉션처럼 자유롭게 탐색 가능

> 💡 **그럼 jpa.find에서 관련된걸 전부 join하는건가? 그럼 성능은?**
> 
> → 성능최적화까지 고려되어있음(지연 로딩)
> 

---

## 5. JPA 성능 최적화

### 1차 캐시와 동일성 보장

```java
String memberId = "100";
Member member1 = memberRepository.getMember(memberId); // SQL
Member member2 = memberRepository.getMember(memberId); // 캐시

member1 == member2; // true !!!
```

- JPA는 같은 트랜잭션 내 동일 엔티티는 같은 객체를 반환한다
- 자바의 컬렉션처럼 동일성(==)이 보장된다
- 불필요한 DB 접근을 방지하게 된다

### 트랜잭션 지원 쓰기 지연

JPA에서는 persist 호출 시 INSERT SQL을 버퍼에 모아둔다

트랜잭션이 커밋될 때 JDBC BATCH를 통해 한 번에 전송

- 네트워크 비용 절감

### 지연로딩과 즉시로딩

**지연 로딩(Lazy)**

- 객체의 실제 사용 시점에 연관 데이터를 조회

```java
Member member = memberRepository.find(memberId); // SELECT * FROM MEMBER
Team team = member.getTeam();
String teamName = team.3getName(); // SELECT * FROM TEAM
```

장점

- 초기 조회 쿼리가 가볍다
- 필요할 때만 추가 쿼리를 실행하기 떄문에 성능 최적화

**즉시 로딩(Eager)**

- 연관된 데이터를 JOIN으로 한 번에 조회

```java
Member member = memberRepository.find(memberId); 
  // SELECT M.*, T.* FROM member M JOIN team T ...

Team team = member.getTeam();
String teamName = team.getName();
```

장점

- 연관 데이터를 항상 함께 쓴다면 쿼리 수가 줄어든다

단점

- 불필요한 데이터까지 미리 로딩하게 되어 메모리가 낭비되고 SQL이 커진다

→ 기본은 지연 로딩으로 설정 하자

연관 데이터를 항상 함께 조회하는 경우 그 데이터만 즉시로딩으로 수정하자