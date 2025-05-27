## 1. 상품(Item) 엔티티

---
### 🔹 비즈니스 메서드

```java
// Item.java
//==비즈니스 로직==//
public void addStock(int quantity) {
    this.stockQuantity += quantity;
}

public void removeStock(int quantity) {
    int restStock = this.stockQuantity - quantity;
    if (restStock < 0) {
        throw new NotEnoughStockException("need more stock");
    }
    this.stockQuantity = restStock;
}
```

- 엔티티가 자신의 데이터를 직접 관리하는 객체지향적 설계
- 데이터와 비즈니스 로직을 한 곳에 모아 응집도를 높임

## 2. 상품(Item) 리포지토리

---
```java
@Repository
@RequiredArgsConstructor
public class ItemRepository {

    private final EntityManager em;

    public void save(Item item) {
        if (item.getId() == null) {
            em.persist(item);  // 신규 엔티티 영속화
        } else {
            em.merge(item);    // 기존 엔티티 업데이트
        }
    }

    public Item findOne(Long id) {
        return em.find(Item.class, id);
    }

    public List<Item> findAll() {
        return em.createQuery("select i from Item i", Item.class)
                 .getResultList();
    }
}
```

- save()
    - id가 없으면 새 객체 → em.persist()
    - id가 있으면 기존 객체 수정 → merge() 실행

## 3. 주문(Order) 엔티티

---

### 🔹 생성 메서드 (Factory Method)

- 주문 엔티티는 OrderItem, Delivery 등 복잡한 연관관계를 가진다
- 복잡한 연관관계 초기화를 위해 생성 메서드(Factory Method) 패턴 사용

```java
//==생성 메서드==//
public static Order createOrder(Member member, Delivery delivery, OrderItem... orderItems) {
	
	Order order = new Order();
	
	order.setMember(member);
	order.setDelivery(delivery);
	for (OrderItem orderItem : orderItems) {
		order.addOrderItem(orderItem);
	}
	order.setStatus(OrderStatus.ORDER);
	order.setOrderDate(LocalDateTime.now());
	
	return order;
}
```

- 생성 메서드를 통해 연관관계를 한 번에 설정한다
- 생성 로직 변경 시 이 메서드만 수정하면 된다

### 🔹 기본 생성자 접근 제한

```java
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class Order {...}
```

- 위에서 만든 생성 메서드 (createOrder)를 사용하지 않고 외부에서 직접 생성하는 것을 방지하기 위해 기본 생성자를 protected로 제한한다
- 롬복 어노테이션을 사용해 간결하게 표현 가능

### 🔹 비즈니스 로직 - 주문 취소

```java
//==비즈니스 로직==//

public void cancel() {
	if(delivery.getStatus() == DeliveryStatus.COMP) {
		throw new IllegalStateException("이미 배송완료된 상품은 취소가 불가능합니다.");
	}

	this.setStatus(OrderStatus.CANCEL);
	for(OrderItem orderItem : orderItems) {
		orderItem.cancel();
	}
}
```

- 배송 완료 시 주문 취소 불가
- 주문 상태 변경과 함께 각 주문 상품의 취소 로직도 실행

## 4. 주문 상품(OrderItem) 엔티티

---
### 🔹 생성 메서드 (Factory Method)

```java
//==생성 메서드==//
public static OrderItem createOrderItem(Item item, int orderPrice, int count) {
	OrderItem orderItem = new OrderItem();
	orderItem.setItem(item);
	orderItem.setOrderPrice(orderPrice);
	orderItem.setCount(count);

	item.removeStock(count); // 주문 수량만큼 재고 차감
	return orderItem;
}
```

### 🔹 기본 생성자 접근 제한

```java
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class OrderItem {}
```

- OrderItem은 createOrderItem 생성 메서드를 통해서만 생성하기 위해 기본 생성자를 protected로 제한

### 🔹 비즈니스 로직

```java
//==비즈니스 로직==//
public void cancel() {
	getItem().addStock(count); // 주문 취소 시 재고 복구
}

public int getTotalPrice() {
	return getOrderPrice() * getCount();
}
```

- 주문 생성 시 재고 차감 처리
- 주문 취소 시 각 상품에 대해 재고 복구 처리

## 5. 주문(Order) 서비스

---
### 🔹 상품 주문

```java
@Service
@Transactional(readOnly = true)
@RequiredArgsConstructor
public class OrderService {

	private final OrderRepository orderRepository;
	private final MemberRepository memberRepository;
	private final ItemRepository itemRepository;

	@Transactional
	public Long order(Long memberId, Long itemId, int count) {
		// 엔티티 조회
		Member member = memberRepository.findOne(memberId);
		Item item = itemRepository.findOne(itemId);

		// 배송 정보 생성
		Delivery delivery = new Delivery();
		delivery.setAddress(member.getAddress());

		// 주문상품 생성
		OrderItem orderItem = OrderItem.createOrderItem(item, item.getPrice(), count);

		// 주문 생성
		Order order = Order.createOrder(member, delivery, orderItem);

		// 주문 저장
		orderRepository.save(order);
		return order.getId();
	}
}
```

Delivery와 OrderItem은 별도의 save() 호출 없이 Order 저장 시 함께 영속화 된다

```java
@OneToMany(mappedBy = "order", **cascade = CascadeType.ALL**)
private List<OrderItem> orderItems = new ArrayList<>();

@OneToOne(fetch = FetchType.LAZY, **cascade = CascadeType.ALL**)
@JoinColumn(name = "delivery_id")
private Delivery delivery;
```

- cascade = CascadeType.ALL 옵션을 통해 Order를 저장할 때 연관된 OrderItem, Delivery도 함께 영속성 컨텍스트에 저장 및 삭제된다.
- Order가 OrderItem, Delivery의 생명주기를 관리하는 비즈니스 주인 역할을 한다


> ### 📌 JPA 연관관계 주인 vs 도메인 모델 비즈니스 주인
> 
> **JPA에서 말하는 연관관계 주인(Owner)**
> 
> - 외래키(FK)를 실제로 관리하는 쪽
> - 양방향 연관관계 시 mappedBy가 없는 쪽
> - 여기서는 OrderItem이 Order의 ID를 외래 키로 가지고 있기 때문에 OrderItem이 연관관계 주인이고, Order가 Delivery의 ID를 가지고 있어서 Order가 연관관계의 주인인 상태
> 
> **비즈니스에서 말하는 주인**
> 
> - 도메인 설계 측면에서 객체 생명주기를 책임지는 주체를 의미
> - Order가 Delivery, OrderItem을 관리하며, 이 둘은 Order가 없이는 의미가 없는 종속 객체이다
> - Order가 삭제될 때 Delivery, OrderItem도 함께 삭제하는 등 비즈니스 생명주기까지 관리한다


> ### 📌 cascade 범위 설정에 대한 고민
> 
> Delivery와 OrderItem은 **Order 외에는 참조하지 않는 종속 객체(Private Owner)이다**
> 
> - 즉, `Order`가 이들의 생명주기를 온전히 관리하는 주체이며, 다른 곳에서 사용하지 않는다
> - 이런 경우에는 cascade 옵션을 적극 사용하는 것이 유지보수성과 일관성 측면에서 좋다
> 
> 반대로, 만약 Delivery나 OrderItem이 여러 다른 엔티티에서 공유되거나 참조된다면
> 
> - cascade 옵션을 무분별하게 사용하는 것은 위험하다
> - cascade 범위를 좁히거나 직접 관리하는 방법을 고민해야 한다
> 
> 잘못된 cascade 사용은 의도치 않은 삭제나 영속성 전파를 일으킬 수 있어서, 참조 범위와 생명주기 소유권을 명확히 하는 것이 중요하다  
>
> | **CascadeType** | **설명** |
> | --- | --- |
> | **ALL** | 모든 작업을 전파(PERSIST, MERGE, REMOVE, REFRESH, DETACH 모두 포함) |
> | **PERSIST** | 엔티티 저장 시 연관 엔티티도 함께 저장 |
> | **MERGE** | 엔티티 병합(업데이트) 시 연관 엔티티도 병합 |
> | **REMOVE** | 엔티티 삭제 시 연관 엔티티도 함께 삭제 |
> | **REFRESH** | 엔티티 새로고침 시 연관 엔티티도 새로고침 |
> | **DETACH** | 영속성 컨텍스트에서 분리 시 연관 엔티티도 함께 분리 | 

### 🔹 주문 취소

```java
public class OrderService {
  ...
  
	@Transactional
	public void cancelOrder(Long orderId) {
		Order order = orderRepository.findOne(orderId);
		order.cancel();
	}
}
```

**코드가 간결한 이유**

- order.cancel() 호출로 주문 상태 변경과 Item의 재고 변경이 엔티티 내부에서 처리된다
- 별도로 save나 update를 호출하지 않아도 된다
- 이 방식이 가능한 이유는 Order과 관련된 모든 상태 변경이 Order 엔티티 내부 메서드에서 일어나기 때문

> ### 📌 JPA 변경 감지(Dirty Checking)
> 
> - 트랜잭션이 끝나는 시점에 JPA가 영속성 컨텍스트 내의 엔티티 상태 변화를 자동으로 감지한다
> - 변경된 엔티티에 대해 필요한 UPDATE SQL 쿼리를 자동으로 생성해 DB에 반영한다
> - 개발자는 상태 변경 후 별도의 update 호출 없이 트랜잭션 커밋 시점에 변경 내용이 반영된다
>    - 주문 취소 시 Order 상태 변경과 Item 재고 변경이 모두 반영된다


> ### 📌 도메인 모델 패턴 vs 트랜잭션 스크립트 패턴
> 
> | **구분** | **도메인 모델 패턴** | **트랜잭션 스크립트 패턴** |
> | --- | --- | --- |
> | **비즈니스 로직 위치** | 엔티티 내부에 비즈니스 로직 포함 | 서비스(트랜잭션) 계층에서 비즈니스 로직 처리 |
> | **설계 스타일** | 객체지향적, 응집도 높고 유지보수성 우수 | 절차적, 단순하고 빠른 구현 가능 |
> | **장점** | 코드가 더 직관적이고 응집도가 높음 | 구현이 간단하고 빠름 |
> | **단점** | 설계가 복잡할 수 있고 학습 비용이 있음 | 비즈니스 로직이 여러 곳에 흩어질 수 있음 |
> - Order, OrderItem 엔티티 내부에 비즈니스 메서드(cancel(), removeStock() 등)를 구현해서 도메인 모델 패턴을 따르고 있다
> - 객체지향적 설계로 엔티티가 자신의 상태를 스스로 관리하게 하여 응집도와 유지보수성을 높이는 방향이다
> - 서비스 계층에서 모든 비즈니스 로직을 처리하는 설계는 트랜잭션 스크립트 패턴이라고 한다
> - 복잡한 도메인 로직이 많아질수록 도메인 모델 패턴이 효과적이다
