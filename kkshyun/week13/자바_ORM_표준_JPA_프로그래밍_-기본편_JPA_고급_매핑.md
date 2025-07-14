[[자바 ORM 표준 JPA 프로그래밍 - 기본편] 섹션 8. 고급 매핑 / 9. 프록시와 연관관계 관리 ](https://velog.io/@sehyun56/%EC%9E%90%EB%B0%94-ORM-%ED%91%9C%EC%A4%80-JPA-%ED%94%84%EB%A1%9C%EA%B7%B8%EB%9E%98%EB%B0%8D-%EA%B8%B0%EB%B3%B8%ED%8E%B8-%EC%84%B9%EC%85%98-8.-%EA%B3%A0%EA%B8%89-%EB%A7%A4%ED%95%91-9.-%ED%94%84%EB%A1%9D%EC%8B%9C%EC%99%80-%EC%97%B0%EA%B4%80%EA%B4%80%EA%B3%84-%EA%B4%80%EB%A6%AC)

### 💡 상속관계 매핑

- 관계형 데이터베이스는 상속 관계가 없음
- 슈퍼타입 서브타입 관계 모델링 기법이 객체 상속과 유사
- 상속관계 매핑 : 객체의 상속과 구조와 DB의 슈퍼타입 서브타입 관계를 매핑
- 슈퍼타입 서브타입 논리 모델을 실제 물리 모델로 구현하는 방법
    - 조인 전략 : 각각 테이블로 변환
    - 단일 테이블 전략 : 통합 테이블로 변환
    - 구현 클래스마다 테이블 전략 : 서브타입 테이블로 변환
- dtype은 운영상 항상 있는 게 좋음
- 주요 어노테이션

    ```java
    @Inheritance(strategy = InheritanceType.JOINED) // 조인 전략
    @Inheritance(strategy = InheritanceType.SINGLE_TABLE) // 단일 테이블 전략
    @Inheritance(strategy = InheritanceType.TABLE_PER_CLASS) // 구현 클래스마다 테이블 전략
    @DiscriminatorColumn(name=“DTYPE”) // 어떤 서브클래스인지 구분하기 위해 사용하는 컬럼을 지정
    @DiscriminatorValue(“A”) // 해당 서브클래스가 어떤 타입 문자열로 저장될지 정의, 부모 테이블의 DTYPE 컬럼에 저장될 값
    ```

- 예시 코드

    ```java
    @Entity
    @Inheritance(strategy = InheritanceType.SINGLE_TABLE)
    @DiscriminatorColumn
    public abstract class Item { // Item 을 직접 만들지 않을거면 abstract
        @Id @GeneratedValue
        @Column(name = "ITEM_ID")
        private Long id;
    
        private String name;
        private int price;
        private int stockQuantity;
    }
    
    @Entity
    @DiscriminatorValue("B")
    public class Book extends Item { }
    ```


---

### 💡 상속관계 매핑 전략 장단점

- 조인 전략 : 정석 느낌
    - 장점 : 테이블 정규화, 저장공간 효율화
    - 단점 : 조회 시 조인을 많이 사용, 성능 저하
- 단일 테이블 전략
    - 장점 : 조인이 필요 없으므로 일반적으로 조회 성능이 빠름, 조회 쿼리 단순
    - 단점 : 자식 엔티티가 매핑한 컬럼은 모두 null 허용, 단일 테이블에 모든 것 저장하므로 테이블 커져서 오히려 조회 성능이 느려질 수 있음
- 구현 클래스마다 테이블 전략
    - 데이터베이스 설계자와 ORM 전문가 둘 다 추천 안 함
    - 단점 : 여러 자식 테이블을 함께 조회할 때 성능이 느림(UNION SQL 필요)
- 조인 전략을 기본으로 하고 단순한 테이블은 단일 테이블 고려해보는 걸 권장

---

### 💡 @MappedSuperclass

- 공통 매핑 정보가 필요할 때 사용
- ex) 등록일, 등록자 등
- 상속관계 매핑 아님
- 엔티티 아니고 테이블과 매핑되지 않음 → 단순히 엔티티가 공통으로 사용하는 매핑 정보를 모으는 역할
- 부모 클래스를 상속 받는 자식 클래스에 매핑 정보만 제공
- 부모 타입으로 조회 불가
- 추상 클래스 권장
- 예시 코드

    ```java
    @MappedSuperclass // 공통 매핑 정보 사용할 때
    public abstract class BaseEntity {
        private String createdBy;
        private LocalDateTime createdDate;
        private String lastModifiedBy;
        private LocalDateTime lastModifiedDate;
    }
    
    // Member
    public class Member extends BaseEntity{ }
    ```


---

### 💡 새롭게 알게 된 점 및 중요한 점 정리

1. 관계형 데이터베이스는 상속 관계가 없음
2. 객체의 상속과 구조와 DB의 슈퍼타입 서브타입 관계를 매핑해야함
3. 슈퍼타입 서브타입 논리 모델을 실제 물리 모델로 구현하는 방법에는 조인 전략, 단일 테이블 전략, 구현 클래스마다 테이블 전략이 있음
4. 조인 전략을 기본으로 하고 단순한 테이블은 단일 테이블 고려해보기, 구현 클래스마다 테이블 전략은 고려하지 않는 걸 권장
5. @MappedSuperclass는 공통 매핑 정보가 필요할 때 사용함