# 프록시

---

Member와 Team이 있을때,

member만 필요하면 굳이 team까지 같이 조회할 필요가 없다

하지만 어떤 상황에서는 둘 다 필요할 수도 있다

어떻게 해야 좋을까?

## em.find() vs em.getReference()

- em.find → DB에서 실제 엔티티 객체를 조회
- em.Reference() → DB 조회를 미루는 프록시 객체를 반환 (가짜 껍데기)

<img src="img/프록시.png" width="400"/>

### **프록시 객체란?**

프록시는 실제 클래스를 **상속받아 만들어진 가짜 객체**라서

겉보기엔 진짜 객체와 **똑같이 생겼다**

내부에는 실제 객체를 참조하는 target이라는 필드가 있고,

프록시가 호출되면 실제 객체의 메서드를 대신 호출해 준다

### 프록시 초기화 과정

프록시에 없는 값을 요청할 경우..

1. 프록시는 영속성 컨텍스트에 초기화를 요청한다
2. 영속성 컨텍스트는 DB에서 실제 엔티티를 조회한다
3. 그 결과로 프록시 내부의 target에 실제 엔티티가 채워진다
4. 이후부터는 프록시가 실제 객체처럼 동작한다

> 프록시는 처음 사용할 때 딱 한번만 초기화된다
>


> ⚠️ **프록시는 실제 객체로 바뀌지 않는다**
>
>초기화가 되어도 프록시 객체는 여전히 프록시다
>
>프록시 내부에 target이 채워져서 실제 데이터에 접근할 수 있을 뿐이다

> 🚫 **프록시 vs 실제 객체 비교**
>
>== 비교는 절대 금지 → false 나올 수 있음 (타입 다름)
>
>반드시 instanceof로 비교해야 안전함
>
>```java
>if (entity instanceof Member) {
>    ...
>}
>```
>
>왜냐하면 코드 어디에서든 프록시가 들어올 수도, 실제 객체가 들어올 수도 있기 때문

### 프록시와 영속성 컨텍스트의 관계

이미 member가 영속성 컨텍스트에 올라와 있으면

em.getReference()를 호출해도 프록시가 아닌 진짜 객체가 반환된다

<img src="img/프록시2.png" width="300"/>


반대로, 먼저 em.getReference()로 프록시를 얻고

그 다음에 em.find()를 호출해도

프록시가 그대로 반환된다

<img src="img/프록시3.png" width="400"/>


왜냐면 같은 트랜잭션, 같은 영속성 컨텍스트에서는

동일한 ID의 엔티티는 == 비교시 JPA는 true를 보장해야 하기 때문이다

## 프록시 상태 확인 방법

- **초기화 여부 확인:**

  PersistenceUnitUtil.isLoaded(Object entity)

<img src="img/프록시4.png" width="500"/>

<img src="img/프록시5.png" width="300"/>


- 프록시 강제 초기화:

  Hibernate.initialize(Object entity)

<img src="img/프록시6.png" width="300"/>


# 즉시 로딩과 지연 로딩

---

## 지연 로딩(LAZY)

```java
@Entity
public class Member extends BaseEntity{
	...
	@ManyToOne(fetch = FetchType.LAZY) // 지연 로딩
	@JoinColumn(name = "TEAM_ID")
	private Team team;
	...
}
```

```java
Member m = em.find(Member.class, member.getId()); // Member만 조회
System.out.println("m = " + m.getTeam().getClass()); // Proxy 클래스

System.out.println("==========");
m.getTeam().getName(); // 실제 사용되는 시점. 이때 DB 조회 발생
System.out.println("==========");
```
<img src="img/lazy.png" width="300"/>


지연로딩은 team 내부에 접근하기 전까지 member에 대한 select 쿼리만 나간 걸 볼 수 있다

그리고 === 로 감싸진 부분의 쿼리를 보면

**실제 team을 사용하는 시점**에 Team 프록시를 초기화한다 (DB에서 Team 조회)

## 즉시 로딩(EAGER)

```java
@Entity
public class Member extends BaseEntity{
	...
	@ManyToOne(fetch = FetchType.EAGER) // 즉시 로딩
	@JoinColumn(name = "TEAM_ID")
	private Team team;
	...
}
```
<img src="img/eager.png" width="300"/>

즉시 로딩이면 Member를 조회할 때부터 Team까지 LEFT JOIN 해서 가져옴

→ 원하지 않는 쿼리가 나갈 수 있다

>💡 **실무에서는 **지연로딩**만 사용하자 !!**
>
>- 즉시 로딩을 적용하면 예상 못한 SQL이 발생하고
>- 즉시 로딩은 JPQL에서 N+1 문제가 발생한다
>- @ManyToOne, @OneToOne은 기본값이 즉시로딩이니까 Lazy로 바꾸자
>- @OneTomany, @ManyToMany는 기본이 Lazy

## **N+1 문제란?**

```java
List<Member> members = em.createQuery("SELECT m FROM Member m", Member.class)
                         .getResultList();

for (Member m : members) {
    System.out.println(m.getTeam().getName()); // 여기에 SELECT N번 발생!
}
```

엔티티 **리스트를 조회**한 뒤, 각 엔티티의 **연관 객체를 접근할 때마다**

**추가로 SELECT 쿼리가 N번 발생하는 현상**

→ 총 N+1번의 쿼리 발생

### 원인

- EAGER든 LAZY든 접근할 때마다 SELECT가 나가면 발생
- 특히 JPQL에서 리스트 조회할 때 자주 발생

### **해결 방법 → Fetch Join**

```java
SELECT m FROM Member m JOIN FETCH m.team
```

JPQL에서 명시적으로 JOIN을 사용해

한 번의 쿼리로 관련된 데이터까지 모두 가져오도록 지정하는 방법

져올 수 있는 걸 N번 나누면 → 💣 성능 폭탄

# 영속성 전이: CASCADE

---

<img src="img/cascade.png" width="400"/>

부모, 자식1, 자식2 모두 각각 em.persist() 하는 것은 너무 번거롭다!

부모에 자식들을 추가해뒀다면, 부모만 persist 해도 자식까지 자동 저장 되면 좋지 않을까 ?

→ 이걸 가능하게 해주는 것이 cascade 옵션이다

```java
@Entity
public class Parent {
	...
	
	@OneToMany(mappedBy = "parent", cascade = CascadeType.ALL) // Cascade!!
	private List<Child> childList = new ArrayList<>();
	
	...
}
```
<img src="img/cascade2.png" width="300"/>
<img src="img/cascade3.png" width="300"/>

```sql
Hibernate: 
    /* insert for
        hellojpa.Parent */insert 
    into
        Parent (name, id) 
    values
        (?, ?)
Hibernate: 
    /* insert for
        hellojpa.Child */insert 
    into
        Child (name, parent_id, id) 
    values
        (?, ?, ?)
Hibernate: 
    /* insert for
        hellojpa.Child */insert 
    into
        Child (name, parent_id, id) 
    values
        (?, ?, ?)
```

em.persist(parent) 한 번으로 자식 2명까지 함께 저장되었다

### CascadeType 종류

| 타입 | 의미 |
| --- | --- |
| ALL | 모든 cascade 동작 (persist, remove 등) |
| PERSIST | 저장(persist)만 전파 |
| REMOVE | 삭제(remove)만 전파 |


>⚠️**Cascade 사용 시 주의할 점**
>
>자식이 부모에게만 종속되어 있을 때만 사용해야 한다
>
>즉, 자식의 소유자가 오직 하나일 때만 적용
>
>하지만 자식이 여러 부모 (또는 다른 엔티티)와 관계가 있다면?
>
>→ 절대 cascade 쓰면 안 됨 (예상치 못한 삭제 발생 가능)

# 고아 객체 (orphanRemoval)

---

```java
@Entity
public class Parent {
	...
	
	@OneToMany(mappedBy = "parent", cascade = CascadeType.ALL, **orphanRemoval = true)**
	private List<Child> childList = new ArrayList<>();
	
	...
}
```
<img src="img/orphan.png" width="400"/>
<img src="img/orphan2.png" width="300"/>

부모 컬렉션에서 자식이 제거되면

DB에서도 자동으로 DELETE 발생

> ⚠️**주의**
>
>자식이 오직 부모에게만 소속되어 있을 때만 사용

# 생명주기 통합 관리

---

**CascadeType.ALL** + **orphanRemoval = true**

- 부모 persist/remove → 자식도 함께 저장/삭제
- 완전히 부모가 자식의 생명주기를 관리하는 구조
- 도메인 주도 설계(DDD)에서 Aggregate Root 개념과 일치