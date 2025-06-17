### 1. 회원가입

```java
@Getter
@Setter
public class MemberForm {

	@NotEmpty(message = "회원 이름은 필수입니다.")
	private String name;

	private String city;

	private String street;

	private String zipcode;
}
```


>💡**Validation 어노테이션 차이**
>
> ✅ **@NotNull**
>
> - null만 허용하지 않는다
> - “” (빈 문자열), “ “(공백 문자열)은 허용
>
> **✅ @NotEmpty**
>
> - null과 “”를 허용하지 않는다
> - 하지만 “ “ 은 허용한다
>
> **✅ @NotBlank**
>
> - null과 “”과 “ “  모두 허용하지 않는다
> - 셋 중 가장 validation의 강도가 높다

```java
@Controller
@RequiredArgsConstructor
public class MemberController {

	private final MemberService memberService;

	@GetMapping("/members/new")
	private String createForm(Model model) {
		model.addAttribute("memberForm", new MemberForm());
		return "members/createMemberForm";
	}
```

→ /members/new로 GET 요청 시, 빈 MemberForm을 뷰에 전달해 form을 초기화

```html
// members/createMemberForm
****
<div class="container">
    <div th:replace="~{fragments/bodyHeader :: bodyHeader}"/>
    <form role="form" action="/members/new" th:object="${memberForm}" method="post">
        <div class="form-group">
            <label th:for="name">이름</label>
            <input type="text" th:field="*{name}" class="form-control" placeholder="이름을 입력하세요"
                   th:class="${#fields.hasErrors('name')}? 'form-control fieldError' : 'form-control'">
            <p th:if="${#fields.hasErrors('name')}" th:errors="*{name}">Incorrect date</p>
        </div>
        <div class="form-group">
            <label th:for="city">도시</label>
            <input type="text" th:field="*{city}" class="form-control" placeholder="도시를 입력하세요">
        </div>
        <div class="form-group">
            <label th:for="street">거리</label>
            <input type="text" th:field="*{street}" class="form-control" placeholder="거리를 입력하세요">
        </div>
        <div class="form-group">
            <label th:for="zipcode">우편번호</label>
            <input type="text" th:field="*{zipcode}" class="form-control" placeholder="우편번호를 입력하세요">
        </div>
        <button type="submit" class="btn btn-primary">Submit</button>
    </form>
```

→ 입력값은 memberForm의 필드로 매핑이 되고, 유효성 검증 실패 시 타임리프를 통해 에러 메시지가 화면에 표시된다를

→ 폼이 제출되면 POST 방식으로 /members/new URL로 요청이 전달되고, MemberController의 @PostMapping(”/members/new”) 메서드가 호출된다

```java
	// MemberController
	
	@PostMapping("/members/new")
	private String create(@Valid MemberForm form) {
		
		Address address = new Address(form.getCity(), form.getStreet(), form.getZipcode());

		Member member = new Member();
		member.setName(form.getName());
		member.setAddress(address);

		memberService.join(member);
		return "redirect:/";
	}
```

- @Valid 어노테이션을 통해 MemberForm에 선언된 유효성 검증(@NotEmpty 등)이 실행된다.
- 검증에 성공하면 MemberForm 데이터로 Address, Member 객체를 생성하고 memberService.join()을 통해 회원을 저장한다

**만약 유효성 검증에 실패하면?**

- 예를 들어 이름(name) 필드를 입력하지 않으면 @NotEmpty 검증이 실패한다
- 이때 스프링은 MethodArgumentNotValidException을 발생시키며, 기본적으로는 에러 페이지로 넘어간다

이를 컨트롤러에서 직접 처리하려면 BindingResult를 추가해야 한다

```java
// MemberController

@PostMapping("/members/new")
	private String create(@Valid MemberForm form, **BindingResult result) {

		if(result.hasErrors()) {
			return "members/createMemberForm";
		}**
 ...
}
```

- BindingResult는 유효성 검증 실패 시 컨트롤러가 예외를 던지지 않고 이 객체를 통해 에러 정보를 확인할 수 있게 해준다
- result.hasErrors()가 true라면 검증에 실패한 것이므로, 다시 회원가입 폼으로 돌아가서 에러 메시지를 출력한다
- 타임리프와 같은 템플릿 엔진이 이 에러 정보를 활용하여 사용자에게 어떤 값이 잘못되었는지 화면에 표시할 수 있다

---

### 2. 회원목록조회

```java
// MemberController 
****
@GetMapping("/members")
	public String list(Model model) {
	
		List<Member> members = memberService.findMembers();
		model.addAttribute("members", members);
		
		return "members/memberList";
	}
```

- GET /members 요청이 들어오면 MemberService를 통해 전체 회원 목록을 조회한다
- 조회된 회원 목록을 Model에 담아 members/memberList.html 뷰로 넘긴다
- 뷰에서는 이 데이터를 사용해 회원 리스트를 화면에 출력한다.


> ### 💡왜 Member 엔티티를 직접 뷰나 API에 전달하지 않고 DTO (MemberForm, MemberDto)를 써야 할까?
>
> **엔티티는 핵심 비즈니스 로직만 담당하고 화면이나 API 스펙에 의존하지 않아야 한다**
>
> - 엔티티에 화면/요청에 특화된 데이터나 로직이 추가되면 코드가 점점 무거워지고 유지보수가 어려워진다
>
> **화면/API에 엔티티를 직접 노출하면 보안과 안정성 문제 발생**
>
> - 만약 Member 엔티티에 password 필드가 나중에 추가되면, 이를 외부에 실수로 노출할 수 있다.
> - 엔티티 구조가 변경되면 API 스펙도 의도치 않게 바뀔 수 있다 (ex. 새로운 필드 노출, JSON 구조 변경)
>
> **DTO는 화면/요청 요구사항에 맞게 데이터를 가공해 전달할 수 있다**
>
> - 필요한 필드만 담고, 포맷도 자유롭게 조정 가능
>
> **유지보수와 재사용성을 높일 수 있다**
>
> - 엔티티는 서비스 로직에 집중하고, DTO는 각기 다른 API나 화면에 맞게 커스터마이징
>
> ### **✅ 결론**
>
> - 컨트롤러나 API에서는 엔티티 대신 DTO나 Form 객체를 사용하자
> - 엔티티는 DB와 비즈니스 로직 중심, DTO는 화면/API 데이터 전달용으로 명확히 역할을 분리하자

---

### 3. 상품 등록

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

	@PostMapping("/items/new")
	public String create(BookForm form) {

		Book book = new Book();
		book.setName(form.getName());
		book.setPrice(form.getPrice());
		book.setStockQuantity(form.getStockQuantity());
		book.setAuthor(form.getAuthor());
		book.setIsbn(form.getIsbn());

		itemService.saveItem(book);
		return "redirect:/items";
	}
}
```

**GET /items/new 요청이 들어오면**

- 컨트롤러는 빈 BookForm 객체를 생성해 Model에 담고 items/createItemForm.html 뷰로 전달한다
- 뷰에서는 이 BookForm을 기반으로 입력 폼을 초기화하고 사용자에게 빈 입력창을 제공

**사용자가 입력한 데이터를 제출하면**

- items/createItemForm.html의 form 태그가 POST 방식으로 /items/new URL로 데이터를 전송한다
- 이때 form의 입력값은 BookForm 객체의 필드에 자동으로 매핑된다

**POST /items/new 요청이 들어오면**

- 컨트롤러는 전달받은 BookForm 데이터를 기반으로 새로운 Book 엔티티를 생성한다
- Book 객체의 각 필드는 BookForm 객체의 필드에 자동으로 매핑된다

---

## 4. 상품 수정

**수정 폼 조회**

```java
// ItemController

@GetMapping("/items/{itemId}/edit")
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
```

- 수정하고자 하는 상품의 itemId를 @PathVariable로 받아, 기존 상품 데이터를 조회한다
- 조회한 데이터를 BookForm에 담아 수정 화면(items/updateItemForm.html)으로 전달한다
- 사용자에게 기존 데이터가 채워진 수정 폼을 제공한다

**병합(merge) 방식 수정 (권장 X)**

```java
// ItemController

@PostMapping("/items/{itemId}/edit")
	public String updateItem(@ModelAttribute("form") BookForm form) {

		Book book = new Book();
		book.setId(form.getId());
		book.setName(form.getName());
		book.setPrice(form.getPrice());
		book.setStockQuantity(form.getStockQuantity());
		book.setAuthor(form.getAuthor());
		book.setIsbn(form.getIsbn());

		itemService.saveItem(book); // 내부적으로 em.merge 호출
		return "redirect:/items";
	}
```

- form 데이터로 새로운 Book 객체를 생성하고 id를 포함한 모든 필드를 채운다
- itemService.saveItem(book)을 호출하면 내부적으로 em.merge(book)이 실행된다


> 💡**@ModelAttribute**
>
> - @ModelAttribute(”form”) BookForm form은 요청 파라미터를 BookForm 객체의 필드에 자동으로 바인딩하고, Model에 “form”이름으로 추가한다
> - 뷰에서 ${form}으로 접근 가능하고, 생략하더라도 스프링이 자동으로 @ModelAttribute처럼 처리하지만 이름 지정과 코드 의도 전달을 위해 명시하면 더 정확하다


> 💡**준영속 엔티티란?**
>
> 준영속 엔티티는 영속성 컨텍스트가 더는 관리하지 않는 엔티티 객체를 말한다
>
> merge 방식에서는 form 데이터로 새로 생성한 Book이 준영속 엔티티 상태다
>
> 이 객체는 JPA가 변경 감지를 하지 않으며, 단순히 데이터 전달 용도로만 사용된다


> 💡**merge의 동작 방식**
>
> 1️⃣ 준영속 엔티티의 id로 1차 캐시 (영속성 컨텍스트)에서 엔티티를 조회
>
> - 만약 1차 캐시에 없으면 DB에서 기존 엔티티 조회
>
> 2️⃣  조회해온 영속 엔티티에 준영속 엔티티 값 전체 덮어쓰기
>
> - null 값도 덮어쓰기 때문에 기존 데이터 유실 가능 (권장하지 않는 이유)
>
> 3️⃣ 새로운 영속 엔티티를 반환 (원래 객체는 여전히 준영속 상태)


> 💡**실무에서 merge 방식의 문제**
>
> 실무에서는 상품 생성 시 작성했던 모든 필드를 수정하는 경우는 드물고, 일부 필드만 수정하는 경우가 대부분이다
>
> merge는 전체 덮어쓰기가 기본 동작이라 비어있는 필드(nul가 기존 데이터를 덮어쓸 위험이 있다
>
> merge를 안전하게 사용하려면 모든 필드를 신경 써서 채워야 하는데, 코드도 번거롭고 오류 가능성도 높다



**변경 감지(dirty checking) 방식 (권장 O)**

```java
// ItemController

@PostMapping("/items/{itemId}/edit")
	public String updateItem(@PathVariable("itemId") Long itemId, @ModelAttribute("form") BookForm form) {
		itemService.updateItem(itemId, form.getName(), form.getPrice(), form.getStockQuantity());
		return "redirect:/items";
	}
```

```java
// ItemService

@Transactional
public void updateItem(Long id, String name, int price, int stockQuantity) {
	
	Item item = itemRepository.findOne(id);
	
	item.setName(name);
	item.setPrice(price);
	item.setStockQuantity(stockQuantity);
}
```

- 영속 상태의 엔티티를 트랜잭션 안에서 조회하고 수정이 필요한 필드만 setter로 변경
- 트랜잭션 커밋 시점에 변경 감지가 되어 변경된 필드만 update 쿼리가 날아간다.
- null 덮어쓰기 위험이 없고, save 메서드를 호출하지 않아도 된다

---

### 5. 상품 주문

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
	public String order(
		@RequestParam("memberId") Long memberId,
		@RequestParam("itemId") Long itemId,
		@RequestParam("count") int count
	) {
		orderService.order(memberId, itemId, count);
		return "redirect:/orders";
	}
}
```

**GET /order 요쳥이 들어오면**

- createForm() 메서드가 호출되면서, 전체 회원 목록과 전체 아이템 목록 조회하고
- 조회된 데이터를 Model에 담아 order/orderForm.html 뷰로 전달한다
- 사용자에게 회원, 상품, 수량을 선택할 수 있는 폼을 제공한다

**POST /order 요청이 들어오면**

- order() 메서드가 호출되면서, 폼에서 입력받은 회원 id, 상품 id, 주문 수량을 파라미터로 전달받는다
- orderService.order() 메서드로 회원 id, 아이템id, 수량 데이터를 담아 실행


> 💡**실무 권장 패턴**
>
> - 컨트롤러는 최대한 단순하고 가볍게 작성
> - 핵심 비즈니스 로직은 트랜잭션과 함께 서비스 계층에서 처리
> - 서비스 계층에서 식별자로 영속성 엔티티를 조회하고 처리
