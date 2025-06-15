[[[실전! 스프링 부트와 JPA 활용1 - 웹 애플리케이션 개발] 섹션8. 웹 계층 개발]](https://velog.io/@sehyun56/%EC%8B%A4%EC%A0%84-%EC%8A%A4%ED%94%84%EB%A7%81-%EB%B6%80%ED%8A%B8%EC%99%80-JPA-%ED%99%9C%EC%9A%A91-%EC%9B%B9-%EC%95%A0%ED%94%8C%EB%A6%AC%EC%BC%80%EC%9D%B4%EC%85%98-%EA%B0%9C%EB%B0%9C-%EC%84%B9%EC%85%988.-%EC%9B%B9-%EA%B3%84%EC%B8%B5-%EA%B0%9C%EB%B0%9C)

## ⭐️ 섹션 8 웹 계층 개발

### 💡 **홈 화면과 레이아웃**

- @Slf4j
    - 로그를 찍기 위한 롬복의 어노테이션

    ```java
    log.info("home controller");
    ```


---

### 💡 회원 개발

- 회원 등록 폼 객체
    - 폼 객체를 사용해 화면 계층과 서비스 계층 분리

    ```java
    @Getter @Setter
    public class MemberForm {
    		// 값이 비어있으면 오류 남
        @NotEmpty(message = "회원 이름은 필수 입니다") 
        private String name;
    
        private String city;
        private String street;
        private String zipcode;
    }
    ```

- 회원 컨트롤러
    - Get 요청이 들어오면 model 에 데이터를 담아 뷰 이름을 반환하고 뷰는 화면을 그려서 줌
    - Post 요청이 들어오면 받은 객체를 꺼내서 로직을 수행하고 페이지 이동
    - @Valid 를 붙이면 NotEmpty 등 붙였던 어노테이션을 기반으로 간단하게 validation 을 해줌
    - validation 실패시 BindingResult에 결과가 담아져 있기 때문에 동일한 폼 화면을 다시 보여줄 수 있음

    ```java
    @Controller
    @RequiredArgsConstructor
    public class MemberController {
        private final MemberService memberService;
    
        @GetMapping("/members/new")
        public String createForm(Model model) {
            model.addAttribute("memberForm", new MemberForm());
            return "members/createMemberForm";
        }
    
        @PostMapping("/members/new")
        // NotEmpty 등 어노테이션을 기반으로 간단하게 validation 해줌
        public String create(@Valid MemberForm form, BindingResult result) { 
            if(result.hasErrors()) {
                return "members/createMemberForm";
            }
    
            Address address = new Address(form.getCity(), form.getStreet(), form.getZipcode());
            Member member = new Member();
            member.setName(form.getName());
            member.setAddress(address);
    
            memberService.join(member);
            return "redirect:/"; // 첫번째 페이지로 넘어감
        }
    
        @GetMapping("/members")
        public String list(Model model) {
            List<Member> members = memberService.findMembers();
            model.addAttribute("members", members);
            return "members/memberList";
        }
    }
    ```

- 폼 객체 vs 엔티티 직접 사용
    - 실무는 복잡하기 때문에 엔티티를 직접 사용하기 보다는 화면이나 API에 맞는 폼 객체나 DTO를 사용해 엔티티를 최대한 순수하게 유지시키기는 것이 좋음

---

### 💡 상품 개발

- 상품 controller
    - 컨트롤러에서 파라미터로 넘어온 item 엔티티 인스턴스는 준영속 상태로 데이터를 수정해도 변경 감지 기능이 동작하지 않음

    ```java
    @Controller
    @RequiredArgsConstructor
    public class ItemController {
    
        private final ItemService itemService;
    
        @GetMapping("/items/new")
        public String createForm(Model model) {
            model.addAttribute("form", new BookForm());
            return "items/createItemForm";
        }
    
        @PostMapping("items/new")
        public String create(BookForm form) {
            Book book = new Book();
            book.setName(form.getName());
            book.setPrice(form.getPrice());
            book.setStockQuantity(form.getStockQuantity());
            book.setAuthor(form.getAuthor());
            book.setIsbn(form.getIsbn());
    
            itemService.saveItem(book);
            return "redirect:/";
        }
    
        @GetMapping("/items")
        public String list(Model model) {
            List<Item> items = itemService.findItems();
            model.addAttribute("items",items);
            return "items/itemList";
        }
    
        @GetMapping("items/{itemId}/edit")
        public String updateItemForm(@PathVariable("itemId") Long itemId, Model model) {
            Book item = (Book) itemService.findOne(itemId);
            BookForm form = new BookForm();
            form.setId(item.getId());
            form.setName(item.getName());
            form.setPrice(item.getPrice());
            form.setStockQuantity(item.getStockQuantity());
            form.setAuthor(item.getAuthor());
            form.setIsbn(item.getIsbn());
            model.addAttribute("form", form);
            return "items/updateItemForm";
    
        }
    
        @PostMapping("items/{itemId}/edit")
        public String updateItem(@PathVariable("itemId") Long itemId, @ModelAttribute("form") BookForm form) {
    
    //        Book book = new Book();
    //        book.setId(form.getId());
    //        book.setName(form.getName());
    //        book.setPrice(form.getPrice());
    //        book.setStockQuantity(form.getStockQuantity());
    //        book.setAuthor(form.getAuthor());
    //        book.setIsbn(form.getIsbn());
    //        itemService.saveItem(book);
    
            itemService.updateItem(itemId, form.getName(), form.getPrice(), form.getStockQuantity());
            return "redirect:/items";
        }
    }
    ```

- 변경 감지와 병합
    - 준영속 엔티티
        - 영속성 컨텍스트가 더는 관리하지 않는 엔티티
        - 임의로 만들어낸 엔티티도 기존 식별자를 가지고 있으면 준영속 엔티티로 볼 수 있음
        - 준영속 엔티티를 수정하는 방법 → 변경 감지 기능 / 병합
    - 변경 감지
        - 트랜잭션 커밋 시점에 변경 감지(Dirty Checking)이 동작해서 데이터베이스에 UPDATE SQL 실행

        ```java
        @Transactional
        void update(Item itemParam) {
        	Item findItem = em.find(Item.class, itemParam.getId());
        	findItem.setPrice(itemParam.getPrice());
        }
        ```

    - 병합
        - 준영속 상태의 엔티티를 영속 상태로 변경할 때 사용
        - 영속 엔티티의 값을 준영속 엔티티의 값으로 모두 교체

        ```java
        @Transactional
        void update(Item itemParam) {
            Item mergeItem = em.merge(itemParam);
        }
        ```

    - 해결 방법
        - 변경 감지 기능을 사용하면 원하는 속성만 선택해서 변경할 수 있지만 병합을 사용하면 모든 속성이 변경됨, 병합은 값이 없으면 null로 업데이트 가능 → 엔티티를 변경할 때는 병합보다 변경 감지를 사용
        - 트랜잭션이 있는 서비스 계층에 식별자(id)와 변경할 데이터를 전달(파라미터 혹은 DTO)
        - 트랜잭션이 있는 서비스 계층에서 영속 상태의 엔티티를 조회하고 엔티티의 데이터를 직접 변경
        - 트랜잭션 커밋 시점에 변경 감지가 실행

        ```java
        // controller
        @PostMapping(value = "/items/{itemId}/edit")
        public String updateItem(@PathVariable Long itemId, @ModelAttribute("form") BookForm form) {
            itemService.updateItem(itemId, form.getName(), form.getPrice(), form.getStockQuantity());
            return "redirect:/items";
        }
        
        // service
        @Transactional
        public void updateItem(Long id, String name, int price, int stockQuantity)
        {
          Item item = itemRepository.findOne(id);
          item.setName(name);
          item.setPrice(price);
          item.setStockQuantity(stockQuantity);
        }
        ```


---

### 💡 상품 주문 개발

- 상품 주문 controller
    - 넘겨줄 때 식별자만 넘겨주는 것이 좋음 → transaction 어노테이션 없이 객체를 주면 JPA와 관계없는 게 넘어가기 때문에 변경 감지가 애매해짐

    ```java
    @Controller
    @RequiredArgsConstructor
    public class OrderController {
    
        private final OrderService orderService;
        private final MemberService memberService;
        private final ItemService itemService;
    
        @GetMapping("/order")
        public String createForm(Model model) {
            List<Member> members = memberService.findMembers();
            List<Item> items = itemService.findItems();
            model.addAttribute("members", members);
            model.addAttribute("items", items);
    
            return "order/orderForm";
        }
    
        @PostMapping("/order")
        public String order(@RequestParam("memberId") Long memberId,
                            @RequestParam("itemId") Long itemId,
                            @RequestParam("count") int count){
    
            // 식별자만 넘겨주기, transaction 없이 객체를 주면 JPA와 관계없는 게 넘어가기 때문에 변경 감지가 애매해짐
            orderService.order(memberId, itemId, count);
            return "redirect:/orders";
        }
    
        @GetMapping("/orders")
        public String orderList(@ModelAttribute("orderSearch") OrderSearch orderSearch, Model model) {
            List<Order> orders = orderService.findOrders(orderSearch);
            model.addAttribute("orders", orders);
    
            return "order/orderList";
        }
    
        @PostMapping("/orders/{orderId}/cancel")
        public String cancelOrder(@PathVariable("orderId") Long orderId) {
            orderService.cancelOrder(orderId);
            return "redirect:/orders";
        }
    }
    
    ```


---

### 💡 새롭게 알게 된 점 및 중요한 점 정리

1. 서버 사이드 렌더링(SSR) vs 클라이언트/서버 분리형 아키텍처(CSR)
    - SSR
        - 화면을 서버에서 만들어서 클라이언트에게 보내주는 방식
        - ex) Spring Boot + Thymeleaf 사용
    - CSR
        - 서버는 JSON만 주고 클라이언트가 그걸 받아서 화면을 만듦
        - ex) Spring Boot + React 사용
2. @Valid
    - 파라미터로 @RequestBody 어노테이션 옆에 @Valid를 작성하면 RequestBody로 들어오는 객체에 대한 validation을 수행
3. 변경 감지와 병합
    - 준영속 엔티티를 수정하는 방법은 2가지
        - 변경 감지 : 트랜잭션 커밋 시점에 변경 감지(Dirty Checking)이 동작해서 데이터베이스에 UPDATE SQL 실행
        - 병합 : 영속 엔티티의 값을 준영속 엔티티의 값으로 모두 교체
    - 병합은 전체 값을 교체하기 때문에 값을 주지 않는 필드에 null 값이 들어갈 수 있고 실무에서 전체 값을 교체할 일은 많지 않기 때문에 부분적으로 값을 교체할 수 있는 변경 감지를 사용