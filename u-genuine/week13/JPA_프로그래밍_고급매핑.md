# 상속 관계 매핑

객체에는 상속이 있지만, RDB에는 없다

하지만 RDB에서는 논리적으로 **슈퍼타입-서브타입 관계라는 개념이 있고,** 이를 JPA에서도 사용할 수 있다

이 관계를 물리적인 테이블로 구현하는 방법에는 총 세 가지가 있다

1. 각각 테이블로 변환 → **JOIN 전략**
2. 통합 테이블로 변환 → **단일 테이블 전략**
3. 서브타입 테이블로 변환 → **구현 클래스마다 테이블 전략**

### 참고: JPA의 기본 전략은 단일 테이블 전략

```java
@Entity
public abstract class Item {

	@Id
	@GeneratedValue
	private Long id;

	private String name;
	private int price;
}

@Entity
public class Album extends Item {
	private String artist;
}

@Entity
public class Movie extends Item {
	private String director;
	private String actor;
}

@Entity
public class Book extends Item {
	private String author;
	private String isbn;
}
```
<img src="img/image.png" width="400"/>


세 가지 방법을 하나씩 살펴보자 !

## 1. 각각 테이블로 변환 → JOIN 전략

---

<img src="img/join전략.png" width="800"/>


```java
@Entity
@Inheritance(strategy = InheritanceType.JOINED) // 조인 전략 적용
@DiscriminatorColumn // 구분 컬럼(DTYPE) 추가
public abstract class Item {
	...
}
```

<img src="img/join전략2.png" width="300"/>

각 엔티티마다 테이블을 분리하고, JOIN을 통해 데이터를 조회한다

DTYPE 컬럼은 기본값으로 엔티티명이 들어가고, 어떤 하위 타입인지 구분 가능하다

테이블 구조가 정규화되고, 외래 키 제약 조건 적용 가능하다

### JOIN 전략 장점

정규화 구조

외래키 제약 가능

저장공간 효율적

### JOIN 전략 단점

JOIN이 많아져 조회 성능이 저하됨

INSERT 시 2번 쿼리 나감 (부모 + 자식)

## 2. 통합 테이블로 변환 → 단일 테이블 전략

---

<img src="img/단일테이블전략.png" width="800"/>

```java
@Entity
@Inheritance(strategy = InheritanceType.SINGLE_TABLE) // 단일 테이블 전략 적용
public abstract class Item {
	...
}
```

<img src="img/단일테이블전략2.png" width="400"/>

단일 테이블 전략에서는 `@DiscriminatorColumn` 어노테이션을 붙이지 않아도 DTYPE 컬럼이 자동으로 생성된다

JOIN 전략과 다르게 단일 테이블 전략은 하나의 테이블에서 모든 데이터를 관리하기 때문에 DTYPE 컬럼이 없이는 구분이 안되기 때문이다

### 삽입

<img src="img/단일테이블전략3.png" width="400"/>

테이블이 한 개니까 데이터가 삽입될 때 1개의 INSERT 쿼리만 나간다

### 조회

<img src="img/단일테이블전략4.png" width="300"/>


테이블이 한 개니까 조회 시 JOIN을 하지 않는 걸 볼 수 있다

→ 성능이 좋다!

### 단일 테이블 전략 장점

JOIN 없이 조회 가능 → 성능 빠름

쿼리가 단순하다

### 단일 테이블 전락 단점

자식 엔티티 필드 중 일부는 무조건 null 허용

테이블이 커질 수 있고, 데이터가 많아지면 오히려 성능 저하될 수 있음

## 3. 서브타입 테이블로 변환 → 구현 클래스마다 테이블 전략

---

<img src="img/서브타입테이블.png" width="800"/>

```java
@Entity
@Inheritance(strategy = InheritanceType.TABLE_PER_CLASS) // 구현 클래스마다 테이블 전략 적용
public abstract class Item {
	...
}
```

부모 테이블(Item)없이, 각 자식 클래스마다 독립된 테이블을 가짐

공통 속성도 각 테이블에 반복됨 (name, price)

### 삽입
<img src="img/서브타입테이블2.png" width="400"/>


Movie 테이블에만 데이터를 삽입해서 쿼리가 1번만 나간다

### 조회
<img src="img/서브타입테이블3.png" width="250"/>


Movie를 직접 조회하면 Movie 테이블에서만 조회된다

### 문제점

부모 타입으로 (Item) 조회 시 → UNION ALL로 모든 자식 테이블을 뒤져야 해서 성능 저하

```sql
Hibernate: 
	select
	    i1_0.id, i1_0.clazz_, i1_0.name, i1_0.price,
	    i1_0.artist, i1_0.author, i1_0.isbn,
	    i1_0.actor, i1_0.director
	from (
	    select price, id, artist, name,
	           null as author, null as isbn,
	           null as actor, null as director,
	           1 as clazz_
	    from Album
	    union all
	    select price, id, null as artist, name,
	           author, isbn,
	           null as actor, null as director,
	           2 as clazz_
	    from Book
	    union all
	    select price, id, null as artist, name,
	           null as author, null as isbn,
	           actor, director,
	           3 as clazz_
	    from Movie
	) i1_0
	where i1_0.id = ?
```

**왜 그럴까?**

부모 테이블이 없기 때문에 id와 일치하는 데이터를 찾기 위해 자식 테이블을 모두 뒤져야 한다

어느 자식 테이블에 해당 id의 데이터가 있는지 알 방법이 없기 때문

### 장점

서브타입 명확히 구분 간으

NOT NULL 제약 조건 사용 가능

### 단점

부모 타입으로 조회 시 성능 나쁨 (UNION ALL)

자식 테이블 통합 조회가 어려움

실무에서 거의 사용하지 않는다!!

## 정리

| 전략 | 장점 | 단점 |
| --- | --- | --- |
| JOIN 전략 | 정규화, 외래키 제약, 저장 공간 효율 | JOIN이 많아 성능 저하, INSERT 2번 |
| 단일 테이블 | 성능 빠름, 쿼리 단순 | null 컬럼 많음 ,테이블 비대해질 수 있음 |
| 구현 클래스별 | 서브타입 명확, not null 가능 | 성능 나쁨, 조회 어려움, 쓰지 말 것 |

# @MappedSuperclass

---

상속 관계는 아니지만

공통 필드만 상속해서 중복을  제거하고 싶을 때 사용

예:

- createdBy
- createdDate
- updatedBy
- updatedDate
- 등..

```java
@MappedSuperclass
public abstract class BaseEntity {
	
	private String createdBy;
	private LocalDateTime createdDate;
	private String updatedBy;
	private LocalDateTime updatedDate;
}

@Entity
public class Member extends BaseEntity {
	...
}
```

@MappedSuperclass는 테이블이 생성되지 않고

매핑 정보만 자식 엔티티에 제공한다

이 엔티티를 직접 생성해서 사용할 일 없으므로 추상 클래스로 만들자

<img src="img/baseentity.png" width="400"/>
