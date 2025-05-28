[[[실전! 스프링 부트와 JPA 활용1 - 웹 애플리케이션 개발] 섹션6. 상품 도메인 개발 / 섹션7. 주문 도메인 개발]](https://velog.io/@sehyun56/%EC%8B%A4%EC%A0%84-%EC%8A%A4%ED%94%84%EB%A7%81-%EB%B6%80%ED%8A%B8%EC%99%80-JPA-%ED%99%9C%EC%9A%A91-%EC%9B%B9-%EC%95%A0%ED%94%8C%EB%A6%AC%EC%BC%80%EC%9D%B4%EC%85%98-%EA%B0%9C%EB%B0%9C-%EC%84%B9%EC%85%986.-%EC%83%81%ED%92%88-%EB%8F%84%EB%A9%94%EC%9D%B8-%EA%B0%9C%EB%B0%9C-%EC%84%B9%EC%85%987.-%EC%A3%BC%EB%AC%B8-%EB%8F%84%EB%A9%94%EC%9D%B8-%EA%B0%9C%EB%B0%9C)

### 💡 주문 개발

- 도메인 모델 패턴
    - 주문 서비스의 주문과 주문 취소 메서드를 보면 비즈니스 로직 대부분이 엔티티에 있음
    - 서비스 계층은 엔티티에 필요한 요청을 위임하는 역할을 함
    - 엔티티가 비즈니스 로직을 가짐

    ```java
    // OrderItem 클래스
    
    //==생성 메서드==//
    public static OrderItem createOrderItem(Item item, int orderPrice, int
    count) {
         OrderItem orderItem = new OrderItem();
        orderItem.setItem(item);
        orderItem.setOrderPrice(orderPrice);
        orderItem.setCount(count);
        item.removeStock(count);
        return orderItem;
    }
    //==비즈니스 로직==//
    /** 주문 취소 */
    public void cancel() {
        getItem().addStock(count);
    }
    //==조회 로직==//
    /** 주문상품 전체 가격 조회 */ 
    public int getTotalPrice() {
        return getOrderPrice() * getCount();
    }
    ```

    ```java
    // Order 클래스
    
    //==생성 메서드==//
    public static Order createOrder(Member member, Delivery delivery,
    OrderItem... orderItems) {
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
    //==비즈니스 로직==//
    /** 주문 취소 */
    public void cancel() {
    	if (delivery.getStatus() == DeliveryStatus.COMP) {
    		throw new IllegalStateException("이미 배송완료된 상품은 취소가 불가능합니다.");
    	}
      this.setStatus(OrderStatus.CANCEL);
      for (OrderItem orderItem : orderItems) {
          orderItem.cancel();
      }
    }
    //==조회 로직==//
    /** 전체 주문 가격 조회 */
    public int getTotalPrice() {
        int totalPrice = 0;
        for (OrderItem orderItem : orderItems) {
            totalPrice += orderItem.getTotalPrice();
        }
        return totalPrice;
    }
    
    ```

- 트랜잭션 스크립트 패턴
    - 엔티티에는 비즈니스 로직이 거의 없고 서비스 계층에서 대부분의 비즈니스 로직 처리
- @NoArgsConstructor(access = AccessLevel.*PROTECTED*)
    - 클래스에 이 어노테이션을 붙여놓으면 외부에서 기본 생성자로 객체를 생성하지 못하게 하여 이 클래스의 생성자를 사용할 수 없으므로 내가 만든 메소드를 사용해서 객체를 생성하도록 할 수 있음

---

### 💡  주문 검색 기능 개발

- JPA에서 동적 쿼리
- OrderSearch

    ```java
    @Getter @Setter
    public class OrderSearch {
        private String memberName;
        private OrderStatus orderStatus;
    }
    ```

- OrderRepository - findAllByString
    - JPQL 쿼리를 문자로 생성하기는 번거로움
    - 실수로 인한 버그가 발생 가능성 높음

    ```java
    public List<Order> findAllByString(OrderSearch orderSearch) {
      String jpql = "select o From Order o join o.member m";
      boolean isFirstCondition = true;
      //주문 상태 검색
      if (orderSearch.getOrderStatus() != null) {
          if (isFirstCondition) {
              jpql += " where";
              isFirstCondition = false;
          } else {
              jpql += " and";
          }
          jpql += " o.status = :status";
      }
      //회원 이름 검색
      if (StringUtils.hasText(orderSearch.getMemberName())) {
          if (isFirstCondition) {
              jpql += " where";
              isFirstCondition = false;
    
          } else {
              jpql += " and";
          }
          jpql += " m.name like :name";
      }
      TypedQuery<Order> query = em.createQuery(jpql, Order.class) .setMaxResults(1000); //최대 1000건
      if (orderSearch.getOrderStatus() != null) {
          query = query.setParameter("status", orderSearch.getOrderStatus());
      }
      if (StringUtils.hasText(orderSearch.getMemberName())) {
          query = query.setParameter("name", orderSearch.getMemberName());
      }
      return query.getResultList();
    }
    ```

- OrderRepository - findAllByCriteria
    - JPA Criteria 사용
    - JPA 표준 스펙이지만 실무에서 사용하기에는 복잡함

    ```java
    public List<Order> findAllByCriteria(OrderSearch orderSearch) {
        // JPQL을 자바 코드로 할 수 있게 제공해주는 거
        CriteriaBuilder cb = em.getCriteriaBuilder();
        CriteriaQuery<Order> cq = cb.createQuery(Order.class);
        Root<Order> o = cq.from(Order.class);
        Join<Order, Member> m = o.join("member", JoinType.INNER); //회원과 조인
        List<Predicate> criteria = new ArrayList<>();
        //주문 상태 검색
        if (orderSearch.getOrderStatus() != null) {
            Predicate status = cb.equal(o.get("status"),
                    orderSearch.getOrderStatus());
            criteria.add(status);
        }
        //회원 이름 검색
        if (StringUtils.hasText(orderSearch.getMemberName())) {
            Predicate name =
                    cb.like(m.<String>get("name"), "%" +
    
                            orderSearch.getMemberName() + "%");
            criteria.add(name);
        }
        cq.where(cb.and(criteria.toArray(new Predicate[criteria.size()])));
        TypedQuery<Order> query = em.createQuery(cq).setMaxResults(1000); //최대 1000건
        return query.getResultList();
    }
    ```

- 이러한 동적 쿼리 구현의 문제점을 Querydsl 로 해결 가능

---

### 💡 새롭게 알게 된 점 및 중요한 점 정리

1. 도메인 모델 패턴 vs 트랜잭션 스크립트 패턴
    - 도메인 모델 패턴 : 엔티티가 비즈니스 로직을 가짐, 서비스 계층은 엔티티에 필요한 역할을 위임하는 역할
    - 트랜잭션 스크립트 패턴 : 서비스 계층에서 대부분의 비즈니스 로직 처리
2. @NoArgsConstructor(access = AccessLevel.*PROTECTED*)
    - 파라미터가 없는 기본 생성자의 접근을 제한하여 기본 생성자의 무분별한 생성을 막아 의도하지 않은 엔티티를 만드는 것을 막을 수 있음
    - JPA는 지연 로딩 시 프록시 객체를 생성하는데 이때 프록시 객체를 생성하는데 기본 생성자가 필요하며 최소한 protected 접근 제한자여야 함 → 외부에서 직접 객체 생성을 막으면서도 JPA는 내부적으로 객체를 생성할 수 있음
    - @NoArgsConstructor(access = AccessLevel.*PROTECTED*)를 클래스에 붙이고 생성자 위에 @Builder를 붙이면 기본 생성자를 만드는 것을 막고 원하는 형식으로 생성자를 만들 수 있음
3. 동적 쿼리
    - 사용자가 입력한 조건에 따라 SQL의 where 절이나 join 등이 동적으로 바뀌는 쿼리
4. Querydsl
    - 자바 코드로 쿼리를 작성해 컴파일 시점에 오류를 쉽게 발견할 수 있음
    - 복잡한 동적 쿼리를 쉽게 다룰 수 있음