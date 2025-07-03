# 객체와 테이블 매핑

## @Entity

@Entity가 붙은 클래스는 JPA가 관리하는 엔티티

JPA를 사용해서 테이블과 매핑할 클래스는 **@Entity 필수**

**주의**

- 기본 생성자가 필수(public or protected)
- final 클래스, enum, interface, inner 클래스는 사용 X
- 저장할 필드에 final 사용 X

---
# 데이터베이스 스키마 자동 생성

DDL을 애플리케이션 실행 시점에 자동으로 생성한다

데이터베이스 방언을 활용해서 데이터베이스에 맞는 DDL 생성

생성된 DDL은 운영 서버에서는 다듬어서 사용하거나, 사용하지 말아야 함

### 데이터베이스 스키마 자동 생성 속성

create

- 기존 테이블 DROP 후 다시 CREATE

create-drop

- create와 같으나 종료 시점에 DROP

update

- 변경분만 반영 (운영 DB에서는 사용 X)

validate

- 엔티티와 테이블이 정상적으로 매핑되었는지만 확인

운영 장비에서는 절대 create, create-drop, update 사용 XXXX

- 개발 초기 단계는 create 또는 update
- 테스트 서버는 update 또는 validate
- 스테이징과 운영 서버는 validate 또는 none

### DDL 생성 기능

@Column(nullable = false, length = 10, unique = true)

→ 위와 같은 DDL 생성 기능은 DDL을 자동 생성할 때만 사용되고 JPA 실행 로직에는 영향을 주지 않는다. ex) 런타임 시 유니크하지 않은지 체크하거나 필드 길이가 10이 넘은지 체크하거나 등등 X

---
# 필드와 컬럼 매핑

```java
@Entity
public class Member {

	@Id
	private Long id;
	
	@Column(name = "name") // 컬럼 매핑
	private String username;
	
	private Integer age;
	
	@Enumerated(EnumType.STRING)
	private RoleType roleType;
	
	@Temporal(TemporalType.TIMESTAMP)
	private Date createdDate;
	
	@Temporal(TemporalType.TIMESTAMP)
	private Date lastModifiedDate;
	
	@Lob
	private String description;
	//Getter, Setter…
}
```

### @Column

- name: 필드와 매핑할 컬럼 이름
- insertable, updatable: 등록, 변경 가능 여부
- nullable
- unique: @Table 어노테이션의 uniqueConstraint와 같지만, 한 컬럼에 간단히 유니크 제약 조건을 걸 때 사용
- columnDefinition: 데이터베이스 컬럼 정보를 직접 설정
- length
- precision, scale
    - BigDecimal, BigInteger 타입에서 사용
    - precision: 소수점을 포함한 전체 자릿수
    - scale: 소수의 자릿수

### @Enumerated

자바의 enum 타입 매핑할 때 사용

EnumType.ORDINAL → 사용하지 말기 !!

- enum의 순서(0, 1, 2, ..) 로 저장함

EnumType.STRING 을 사용하자!

### @Transient

데이터베이스에 해당 필드를 저장하지도 조회하지도 않음

메모리 상에서 임시로 어떤 값을 보관하고 싶을 때 사용

---
# 기본키  매핑

## @Id

기본키 직접 할당할 때 사용

## @GeneratedValue (strategy = GenerationType.전략 이름)

기본키를 자동 생성할 때 사용

### @GeneratedValue (strategy = GenerationType.**IDENTITY**)

기본 키 생성을 데이터베이스에 위임한다 (MySQL에서는 **AUTO_INCREMENT**)

JPA는 보통 트랜잭션 커밋 시점에 INSERT SQL을 실행하는데,

**AUTO_INCREMENT**는 데이터베이스에 INSERT SQL을 실행한 이후에 ID 값을 알 수 있다.

따라서 IDENTITY 전략은 트랜잭션 커밋 시점이 아니라, em.persist()시점에 즉시 INSERT SQL을 실행하고, ID값을 얻어 1차 캐시에 저장한다

```java
@Entity
public class Member {
	@Id
	@GeneratedValue(strategy = GenerationType.IDENTITY)
	private Long id;
}
```

### @GeneratedValue (strategy = GenerationType.SEQUENCE)

오라클, PostgreSQL, DB2, H2 데이터베이스에서 사용 가능

데이터베이스 시퀀스는 유일한 값을 순서대로 생성하는 특별한 데이터베이스 오브젝트이다

```java
@Entity
@SequenceGenerator(
	name = “MEMBER_SEQ_GENERATOR",
	sequenceName = “MEMBER_SEQ", // 매핑할 데이터베이스 시퀀스 이름
	initialValue = 1, allocationSize = 1)
public class Member {

	@Id
	@GeneratedValue(strategy = GenerationType.SEQUENCE,
	generator = "MEMBER_SEQ_GENERATOR")
	private Long id; 
}
```

SEQUENCE - @SequenceGenerator

- name: 식별자 생성기 이름
- sequenceName: 데이터베이스에 등록되어 있는 시퀀스 이름
- initialValue: 시퀀스 DDL을 생성할 때 처음 시작하는 수 지정
- allocationSize: 시퀀스 한 번 호출에 증가하는 수 (성능 최적화에 사용됨)
    - 데이터베이스 시퀀스 값이 하나씩 증가하도록 설정되어 있으면 이 값을 반드시 1로 설정해야 함. 기본 값이 50이라

### @GeneratedValue (strategy = GenerationType.TABLE)

키 생성 전용 테이블을 하나 만들어, 데이터베이스 시퀀스를 흉내내는 전략

모든 데이터베이스에 사용 가능하다는 장점이 있지만, 성능 단점 존재

```sql
create table MY_SEQUENCE (
	sequence_name varchar(255) not null,
	next_val bigint,
	primary key ( sequence_name )
}
```

```java
@Entity
@TableGenerator(
	name = "MEMBER_SEQ_GENERATOR", // 식별자 생성기 이름
	table = "MY_SEQUENCES", // 키 생성 테이블 명
	pkColumnValue = “MEMBER_SEQ", // 키로 사용할 값 이름
	allocationSize = 1) // 시퀀스 한 번 호출에 증가하는 수(기본 값 50)
public class Member {

	@Id
	@GeneratedValue(strategy = GenerationType.TABLE,
	generator = "MEMBER_SEQ_GENERATOR")
	private Long id;
}
```

TABLE - @TableGenerator

- name: 식별자 생성기 이름
- table: 키 생성 테이블 명
- pkColumnName: 시퀀스 컬럼 명 (기본 값: sequence_name)
- valueColumnNa: 시퀀스 값 컬럼 명 (기본 값: next_val)
- pkColumnValue: 키로 사용할 값 이름 (기본 값: 엔티티 이름)
- initialValue: 초기 값. 마지막으로 생성된 값이 기준 (기본 값: 0)
- allocationSize: 시퀀스 한 번 호출에 증가하는 수 (기본 값: 50)
- catalog, schema
- uniqueConstraint: 유니크 제약 조건 지정 가능

> 💡권장하는 식별자 전략
> 
> 기본 키 제약 조건: null이 아니어야 하고, 유일하며, 변해서는 안됨
> 
> - 미래까지 이 조건을 만족하는 자연키를 찾기 어렵다. 대체키를 사용하자
> - 예를 들어 주민등록 번호도 기본 키로 적절하지 않다
> 
> 권장: Long형 + 대체키 + 키 생성 전략 사용
