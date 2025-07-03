## **EntityManagerFactory와 EntityManager**

EntityManagerFactory는 클라이언트의 요청이 올 때마다 EntityManager들을 생성한다

EntityManager는 커넥션을 사용해 DB에 접근한다

---

## 영속성 컨텍스트

**엔티티를 영구 저장하는 환경**

`EntityManager.persist(entity);` 는 사실 DB에 바로 저장하는 것이 아니라, Entity를 영속성 컨텍스트에 저장하는 메서드다

EntityManager를 통해 영속성 컨텍스트에 접근하며, EntityManager마다 영속성 컨텍스트가 생성된다 (1:1 관계)

---

## Entity의 생명주기

- 비영속
- 영속
- 준영속
- 삭제


### 비영속


- 객체를 생성한 상태
- 영속성 컨텍스트와 전혀 관련이 없는 상태

### 영속




- 객체가 영속성 컨텍스트에 저장되어 관리되는 상태

### 준영속

- 영속 상태였던 엔티티가 영속성 컨텍스트에서 분리된 상태
- DB에는 남아있으나 영속성 컨텍스트가 관리하지 않는다.

### 삭제

- 영속성 컨텍스트에서 제거되어 DB에서도 삭제된 상태

```java
// 비영속
Member member = new Member();
member.setId("member1");
member.setUsername("회원1"); 

EntityManager em = emf.createEntityManager(); // 엔티티 매니저 팩토리에서 엔티티 매니저를 얻어옴
em.getTransaction().begin();

// 영속 (객체를 저장한 상태)
em.persist(member);

// 영속성 컨텍스트에서 분리, 준영속 상태 (DB에는 있음)
em.detach(member);

// 객체를 삭제한 상태 (삭제)
em.remove(member);
```

---

## 영속성 컨텍스트의 이점

- 어플리케이션과 DB 사이에 중간 계층 역할을 함
- 중간 계층이 존재하면 버퍼링, 캐싱 등 기능이 가능해짐

### 1차 캐시

영속성 컨텍스트 내부에는 1차 캐시가 존재한다 (1차 캐시 = 영속성 컨텍스트)

구조

- Key : id
- Valie : Entity

엔티티를 조회하면 DB 접근 전에 1차 캐시에서 먼저 해당 엔티티가 있는지 조회한다

- 만약 1차 캐시에 없으면 DB를 조회해서 1차캐시에 저장하고 반환한다
- 이후에 다시 조회할 땐 1차 캐시에서 반환

```java
// DB에서 조회 (SELECT 쿼리가 실행됨)
Member member1 = em.find(Member.class, 101L);

// 1차 캐시에서 조회 (SELECT 쿼리 실행X)
Member member2 = em.find(Member.class, 101L);
```

다만 1차 캐시는 하나의 요청(트랜잭션) 안에서만 유지되기 때문에 큰 성능 이득은 아니다

- EntityManager는 트랜잭션이 끝날 때 종료됨 (1차 캐시=영속성컨텍스트도 트랜잭션이 종료될 때 사라짐)
- 따라서 고객의 요청마다 새로운 1차캐시가 생성되는 것이고, 여러 고객들이 공유하는 캐시가 아님(이건 2차 캐시)
- 하나의 트랜잭션 내에서만 도움이 된다

### 동일성 보장

```java
Member member1 = em.find(Member.class, 101L);
Member member2 = em.find(Member.class, 101L);

member1 == member2; // true
```

JPA는 영속 엔티티의 동일성을 보장해준다 (마치 자바 컬렉션처럼)

1차 캐시가 있기 때문에 가능한 것이다

- 1차 캐시로 REPEATABLE READ 등급의 트랜잭션 격리 수준을 애플리케이션 차원에서 제공해준다

### 트랜잭션을 지원하는 쓰기 지연 (버퍼링)

트랜잭션 커밋 시까지 실제 INSERT 쿼리를 날리지 않고 **쓰기 지연 SQL 저장소**에 모아두고

커밋 시점에 한번에 DB에 INSERT SQL을 날린다 (성능 굳)

모아서 한번에 처리하는 것 = 배치

배치 사이즈 = 한번에 모아서 보내는 개수

```java
Member member1 = new Member(101L, "A"); 
Member member2 = new Member(102L, "B");

em.persist(member1);
em.persist(member2);
// 여기까지 INSERT SQL을 데이터베이스에 보내지 않는다

transaction.commit(); // 트랜잭션 커밋. 이때 한번에 보냄
```


### 변경 감지(dirty checking)

```java
// 영속 엔티티 조회
Member memberA = em.find(Member.class, "memberA");

// 영속 엔티티 데이터 수정
memberA.setUsername("hi");
memberA.setAge(10);

// em.persist(memberA); -> 필요 없는 코드
transaction.commit();

```

객체의 값만 바꿨고 em.persist를 호출하지 않았는데 어떻게 DB에 자동으로 update 쿼리가 나가는 것일까?

트랜잭션을 커밋할 때

- flush()가 호출됨
- 엔티티와 스냅샷(영속성 컨텍스트에서의 처음 엔티티 상태) 비교함
- 스냅샷과 엔티티 상태가 다른 경우 쓰기 지연 SQL 저장소에 update 쿼리를 저장해두고 DB에 flush될 때 반영

**flush**

- 쓰기 지연 SQL 저장소의 쿼리를 DB에 전송하는 것 (등록, 수정, 삭제 쿼리)
- DB 트랜잭션이 커밋되면 flush가 자동으로 발생한다
- 영속성 컨텍스트의 변경 내용을 데이터베이스에 (커밋 직전에) 동기화
- 수정된 엔티티를 쓰기 지연 SQL 저장소에 등록한다

**영속성 컨텍스트를 플러시하는 방법**

- em.flush() - 직접 호출
- 트랜잭션 커밋 - 플러시 자동 호출
- JPQL 쿼리 실행 - 플러시 자동 호출

---

### **준영속 상태**

- 영속 → 준영속
- 영속 상태의 엔티티가 영속성 컨텍스트에서 분리(detached)
- 영속성 컨텍스트가 제공하는 기능을 사용하지 못함

```java
// 영속
Member member = em.find(Member.class, 150L);
member.setName("AAAAA");

// 영속성 컨텍스트에서 분리
em.detach(member); 

// update 쿼리 실행되지 않음. 영속성 컨텍스트가 더이상 관리 하지 않는 준영속 상태의 엔티티여서
tx.commit();
```

**준영속 상태로 만드는 방법**

em.detach(entity)

- 특정 엔티티만 준영속 상태로 전환

em.clear()

- 영속성 컨텍스트를 완전히 초기화

em.close()

- 영속성 컨텍스트를 종료