[[실전! 스프링 부트와 JPA 활용1 - 웹 애플리케이션 개발]섹션2. 프로젝트 환경설정](https://velog.io/@sehyun56/%EC%8B%A4%EC%A0%84-%EC%8A%A4%ED%94%84%EB%A7%81-%EB%B6%80%ED%8A%B8%EC%99%80-JPA-%ED%99%9C%EC%9A%A91-%EC%9B%B9-%EC%95%A0%ED%94%8C%EB%A6%AC%EC%BC%80%EC%9D%B4%EC%85%98-%EA%B0%9C%EB%B0%9C-%EC%84%B9%EC%85%983.-%EB%8F%84%EB%A9%94%EC%9D%B8-%EB%B6%84%EC%84%9D-%EC%84%A4%EA%B3%84)

### 💡 도메인 모델과 테이블 설계

- 다대다관계는 쓰면 안됨 → 일대다 관계로 풀어내야함
- 일대다 관계에서 다쪽에 외래키가 존재함
    - 외래 키가 있는 주문을 연관관계의 주인으로 정하는 것이 좋음
        - 반대를 연관관계의 주인으로 정하면 관리하지 않는 외래 키 값이 업데이트 될 수 있으므로 관리와 유지보수가 어렵고 추가적으로 별도의 업데이트 쿼리가 발생하는 성능 문제도 있음
- 회원 엔티티 분석
    - 회원(Member), 주문(Order), 주문상품(OrderItem), 상품(Item), 배송(Delivery), 카테고리(Category), 주소(Address)로 구성됨
    - Order 테이블에 외래키 MEMBER_ID(FK), DELIVERY_ID(FK)가 들어있음

---

### 💡  엔티티 클래스 개발 1, 2

- 예제에서는 설명을 쉽게 하기 위해 엔티티 클래스에 Getter, Setter를 모두 열고, 최대한 단순하게 설계
- 실무에서는 가급적 Getter는 열어두고, Setter는 꼭 필요한 경우에만 사용하는 것을 추천
    - 엔티티를 변경할 때는 Setter 대신 변경 지점이 명확하도록 변경을 위한 비지니스 메서드를 별도로 제공하는 것이 좋음
- Enum 요소를 엔티티 컬럼에 쓸 때는 @Enumerated(EnumType.STRING) 붙여야 함
- 연관관계 매핑

    ```java
    // Member 클래스
    @OneToMany(mappedBy = "member") // order 테이블에 있는 member 필드에 의해서 매핑된거를 나타냄
    private List<Order> orders = new ArrayList<>();
    ```

    ```java
    // Order 클래스 - 연관관게의 주인
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name="member_id") // 외래키를 가짐
    private Member member;
    
    //== 연관관계 메서드 ==//
    public void setMember(Member member) {
        this.member = member;
        member.getOrders().add(this);
    }
    ```

- @Embedded, @Embeddable
    - 하나의 엔티티 안에서 재사용 가능한 객체(주소, 좌표 등)를 묶어서 관리할 때 사용
    - 값 타입은 자체 테이블을 가지지 않고, 엔티티 테이블에 칼럼으로 생기게 됨

    ```java
    @Embeddable
    @Getter
    public class Address {
        private String city;
        private String street;
        private String zipcode;
    
    // 자바 기본 생성자를 public 또는 protected로 설정해야 함
        protected Address() {
        }
        public Address(String city, String street, String zipcode) {
            this.city = city;
            this.street = street;
            this.zipcode = zipcode;
        }
    }
    ```

    ```java
    @Entity
    @Getter @Setter
    public class Delivery {
        @Id @GeneratedValue
        @Column(name = "delivery_id")
        private Long id;
    
        @OneToOne(mappedBy = "delivery", fetch = FetchType.LAZY)
        private Order order;
    
        @Embedded
        private Address address;
    
        @Enumerated(EnumType.STRING)
        private DeliveryStatus status; // READY, COMP
    }
    ```

- @Inheritance, @DiscriminatorColumn, @DiscriminatorValue
    - JPA에서 객체의 상속 구조를 DB 테이블에 어떻게 매핑할지 결정할 때 사용

    ```java
    @Entity
    @Inheritance(strategy = InheritanceType.SINGLE_TABLE) // 한 테이블에 다 넣음
    @DiscriminatorColumn(name = "dtype") // dtype : SINGLE_TABLE 전략에서 자동 생성되는 구분 컬럼
    @Getter @Setter
    public abstract class Item {
    
        @Id
        @GeneratedValue
        @Column(name = "item_id")
        private Long id;
    
        private String name;
        private int price;
        private int stockQuantity;
    
        @ManyToMany(mappedBy = "items")
        private List<Category> categories = new ArrayList<>();
    }
    ```

    ```java
    @Entity
    @DiscriminatorValue("M")
    @Getter @Setter
    public class Movie extends Item {
        private String director;
        private String actor;
    }
    ```

- @ManyToMany 안 쓰는게 좋음
    - JoinTable로 만들면 컬럼을 추가할 수 없으므로 실무에서 사용하기에는 한계가 있음
    - 중간엔티티를 만들고 @ManyToOne, @OneToMany로 매핑해서 사용하는 것이 좋음

    ```java
    @ManyToMany
    @JoinTable(name = "category_item",
            joinColumns = @JoinColumn(name = "category_id"),
            inverseJoinColumns = @JoinColumn(name = "item_id")
    )
    private List<Item> items = new ArrayList<>();
    ```


---

### 💡 **엔티티 설계시 주의점**

- 유지보수가 어렵기 때문에 엔티티에는 가급적 Setter를 사용하지 않는게 좋음
- 모든 연관관계는 지연로딩(LAZY)으로 설정해야함
    - 즉시로딩(EAGER)는 예측이 어렵고 어떤 SQL이 실행될지 추적하기 어려움
    - 즉시로딩은 JPQL을 실행할 때 N+1 문제가 자주 발생함
    - 연관된 엔티티를 함께 DB에서 사용해야 하면, fetch join 또는 엔티티 그래프 기능을 사용해야함
    - OneToOne, ManyToOne 의 기본은 즉시로딩이므로 직접 지연로딩으로 설정해야함
        - @ManyToOne(fetch = FetchType.LAZY)
- 컬렉션은 필드에서 바로 초기화 하는 것이 안전함(best)
    - private List<Item> items = new ArrayList<>(); → null 문제 안 생김
- 엔티티(필드) → 테이블(컬럼)
    - 카멜케이스 → 언더스코어(orderPrice → order_price)
- cascade
- 연관관계 편의 메서드

    ```java
    // Order 클래스
    //== 연관관계 메서드 ==//
    public void setMember(Member member) {
        this.member = member;
        member.getOrders().add(this);
    }
    
    public void setDelivery(Delivery delivery) {
        this.delivery = delivery;
        delivery.setOrder(this);
    }
    ```


---

### 💡 새롭게 알게 된 점 및 중요한 점 정리

1. 연관관계 매핑
    - 연관관계 주인은 @ManyToOne 또는 @OneToOne + 지연로딩 설정(직접 명시) + 외래키 설정
    - 연관관계 비주인은 @OneToMany(mappedBy = “주인 필드명”)
    - 연관관계 편의 메서드를 써서 객체 간 양방향 연관관계 맞추기

    ```java
    // 연관관계 주인
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name="member_id")
    private Member member;
    
    // + 연관관계 메서드
    
    // 연관관계 비주인
    @OneToMany(mappedBy = "member")
    private List<Order> orders = new ArrayList<>();
    ```

2. OneToOne, ManyToOne 는 직접 지연로딩으로 설정해야함

    ```java
    @ManyToOne(fetch = FetchType.LAZY)
    ```

3. 연관관계 편의 메서드
    - 단순한 필드 변경이 아닌 연관관계 자체가 바뀌는 경우 객체 간 양방향 연관관계를 한 번에 맞춰줌
    - 연관관계 주인만 DB에 실제로 반영되므로 관계의 주체가 책임지고 양쪽 세팅
4. cascade
    - 부모가 자식 생명주기를 관리할 때 쓰임
    - 부모 엔티티의 생명주기를 자식에게 자동 전파