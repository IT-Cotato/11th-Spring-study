## 1. 단방향 연관관계

---

### Member가 Team을 외래 키로 갖는 데이터 중심 설계 예시

```java
// Member 클래스
@Entity
public class Member {

	@Id
	@GeneratedValue
	@Column(name = "MEMBER_ID")
	private Long id;

	@Column(name = "USERNAME")
	private String username;

	@JoinColumn(name = "TEAM_ID")
	private Long teamId; // 단순 외래 키 필드
}

// Team 클래스
@Entity
public class Team {

	@Id
	@GeneratedValue
	@Column(name = "TEAM_ID")
	private Long id;

	private String name;
}
```

현재 Member는 Team을 객체로 참조하지 않고, 단순히 외래 키(teamId)만 가진다


**저장 & 조회 로직 흐름**

```java
// 팀 저장
Team team = new Team();
team.setName("Team A");
em.persist(team); // 영속 상태 -> ID 할당

// 회원 저장
Member member = new Member();
member.setUsername("member1");
member.setTeamId(team.getId()); // 외래 키만 설정
em.persist(member);

// 회원 조회
Member findMember = em.find(Member.class, member.getId());

// 회원이 속한 팀 정보 확인
Long findTeamId = findMember.getTeamId();
Team findTeam = em.find(Team.class, findTeamId);
```

**문제점**

- 객체지향적으로 findMember.getTeam() 처럼 바로 접근이 불가능하다
- 매번 ID를 꺼내 다시 찾아야 한다 → 데이터 중심 설계

### **Member가 Team 참조하도록 변경**

```java
@Entity
public class Member {

	@Id
	@GeneratedValue
	@Column(name = "MEMBER_ID")
	private Long id;

	@Column(name = "USERNAME")
	private String username;

	@ManyToOne
	@JoinColumn(name = "TEAM_ID")
	private Team team; // Team 객체를 직접 참조
}
```

**저장 & 조회 로직 흐름 (객체지향적)**

```java
// 팀 저장
Team team = new Team();
team.setName("Team A");
em.persist(team);

// 회원 저장
Member member = new Member();
member.setUsername("member1");
member.setTeam(team); // Team 참조
em.persist(member);

em.flush();
em.clear();

// 회원 조회 (1차 캐시)
Member findMember = em.find(Member.class, member.getId());

// 회원이 속한 팀 조회 (객체 지향적)
Team findTeam = **findMember.getTeam();**
System.out.println("findTeam = " + findTeam.getName());

```

```sql
Hibernate: 
    select
        m1_0.MEMBER_ID,
        t1_0.TEAM_ID,
        t1_0.name,
        m1_0.USERNAME 
    from
        Member m1_0 
    left join
        Team t1_0 
            on t1_0.TEAM_ID=m1_0.TEAM_ID 
    where
        m1_0.MEMBER_ID=?
findTeam = Team A
```

- Member와 Team을 한 번에 JOIN으로 조회해서 가져옴

**FetchType.Lazy 설정 시**

```java
class Member {
	...
	@ManyToOne(**fetch = FetchType.LAZY**) // 지연 로딩
	@JoinColumn(name = "TEAM_ID")
	private Team team;
	...
}
```

```sql
Hibernate: 
    select
        m1_0.MEMBER_ID,
        m1_0.TEAM_ID,
        m1_0.USERNAME 
    from
        Member m1_0 
    where
        m1_0.MEMBER_ID=?
        
Hibernate: 
    select
        t1_0.TEAM_ID,
        t1_0.name 
    from
        Team t1_0 
    where
        t1_0.TEAM_ID=?
```

처음부터 Team을 JOIN하지 않고, 실제 접근 시점(`findMember.getTeam()`)에 별도 쿼리로 조회

→ 성능 최적화!

### **연관 관계 수정**

```java
// Team 변경
Team newTeam = em.find(Team.class, 100L); // 존재하는 Team 가정
findMember.setTeam(newTeam);
```

setTeam()으로 연관 관계 수정

- Member의 Team의 관계를 객체지향적으로 관리할 수 있음
- Team 필드를 가지고 있는 Member의 단방향 연관관계 수정

## 2. 양방향 연관관계와 연관관계의 주인

---

**객체와 테이블의 패러다임 차이**

- 객체는 연관된 대상을 참조로 탐색한다
- 테이블은 외래 키로 JOIN을 통해 연관 데이터를 가져온다

위에서 설계한 Team에는 Member 리스트가 없음

- `member.getTeam()`은 되지만
- `team.getMembers()`는 불가능

### 단방향 → 양방향 매핑으로 변경

```java
@Entity
public class Team {
	...
	
	@OneToMany(mappedBy = "team")
	private List<Member> members = new ArrayList<>();
}
```

```java
// 팀에 속한 회원리스트 접근
Member findMember = em.find(Member.class, member.getId());
List<Member> members = findMember.getTeam().getMembers();
```

### 객체와 테이블의 양방향 연관관계 차이

객체의 양방향 관계는 사실 서로 다른 단방향 2개이다

- 회원 → 팀 단방향 연관관계 1개
- 팀 → 회원 단방향 연관관계 1개

테이블은 **외래 키 하나**로 두 테이블의 연관관계를 관리한다

- 회원 ↔ 팀 양방향 연관관계 1개
- `member.team_id` 외래 키 하나로 양방향 연관관계를 가진다 (양쪽 조인 가능)

```sql
-- 회원에서 팀으로 조인
SELECT *
FROM member m
JOIN team t ON m.team_id = t.team_id;

-- 팀에서 회원으로 조인
SELECT *
FROM team t
JOIN member m ON t.team_id = m.team_id;
```

### **연관관계의 주인**

객체에서는 Member에서 Team을 관리할 수도 있고, Team에서 Member를 관리할 수도 있기 떄문에 둘 중 한 쪽에서 외래 키를 관리해야 한다

연관관계의 주인만이 외래 키를 관리(등록, 수정)할 수 있다

주인이 아닌 반대 쪽은 읽기만 가능하다

주인이 아닌 쪽에서 `mappedBy` 속성을 사용한다

현재 예시를 보면

```java
@Entity
class Member {
	...
	@ManyToOne(fetch = FetchType.LAZY)
	@JoinColumn(name = "TEAM_ID")
	private Team team;
	...
}

@Entity
public class Team {
	...
	
	@OneToMany(mappedBy = "team")
	private List<Member> members = new ArrayList<>();
}
```

Team에서 members 필드에 `mappedBy`로 되어있기 때문에

Member의 team이 연관관계의 주인이다

- Member에서만 Team 수정 가능
- Team에서는 Member 제거나 추가 불가능 (조회만 가능)

### 연관관계 주인 지정 규칙

외래 키가 있는 쪽을 주인으로 정하자

- Member 테이블이 Team의 외래 키를 가지고 있으므로 Member 쪽을 연관관계 주인으로 한다
- `Member.team`이 연관관계의 주인이다 (Team에서 Member 삭제, 등록 등등 관리)
- `Team.members`로는 조회만 가능 (팀에  어떤 회원이 있나 조회만 가능하고, 회원을 추가하거나 삭제 불가능)
- `mappedBy`가 붙은 쪽 (`team.members`)은 가짜 매핑이다

**다중성과 연관관계 주인**

- 외래 키가 있는 곳이 항상 N 쪽이고, 없는 쪽이 항상 1 쪽이다
- 외래 키가 있는 쪽이 `@ManyToOne` 쪽, 없는 쪽이 `@OneToMany` 쪽
- 즉, **N쪽이 연관관계 주인이며, `@**ManyToOne` 쪽이 연관관계 주인이다


> 💡**비즈니스 주인 vs 연관관계 주인 (자동차와 바퀴)**
>
> 비즈니스 주인은 더 중요한 엔티티인 ‘자동차’  
> 연관관게 주인은 외래 키를 가진 1:N 중 N에 해당하는 ‘바퀴’
>

>💡 **외래 키를 가진 쪽을 연관관계 주인으로 설정하는 이유**
>
> 예를 들어 Member에서 Team을 삭제하는 경우  
> Member 테이블에 UPDATE 쿼리가 나가는 등 일관성 있는 동작을 보장한다

## 2. 양방향 매핑 시 주의사항

---

### 1. 연관관계의 주인이 아닌 쪽에서 수정 시 문제

```java
// 회원 저장
Member member = new Member();
member.setUsername("member1");
em.persist(member);

// 팀 저장
Team team = new Team();
team.setName("Team A");
team.getMembers().add(member); // 연관관계의 주인이 아닌 쪽에서 수정 반영 안됨
em.persist(team);
```

```sql
Hibernate: 
    /* insert for
        hellojpa.Member */insert 
    into
        Member (TEAM_ID, USERNAME, MEMBER_ID) 
    values
        (?, ?, ?)
        
Hibernate: 
    /* insert for
        hellojpa.Team */insert 
    into
        Team (name, TEAM_ID)
    values
        (?, ?)
```

INSERT 쿼리는 실행되지만

DB에서 Member를 조회해보면 TEAM_ID에 null이 설정되어 있다

**올바른 방법 - 연관관계 주인 쪽에서 수정**

```java
// 팀 저장
Team team = new Team();
team.setName("Team A");
em.persist(team);

// 회원 저장
Member member = new Member();
member.setUsername("member1");
member.setTeam(team); // 연관관계 주인 쪽에서 수정하기
em.persist(member);
```

### 2. 순수 객체 상태를 고려해서 양방향 매핑을 해주자

Member, Team 두 쪽 모두 값을 설정해줘야 객체지향적이다

```java
// 팀 저장
Team team = new Team();
team.setName("Team A");
em.persist(team); // 영속성 컨텍스트에 저장됨

// 회원 저장
Member member = new Member();
member.setUsername("member1");
member.setTeam(team);
em.persist(member);  // 영속성 컨텍스트에 저장됨

// team.getMembers().add(member); // 연관관계 주인이 아닌 쪽에도 셋팅 해야 함

// em.flush();
// em.clear();
// DB에 반영이 트랜잭션 종료 시에 될 예정

Team findTeam = em.find(Team.class, team.getId()); // 1차 캐시에서 조회
List<Member> members = findTeam.getMembers();

System.out.println("================");
for(Member m : members){
    System.out.println("m = " + m.getUsername());
}
System.out.println("================");

tx.commit();
```

**실행 결과**

```bash

================
================
```

- team의 컬렉션에 아무것도 없어서 members가 비어있다
- `em.flush()`를 중간에 하지 않았기 때문에 findTeam은 DB에서 조회되는게 아니라 1차캐시에서 조회된다
- SELECT 쿼리가 나가지 않는다
- 즉, team.getMembers().add(member)를 하지 않으면 findTeam의 getMembers()는 빈 리스트가 된다

**올바른 방법 - 순수 객체 상태를 고려해서 항상 양쪽에 값을 설정하자!**

- 양방향 매핑 시 = 단방향 2개
- 테스트 케이스도 JPA 없이 순수 자바 코드로 작성한다 (양방향 설정 필요)
- 두 쪽에서 모두 설정하는 것을 까먹을 수 있기 때문에 **연관관계 편의 메서드**를 사용하자

### 연관관계 편의 메서드

```java
class Member {
	...
	
	// Team 셋팅 시 team의 getMembers에도 나 자신을 추가하기
	public void setTeam(Team team) {
		this.team = team;
		team.getMembers().add(this);
	}
```

### 3. 무한 루프 주의

양방향 매핑 시에는 무한 루프를 조심하자 (toString(), JSON 생성 시 등)


> 💡 **실무 팁**
> - 단방향 매핑만으로도 이미 연관관계 매핑은 완료된다 (테이블 설계 끝남)
> - 양방향 매핑은 반대 방향으로 조회하는 기능이 추가된 것 뿐이다
> - 테이블에 영향 주지 않고, 객체 그래프 탐색만 가능하게 함
> - JPQL에서 역방향으로 탐색할 일이 많다
>
>**개발 순서**
>
>1. 단방향 매핑을 먼저 잘 하고
>2. 양방향은 필요할 때 추가해도 된다

---