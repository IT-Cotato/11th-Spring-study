[[자바 ORM 표준 JPA 프로그래밍 - 기본편] 섹션 10.값 타입 / 11.객체지향 쿼리 언어1 - 기본 문법](https://velog.io/@sehyun56/%EC%9E%90%EB%B0%94-ORM-%ED%91%9C%EC%A4%80-JPA-%ED%94%84%EB%A1%9C%EA%B7%B8%EB%9E%98%EB%B0%8D-%EA%B8%B0%EB%B3%B8%ED%8E%B8-%EC%84%B9%EC%85%98-10.%EA%B0%92-%ED%83%80%EC%9E%85-11.%EA%B0%9D%EC%B2%B4%EC%A7%80%ED%96%A5-%EC%BF%BC%EB%A6%AC-%EC%96%B8%EC%96%B41-%EA%B8%B0%EB%B3%B8-%EB%AC%B8%EB%B2%95)

### 💡 JPA의 데이터 타입 분류

- 분류 : 엔티티 타입, 값 타입
- 엔티티 타입
  - @Entity로 정의하는 객체
  - 데이터가 변해도 식별자로 지속해서 추적 가능
- 값 타입
  - int, Integer, String 처럼 단순히 값으로 사용하는 자바 기본 타입이나 객체
  - 식별자가 없고 값만 있으므로 변경시 추적 불가
  - 분류 : 기본값 타입, 임베디드 타입, 컬렉션 값 타입

---

### 💡 기본값 타입

- 자바 기본 타입(int, double), 래퍼 클래스(Integer, Long), String
- 생명주기를 엔티티에 의존
- 공유하면 안 됨
- 자바 기본 타입은 항상 값을 복사함
- 래퍼 클래스나 String 등은 공유 가능한 객체이지만 변경 불가

---

### 💡 임베디드 타입(복합 값 타입)

- 새로운 값 타입을 직접 정의할 수 있음
- 기본 값 타입을 모아서 만들어서 복합 값 타입이라고도 함
- (주소 도시, 주소 번지, 주소 우편번호) → (집 주소) 이런식으로 묶어내고 싶을 때 임베디드 타입 사용

```java
// Address
@Embeddable
public class Address {
    private String city;
    private String street;
    private String zipcode;
}

// Member
@Embedded
private Address homeAddress;
```

- 장점
  - 재사용, 높은 응집도
  - 임베디드 타입을 포함한 모든 값 타입은 값 타입을 소유한 엔티티에 생명주기를 의존함
- 임베디드 타입은 엔티티의 값일 뿐
- @AttributeOverride : 속성 재정의
  - 한 엔티티에서 같은 값 타입을 사용하면 컬럼명 중복 → 컬럼명 속성 재정의

    ```java
    @Embedded
    @AttributeOverrides({
    			@AttributeOverride(name = "city",
    			column = @Column(name = "WORK_CITY")),
    			@AttributeOverride(name = "street",
    			column = @Column(name = "WORK_STREET")),
    			@AttributeOverride(name = "zipcode",
    			column = @Column(name = "WORK_ZIPCODE"))
    })
    private Address workAddress;
    ```


---

### 💡 불변 객체

- 임베디드 타입 같은 값 타입을 여러 엔티티에서 공유하면 위험함(side effect 발생)
- 임베디드 타입처럼 직접 정의한 값 타입은 자바의 기본 타입이 아니라 객체 타입 → 참조 값을 직접 대입하는 것을 막을 방법은 없음
- 객체 타입을 수정할 수 없게 만들면 부작용을 원천 차단 할 수 있음(setter 사용 x, private으로 변경)
- 값 타입은 불변 객체로 설계해야함
- 불변 객체 : 생성 시점 이후 절대 값을 변경할 수 없는 객체 ex) Integer, String

---

### 💡 값 타입의 비교

- 동일성 비교 : 인스턴스의 참조 값 비교(==)
- 동등성 비교 : 인스턴스의 값 비교(equals())
  - equals() 메소드를 적절하게 재정의
  - equals() 정의할 때 해시코드도 구현해야함

    ```java
    @Override
    public boolean equals(Object o) {
        if (o == null || getClass() != o.getClass()) return false;
        Address address = (Address) o;
        return Objects.equals(city, address.city) && Objects.equals(street, address.street) && Objects.equals(zipcode, address.zipcode);
    }
    
    @Override
    public int hashCode() {
        return Objects.hash(city, street, zipcode);
    }
    ```


---

### 💡 값 타입 컬렉션

- 값 타입을 하나 이상 저장할 때 사용
- 데이터베이스는 컬렉션을 같은 테이블에 저장할 수 없음
- 컬렉션을 저장하기 위한 별도의 테이블이 필요함
- 값 타입 컬렉션은 영속성 전이 + 고아 객체 제거 기능을 필수로 가진다고 볼 수 있음
- 값 타입은 식별자 개념이 없음(엔티티 : 식별자 개념 있음)
- 값 타입 컬렉션에 변경 사항 발생 → 모든 데이터 삭제하고 값 타입 컬렉션에 있는 현재 값을 모두 다시 저장
- 대안 : 상황에 따라 값 타입 컬렉션 대신에 일대다 관계 고려 → 엔티티 생성(ex. AddressEntity)

---

### 💡 새롭게 알게 된 점 및 중요한 점 정리

1. 식별자가 필요하고, 지속해서 값을 추적, 변경해야 한다면 값 타입이 아닌 엔티티로 만들어야함
2. JPA의 데이터 타입은 엔티티 타입(식별자o, 추적 가능), 값 타입(식별자x, 추적 불가능)으로 분류됨
3. 값 타입은 불변 객체로 설계해야함
4. 상황에 따라 값 타입 컬렉션 대신에 일대다 관계 고려