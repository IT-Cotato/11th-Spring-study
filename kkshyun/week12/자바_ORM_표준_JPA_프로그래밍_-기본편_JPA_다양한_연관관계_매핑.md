[[자바 ORM 표준 JPA 프로그래밍 - 기본편] 섹션6. 연관관계 매핑 기초 / 7. 다양한 연관관계 매핑](https://velog.io/@sehyun56/%EC%9E%90%EB%B0%94-ORM-%ED%91%9C%EC%A4%80-JPA-%ED%94%84%EB%A1%9C%EA%B7%B8%EB%9E%98%EB%B0%8D-%EA%B8%B0%EB%B3%B8%ED%8E%B8-%EC%84%B9%EC%85%98-6.%EC%97%B0%EA%B4%80%EA%B4%80%EA%B3%84-%EB%A7%A4%ED%95%91-%EA%B8%B0%EC%B4%88-7.-%EB%8B%A4%EC%96%91%ED%95%9C-%EC%97%B0%EA%B4%80%EA%B4%80%EA%B3%84-%EB%A7%A4%ED%95%91)

### 💡 단방향, 양방향, 연관관계의 주인

- 테이블
  - 외래 키 하나로 양쪽 조인 가능
- 객체
  - 참조용 필드가 있는 쪽으로만 참조 가능
  - 한쪽만 참조하면 단방향
  - 양쪽이 서로 참조하면 양방향
  - 객체 양방향 관계는 참조가 2군데 있으므로 둘 중 테이블의 외래 키를 관리할 곳을 정해야 함
- 연관관계 주인 : 외래 키를 관리하는 참조
- 주인의 반대편 : 외래 키에 영향을 주지 않음, 단순 조회만 가능

---

### 💡 다대일 [N:1] @ManyToOne

- 다대일 단방향
  - 가장 많이 사용하는 연관관계

    ```java
    // Member
    @ManyToOne
    @JoinColumn(name = "TEAM_ID")
    private Team team;
    ```

- 다대일 양방향
  - 외래 키가 있는 쪽이 연관관계 주인 → ‘다’가 연관관계 주인
  - 양쪽을 서로 참조하도록 개발

    ```java
    // Member
    @ManyToOne
    @JoinColumn(name = "TEAM_ID")
    private Team team;
    
    // Team
    @OneToMany(mappedBy = "team")
    List<Member> members = new ArrayList<Member>();
    ```


---

### 💡 일대다 [1:N] @OneToMany

- 일대다 단방향
  - ‘일’이 연관관계 주인
  - ‘다’쪽에 외래 키가 있음
  - 객체와 테이블 차이 때문에 반대편 테이블의 외래키를 관리하는 구조
  - 연관관계 관리를 위해 추가로 UPDATE SQL 실행
  - @JoinColumn 을 꼭 사용해야 함 그렇지 않으면 조인 테이블 방식 사용
  - 다대일 양방향 매핑 권장

    ```java
    // Team
    @OneToMany
    @JoinColumn(name = "team_id")
    private List<Member> members = new ArrayList<>();
    ```

- 일대다 양방향
  - 공식적으로 존재하지 않음
  - @JoinColumn(insertable = false, updatable = false) 사용
  - 읽기 전용 필드를 사용해서 양방향처럼 사용하는 방법
  - 다대일 양방향 매핑 권장

    ```java
    // Team
    @OneToMany
    @JoinColumn(name = "team_id")
    private List<Member> members = new ArrayList<>();
    
    // Member
    @ManyToOne
    @JoinColumn(name = "TEAM_ID", insertable = false, updatable = false)
    private Team team;
    ```


---

### 💡 일대일 [1:1] @OneToOne

- 주 테이블이나 대상 테이블 중에 외래 키 선택 가능
- 외래 키에 데이터베이스 유니크 제약 조건 추가
- 주 테이블에 외래 키 설정 권장
- 주 테이블에 외래 키
  - 주 테이블만 조회해도 대상 테이블에 데이터가 있는지 확인 가능
  - 값이 없으면 외래 키에 null 허용

    ```java
    // 주 테이블에 외래 키 양방향
    
    // Member
    @OneToOne
    @JoinColumn(name = "LOCKER_ID") // @JoinColumn(name = "LOCKER_ID", unique = true)
    private Locker locker;
    
    // Locker
    @OneToOne(mappedBy = "locker")
    private Member member;
    ```

- 대상 테이블에 외래 키
  - 주 테이블과 대상 테이블을 일대일에서 일대다 관계로 변경할 때 테이블 구조 유지
  - 프록시 기능의 한계로 지연 로딩으로 설정해도 항상 즉시 로딩됨

---

### 💡 다대다 [N:M] @ManyToMany

- 실무에서 쓰면 안 됨
- 관계형 데이터베이스는 정규화된 테이블 2개로 다대다 관계를 표현할 수 없음
- 연결 테이블(중간 테이블)을 추가해서 일대다, 다대일 관계로 풀어내야함
- 객체는 컬렉션을 사용해서 객체 2개로 다대다 관계 가능
- @ManyToMany 사용, @JoinTable로 연결 테이블 지정
- 한계
  - 편리해보이지만 실무에서 사용하면 안 됨
  - 연결 테이블이 단순히 연결만 하는 것이 아닌 다른 속성도 들어갈 가능성 많음
- 극복
  - 연결 테이블용 엔티티 추가
    - pk는 관계없는 값으로 놓아야 유연성 생김
  - @ManyToMany → @OneToMany, @ManyToOne
  - N:M 관계는 중간 테이블을 이용해서 1:N, N:1로 변경

    ```java
    // Member
    @OneToMany(mappedBy = "member")
    private List<MemberProduct> memberProducts = new ArrayList<>();
    
    // MemberProduct(중간테이블)
    @ManyToOne
    @JoinColumn(name = "MEMBER_ID")
    private Member member;
    
    @ManyToOne
    @JoinColumn(name = "PRODUCT_ID")
    private Product product;
    
    // Product
    @OneToMany(mappedBy = "product")
    private List<MemberProduct> memberProducts = new ArrayList<>();
    ```


---

### 💡 @JoinColumn, @ManyToOne, @OneToMany

- @JoinColumn
  - 외래 키를 매핑할 때 사용
  - name : 매핑할 외래 키 이름, 기본 값 : 필드명 + _ + 참조하는 테이블의 기본 키 컬럼명
- @ManyToOne
  - 다대일 관계 매핑
  - 속성 : optional, fetch, cascade
- @OneToMany
  - 일대다 관계 매핑
  - 속성 : mappedBy, fetch, cascade

---

### 💡 새롭게 알게 된 점 및 중요한 점 정리

1. 다대일 : 가장 많이 사용하는 연관관계
2. 일대다 → 다대일 양방향 매핑 권장
3. 다대다 → 중간 테이블 만들어서 다대일 관계 2개로 표현 권장