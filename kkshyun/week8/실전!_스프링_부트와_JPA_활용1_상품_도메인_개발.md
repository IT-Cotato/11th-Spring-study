[[[실전! 스프링 부트와 JPA 활용1 - 웹 애플리케이션 개발] 섹션6. 상품 도메인 개발 / 섹션7. 주문 도메인 개발]](https://velog.io/@sehyun56/%EC%8B%A4%EC%A0%84-%EC%8A%A4%ED%94%84%EB%A7%81-%EB%B6%80%ED%8A%B8%EC%99%80-JPA-%ED%99%9C%EC%9A%A91-%EC%9B%B9-%EC%95%A0%ED%94%8C%EB%A6%AC%EC%BC%80%EC%9D%B4%EC%85%98-%EA%B0%9C%EB%B0%9C-%EC%84%B9%EC%85%986.-%EC%83%81%ED%92%88-%EB%8F%84%EB%A9%94%EC%9D%B8-%EA%B0%9C%EB%B0%9C-%EC%84%B9%EC%85%987.-%EC%A3%BC%EB%AC%B8-%EB%8F%84%EB%A9%94%EC%9D%B8-%EA%B0%9C%EB%B0%9C)

### 상품 개발

- Item
    - Item에 비즈니스 로직 추가
    - 재고 감소시키는 비즈니스 로직에서는 원래 있는 재고보다 많은 양을 감소시키면 NotEnoughStockException 예외를 터트림

    ```java
    // 비즈니스 로직
    // 재고 증가
    public void addStock(int quantity) {
        this.stockQuantity += quantity;
    }
    
    // 재고 감소
    public void removeStock(int quantity) {
        int restStock = this.stockQuantity - quantity;
        if(restStock < 0) {
            throw new NotEnoughStockException("need more stock");
        }
        this.stockQuantity = restStock;
    }
    ```

    ```java
    public class NotEnoughStockException extends RuntimeException{
    // Generate > Override methods
        public NotEnoughStockException() {
            super();
        }
    
        public NotEnoughStockException(String message) {
            super(message);
        }
    
        public NotEnoughStockException(String message, Throwable cause) {
            super(message, cause);
        }
    
        public NotEnoughStockException(Throwable cause) {
            super(cause);
        }
    }
    ```

- ItemRepository
    - 이미 Item이 저장되어 있지 않으면(ID 값이 없으면) 새로 저장
    - ID 값이 존재하면 업데이트

    ```java
    @Repository
    @RequiredArgsConstructor
    public class ItemRepository {
        private final EntityManager em;
    
        // Item은 JPA 저장하기 전까지 ID 값이 없으면 새로 저장
        // ID 값이 있으면 이미 DB에 있으므로 업데이트
        public void save(Item item) {
            if(item.getId() == null) {
                em.persist(item);
            } else {
                em.merge(item); // 업데이트랑 비슷함
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

- ItemService
    - 클래스에 @Transactional(readOnly = true) 붙이고 saveItem 메소드에만 @Transactional 붙여서 성능 최적화시킴
    ```java
    @Service
    @Transactional(readOnly = true)
    @RequiredArgsConstructor
    public class ItemService {
    private final ItemRepository itemRepository;
    @Transactional
    public void saveItem(Item item) {
    itemRepository.save(item);
    }
    
        public List<Item> findItem() {
            return itemRepository.findAll();
        }
    
        public Item findOne(Long itemId) {
            return itemRepository.findOne(itemId);
        }
    }
    ```