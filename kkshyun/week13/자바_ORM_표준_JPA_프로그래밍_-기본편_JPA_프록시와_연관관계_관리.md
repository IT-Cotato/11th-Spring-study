[[자바 ORM 표준 JPA 프로그래밍 - 기본편] 섹션 8. 고급 매핑 / 9. 프록시와 연관관계 관리 ](https://velog.io/@sehyun56/%EC%9E%90%EB%B0%94-ORM-%ED%91%9C%EC%A4%80-JPA-%ED%94%84%EB%A1%9C%EA%B7%B8%EB%9E%98%EB%B0%8D-%EA%B8%B0%EB%B3%B8%ED%8E%B8-%EC%84%B9%EC%85%98-8.-%EA%B3%A0%EA%B8%89-%EB%A7%A4%ED%95%91-9.-%ED%94%84%EB%A1%9D%EC%8B%9C%EC%99%80-%EC%97%B0%EA%B4%80%EA%B4%80%EA%B3%84-%EA%B4%80%EB%A6%AC)

### 💡 프록시 기초

- em.find() : 데이터베이스를 통해서 실제 엔티티 객체 조회
- em.getReference() : 데이터베이스 조회를 미루는 프록시 엔티티 객체 조회

```java
Member m1 = em.find(Member.class, member1.getId());
Member reference = em.getReference(Member.class, member1.getId());
```

---

### 💡 프록시 특징

- 실제 클래스를 상속 받아서 만들어짐
- 실제 클래스와 겉 모양이 같음
- 하이버네이트가 내부적으로 프록시 라이브러리를 써서 만듦
- 프록시는 ID가 아닌 실제 데이터(비 - ID 필드)에 접근할 때 초기화됨. 이때 데이터베이스 쿼리가 발생함
- 프록시 객체는 실제 객체의 참조를 보관
- 프록시에 내용이 없을 때 영속성 컨텍스트를 통해서 초기화를 요청함
- 프록시 객체는 처음 사용할 때만 초기화됨. 초기화되면 프록시 객체를 통해서 실제 엔티티에 접근 가능
- 프록시 객체랑 원본 엔티티 타입 체크할 때는 instance of 사용
- 이미 영속성 컨텍스트에 찾는 엔티티가 있으면 em.getReference()를 호출해도 실제 엔티티 반환
- 한 트랜잭션 안에서 한 영속성에서 가져오고 pk가 같으면 true가 되어야함
- em.detach() 등을 통해 준영속 상태가 되면, 이때 프록시를 초기화하면 org.hibernate.LazyInitializationException 예외를 터트림

---

### 💡 프록시 확인

- 프록시 인스턴스의 초기화 여부 확인
- 프록시 강제 초기화

```java
Hibernate.initialize(refMember); // 강제 초기화(refMember.getName() 등 대신)
System.out.println("isLoaded = " + emf.getPersistenceUnitUtil().isLoaded(refMember)); // true, 초기화됨
```

---

### 💡 즉시 로딩과 지연 로딩

- 지연 로딩
    - LAZY를 사용해서 프록시로 조회
    - 프록시로 가져오고 나중에 접근하는 시점에 가져옴

    ```java
    // Member
    @ManyToOne(fetch = FetchType.LAZY) // 지연로딩
    @JoinColumn(name = "TEAM_ID", insertable = false, updatable = false)
    private Team team;
    
    // main
    Member member = em.find(Member.class, 1L);
    Team team = member.getTeam();
    team.getName(); // 실제 team을 사용하는 시점에 초기화(DB 조회), 이때 쿼리 나감
    ```

- 즉시 로딩
    - EAGER를 사용해서 함께 조회
    - JPA 구현체는 가능하면 조인을 사용해서 SQL 한번에 함께 조회

    ```java
    // Member
    @ManyToOne(fetch = FetchType.EAGER) // 즉시로딩
    @JoinColumn(name = "TEAM_ID", insertable = false, updatable = false)
    private Team team;
    
    // main
    Member member = em.find(Member.class, 1L); // Member 조회시 항상 Team도 조회(즉시로딩)
    Team team = member.getTeam();
    team.getName();
    ```


---

### 💡 주의할 점

- 최대한 지연 로딩 사용
- 즉시 로딩은 JPQL에서 N+1문제를 일으킴
- @ManyToOne, @OneToOne은 기본이 즉시 로딩 → LAZY로 설정
- 일단 지연 로딩으로 다 깔고, 필요하면 fetch join을 사용

    ```java
    List<Member> members = em.createQuery("select m from Member m join fetch m.team", Member.class)
                        .getResultList();
    ```


---

### 💡 영속성 전이 : CASCADE

- 특정 엔티티를 영속 상태로 만들 때 연관된 엔티티도 함께 영속 상태로 만들고 싶을 때 사용
    - 예) 부모 엔티티를 저장할 때 자식 엔티티도 함께 저장
- 영속성 전이는 연관관계 매핑하는 것과는 관련이 없음
- 소유자가 하나일 때 사용해야함
- CASCADE 종류
    - ALL(모두 적용), PERSIST(영속), REMOVE(삭제)

---

### 💡 고아 객체

- 고아 객체 제거 : 부모 엔티티와 연관관계가 끊어진 자식 엔티티를 자동으로 삭제
- orphanRemoval = true
- 참조가 제거된 엔티티는 다른 곳에서 참조하지 않는 고아 객체로 보고 삭제하는 기능
- 참조하는 곳이 하나일 때 사용해야함
- 부모를 제거하면 자식은 고아가 됨 → 고아 객체 제거 기능을 활성화 하면, 부모를 제거할 때 자식도 함께
  제거됨(CascadeType.REMOVE 비슷하게 작동)

---

### 💡 영속성 전이 + 고아 객체

- CascadeType.ALL + orphanRemoval = true
- 두 옵션 모두 활성화하면 부모 엔티티를 통해서 자식의 생명 주기를 관리할 수 있음
    - 자식 생명주기를 부모에 완전히 위임
- 도메인 주도 설계의 Aggregate Root 개념을 구현할 때 유용(repository를 따로 만들지 않고 root로 관리)

```java
 // 영속성 전이가 모두 전이되고, 부모 컬렉션에서 자식이 제거되면 DB에서도 삭제됨
@OneToMany(mappedBy = "parent", cascade = CascadeType.ALL, orphanRemoval = true)
private List<Child> childList = new ArrayList<>();
```

---

### 💡 새롭게 알게 된 점 및 중요한 점 정리

1. 프록시 객체는 ID가 아닌 실제 데이터에 접근할 때 초기화됨
2. 지연 로딩을 사용했을 때 프록시로 가져오고 실제 데이터는 나중에 접근하는 시점에 가져옴
3. @ManyToOne, @OneToOne은 기본이 즉시 로딩이므로 직접 LAZY로 설정
4. CASCADE(영속성 전이) : 특정 엔티티를 영속 상태로 만들 때 연관된 엔티티도 함께 영속 상태로 만들고 싶을 때 사용 ex) 부모 엔티티를 저장할 때 자식 엔티티도 함께 저장
5. 영속성 전이와 고아 객체를 함께 사용하면 부모 엔티티를 통해서 자식의 생명 주기를 관리할 수 있음