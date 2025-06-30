[[자바 ORM 표준 JPA 프로그래밍 - 기본편] 섹션4. 영속성 관리 - 내부 동작 방식 / 5. 엔티티 매핑](https://velog.io/@sehyun56/%EC%9E%90%EB%B0%94-ORM-%ED%91%9C%EC%A4%80-JPA-%ED%94%84%EB%A1%9C%EA%B7%B8%EB%9E%98%EB%B0%8D-%EA%B8%B0%EB%B3%B8%ED%8E%B8-%EC%84%B9%EC%85%984.-%EC%98%81%EC%86%8D%EC%84%B1-%EA%B4%80%EB%A6%AC-%EB%82%B4%EB%B6%80-%EB%8F%99%EC%9E%91-%EB%B0%A9%EC%8B%9D-5.-%EC%97%94%ED%8B%B0%ED%8B%B0-%EB%A7%A4%ED%95%91)

### 💡 엔티티 매핑

- 객체와 테이블 매핑 : @Entity, @Table
- 필드와 컬럼 매핑 : @Column
- 기본 키 매핑 : @Id
- 연관관계 매핑 : @ManyToOne, @JoinColumn

---

### 💡 객체와 테이블 매핑

- @Entity가 붙은 클래스는 JPA가 관리하고 엔티티라고 함
- JPA를 사용해서 테이블과 매핑할 클래스는 @Entity 필수
- 기본 생성자 필수 (public, protected)
    - JPA 구현체는 객체를 동적으로 생성하고 조작하기 위해 기본 생성자를 사용
- @Entity의 name 속성 : JPA에서 사용할 엔티티 이름 지정, 기본 값은 클래스 이름
- @Table : 엔티티와 매핑할 테이블 지정
    - name : 매핑할 테이블 이름(기본으로 엔티티 이름 사용)
    - uniqueConstraints(DDL) : DDL 생성 시에 유니크 제약 조건 생성
        - DDL(데이터 구조 정의, 변경할 때 사용하는 SQL) : create, alter, drop 등
- 데이터베이스 스키마 자동 생성
    - DDL을 애플리케이션 실행 시점에 자동 생성
    - 데이터베이스 방언을 활용해서 데이터베이스에 맞는 적절한 DDL 생성
    - create : 기존 테이블 삭제 후 다시 생성
    - update : 변경분만 반영
    - validate : 엔티티와 테이블이 정상 매핑되었는지만 확인
    - none : 사용하지 않음

  → 운영 장비에는 create, create-drop, update 사용하면 안 됨

- DDL 생성 기능
    - 제약조건 추가 : @Column(nullable = false, length = 10)
    - DDL 생성 기능은 DDL을 자동 생성할 때만 사용되고 JPA의 실행 로직에는 영향을 주지 않음

---

### 💡 필드와 컬럼 매핑

- @Column : 컬럼 매핑
    - name, nullable(DDL), unique(DDL), length(DDL)
- @Temporal : 날짜 타입 매핑
    - TemporalType.DATE, TIME, TIMESTAMP 로 분류됨
    - LocalDate, LocalDateTime 사용하면 생략 가능
- @Enumerated : enum 타입 매핑
    - EnumType.STRING 써야함 : 이름을 데이터베이스에 저장

---

### 💡 기본 키 매핑

- 직접 할당할 때는 @Id 만 사용
- 자동 생성할 때는 @GeneratedValue 사용
    - IDENTITY, SEQUENCE, TABLE, AUTO
    - IDENTITY
        - 기본 키 생성을 데이터베이스에 위임
        - ID 생성을 DB에 위임하므로, ID 값을 알기 위해 em.persist 시점에 즉시 INSERT 쿼리 날림

        ```jsx
        @Entity
        public class Member {
        	@Id
        	@GeneratedValue(strategy = GenerationType.IDENTITY)
        	private Long id;
        }
        ```

    - SEQUENCE : 데이터베이스 시퀀스는 유일한 값을 순서대로 생성하는 특별한 데이터베이스 오브젝트
    - TABLE : 키 생성 전용 테이블을 하나 만들어서 데이터베이스 시퀀스를 흉내내는 전략

---

### 💡 권장하는 식별자 권략

- 기본 키 제약 조건 : null 아님, 유일, 변하면 안 됨
- 권장 : Long 형 + 대체 키 + 키 생성전략 사용

---

### 💡 데이터 중심 설계의 문제점

- 테이블의 외래키를 객체에 그대로 가져오는건 관계형 데이터 베이스에 초점을 맞춘 방식
- 객체 그래프 탐색이 불가능
- 엔티티에서 객체지향적으로 매핑하기 위해 직접 엔티티 참조

---

### 💡 새롭게 알게 된 점 및 중요한 점 정리

1. @Entity : @Entity가 붙은 클래스는 JPA가 관리하고 엔티티라고 함, JPA를 사용해서 테이블과 매핑할 클래스는 @Entity 필수, 기본 생성자 필수
2. LocalDate, LocalDateTime 사용하면 필드에 @Temporal 생략 가능
3. enum 타입 매핑할 때는 @Enumerated(EnumType.STRING)를 사용해야함