---
title: "Spring 백엔드에서 API 요청 하나는 어떻게 처리될까?"
date: 2026-09-03 09:50:05 +0900
last_modified_at: 2026-10-01
categories: [Backend, Spring]
tags: [spring-mvc, api, request-flow]
source_url: "https://velog.io/@joker901010/Spring-%EB%B0%B1%EC%97%94%EB%93%9C%EC%97%90%EC%84%9C-API-%EC%9A%94%EC%B2%AD-%ED%95%98%EB%82%98%EB%8A%94-%EC%96%B4%EB%96%BB%EA%B2%8C-%EC%B2%98%EB%A6%AC%EB%90%A0%EA%B9%8C"
---
백엔드를 처음 공부할 때는 API 하나를 굉장히 단순하게 생각했다.

프론트에서 API를 호출하면 Controller가 요청을 받고, Service를 호출한 다음 DB에 저장하고 응답을 돌려주는 정도라고 생각했다.

큰 틀에서는 맞는 말이다.

그런데 실제 코드를 작성하다 보면 그 사이에 생각보다 많은 과정이 있다는 걸 알게 된다.

예를 들어 주문 하나를 생성하는 API만 만들어도 다음과 같은 것들을 고민해야 한다.

- 요청으로 어떤 값을 받을지
- 잘못된 값이 들어오면 어디에서 막을지
- 사용자가 실제로 존재하는지
- 상품이 존재하는지
- 재고가 충분한지
- 주문과 재고 변경을 하나의 작업으로 묶을지
- 실패했을 때 어떤 상태 코드를 내려줄지
- 응답에는 어떤 값을 보여줄지

이번 글에서는 아주 작은 **주문 생성 API 하나가 서버 안에서 어떤 과정을 거쳐 처리되는지** 처음부터 끝까지 따라가 보려고 한다.

---

## 먼저 전체 흐름부터 보자

Spring MVC를 기준으로 API 요청 하나는 대략 다음과 같은 흐름으로 처리된다.

```
HTTP Request
    ↓
DispatcherServlet
    ↓
Controller
    ↓
Request DTO / Validation
    ↓
Service
    ↓
Domain
    ↓
Repository
    ↓
Database
    ↓
Response DTO
    ↓
HTTP Response
```

처음 보면 단계가 많아 보이지만 하나씩 보면 어렵지 않다.

각 계층이 서로 다른 역할을 나눠 가지고 있을 뿐이다.

이번 글에서는 아래 주문 생성 API를 기준으로 하나씩 살펴보자.

```
POST /orders
Content-Type: application/json

{
  "userId": 1,
  "productId": 10,
  "quantity": 2
}
```

사용자 1번이 상품 10번을 2개 주문하는 요청이다.

---

## 1. 클라이언트가 HTTP 요청을 보낸다

모든 시작은 클라이언트의 요청이다.

클라이언트는 브라우저일 수도 있고 모바일 앱일 수도 있고, 다른 서버일 수도 있다.

```
POST /orders
Content-Type: application/json

{
  "userId": 1,
  "productId": 10,
  "quantity": 2
}
```

여기에는 크게 세 가지 정보가 들어 있다.

```
POST        → 어떤 동작을 원하는지
/orders     → 어떤 API를 호출하는지
JSON Body   → 주문 생성에 필요한 데이터
```

Spring 서버는 이 요청을 받아 적절한 Controller 메서드와 연결해야 한다.

---

## 2. DispatcherServlet이 요청을 받는다

Spring MVC에서는 HTTP 요청이 들어오면 `DispatcherServlet`이 중심에서 요청 처리를 담당한다.

우리가 직접 DispatcherServlet을 호출하는 것은 아니지만 내부적으로는 요청에 맞는 Controller를 찾아 연결해 준다.

예를 들어 다음 Controller가 있다고 해보자.

```
@RestController
@RequestMapping("/orders")
public class OrderController {

    @PostMapping
    public ResponseEntity<OrderResponse> createOrder(
            @RequestBody OrderCreateRequest request
    ) {
        // ...
    }
}
```

`POST /orders` 요청이 들어오면 Spring은 요청 URL과 HTTP Method를 확인하고 이 메서드를 찾아 실행한다.

즉,

```
POST /orders
```

라는 HTTP 요청이

```
createOrder(...)
```

라는 Java 메서드 호출로 연결되는 것이다.

---

## 3. JSON을 Java 객체로 변환한다

하지만 한 가지 문제가 있다.

클라이언트가 보낸 것은 JSON이다.

```
{
  "userId": 1,
  "productId": 10,
  "quantity": 2
}
```

반면 Controller가 받고 싶은 것은 Java 객체다.

```
OrderCreateRequest request
```

Spring은 이 둘을 자동으로 변환해 준다.

```
public record OrderCreateRequest(

    @NotNull
    Long userId,

    @NotNull
    Long productId,

    @Min(1)
    int quantity

) {
}
```

그리고 Controller에서는 `@RequestBody`를 붙여 요청 Body를 객체로 받을 수 있다.

```
@PostMapping
public ResponseEntity<OrderResponse> createOrder(
        @Valid @RequestBody OrderCreateRequest request
) {
    // ...
}
```

Spring MVC에서는 `HttpMessageConverter`가 HTTP Body를 읽고 Java 객체로 변환하는 역할을 한다.

Jackson을 사용하는 일반적인 Spring Boot 환경이라면 JSON이 자연스럽게 Java 객체로 변환된다.

우리는 단순히

```
request.userId()
request.productId()
request.quantity()
```

처럼 사용할 수 있다.

---

## 4. 잘못된 요청은 먼저 걸러낸다

요청이 Java 객체로 변환됐다고 바로 Service로 넘기는 것은 아니다.

먼저 기본적인 입력값을 확인할 수 있다.

```
public record OrderCreateRequest(

    @NotNull
    Long userId,

    @NotNull
    Long productId,

    @Min(1)
    int quantity

) {
}
```

예를 들어 사용자가 다음과 같이 요청했다고 해보자.

```
{
  "userId": 1,
  "productId": 10,
  "quantity": 0
}
```

주문 수량이 0이다.

`quantity`에는 `@Min(1)`이 있기 때문에 Validation에 실패한다.

```
Request
   ↓
DTO 변환
   ↓
Validation 실패
   ↓
400 Bad Request
```

이 경우 Service까지 들어갈 필요가 없다.

애초에 정상적인 주문 요청이 아니기 때문이다.

응답은 프로젝트 정책에 따라 다음처럼 만들 수 있다.

```
{
  "code": "VALIDATION_ERROR",
  "message": "입력값이 올바르지 않습니다.",
  "errors": [
    {
      "field": "quantity",
      "message": "1 이상이어야 합니다."
    }
  ]
}
```

여기서 중요한 점이 하나 있다.

Validation은 보통 **입력값 자체의 형식이나 범위**를 검증하는 데 사용한다.

예를 들면 이런 것들이다.

```
값이 비어 있는가?
숫자가 최소값 이상인가?
문자열 길이가 올바른가?
이메일 형식이 맞는가?
```

반대로

> 이 상품의 실제 재고가 충분한가?

같은 것은 단순 입력값 검증이 아니라 비즈니스 규칙에 가깝다.

이런 검증은 뒤에서 처리하게 된다.

---

## 5. Controller는 Service에게 실제 작업을 맡긴다

Validation까지 통과했다면 Controller가 Service를 호출한다.

```
@RestController
@RequestMapping("/orders")
public class OrderController {

    private final OrderService orderService;

    public OrderController(OrderService orderService) {
        this.orderService = orderService;
    }

    @PostMapping
    public ResponseEntity<OrderResponse> createOrder(
            @Valid @RequestBody OrderCreateRequest request
    ) {

        OrderResponse response = orderService.createOrder(request);

        return ResponseEntity
                .status(HttpStatus.CREATED)
                .body(response);
    }
}
```

Controller가 하는 일은 생각보다 많지 않다.

```
요청을 받는다.
↓
입력값을 전달한다.
↓
Service를 호출한다.
↓
결과를 HTTP 응답으로 반환한다.
```

Controller 안에 주문 생성 로직을 전부 넣을 수도 있다.

하지만 그렇게 하면 Controller가 점점 커진다.

사용자 조회, 상품 조회, 재고 확인, 재고 차감, 주문 생성, 주문 저장까지 전부 Controller 안에 들어가기 시작하면 코드를 읽기도 어렵고 테스트하기도 힘들어진다.

그래서 실제 기능의 흐름은 Service에 맡긴다.

---

## 6. Service에서 주문 생성 흐름을 진행한다

Service를 보자.

```
@Service
public class OrderService {

    private final UserRepository userRepository;
    private final ProductRepository productRepository;
    private final OrderRepository orderRepository;

    public OrderService(
            UserRepository userRepository,
            ProductRepository productRepository,
            OrderRepository orderRepository
    ) {
        this.userRepository = userRepository;
        this.productRepository = productRepository;
        this.orderRepository = orderRepository;
    }

    @Transactional
    public OrderResponse createOrder(OrderCreateRequest request) {

        User user = userRepository.findById(request.userId())
                .orElseThrow(
                    () -> new UserNotFoundException(request.userId())
                );

        Product product = productRepository.findById(request.productId())
                .orElseThrow(
                    () -> new ProductNotFoundException(request.productId())
                );

        product.decreaseStock(request.quantity());

        Order order = Order.create(
                user,
                product,
                request.quantity()
        );

        Order savedOrder = orderRepository.save(order);

        return OrderResponse.from(savedOrder);
    }
}
```

코드를 순서대로 읽으면 꽤 단순하다.

```
1. 사용자를 조회한다.
2. 상품을 조회한다.
3. 재고를 차감한다.
4. 주문을 생성한다.
5. 주문을 저장한다.
6. 응답 DTO를 만든다.
```

Service의 중요한 역할 중 하나는 이런 **기능의 흐름을 연결하는 것**이다.

---

## 7. Repository를 통해 DB에서 데이터를 가져온다

Service가 직접 SQL을 작성해서 DB와 통신하지는 않는다.

데이터 조회와 저장은 Repository를 통해 처리한다.

```
@Repository
public interface ProductRepository
        extends JpaRepository<Product, Long> {
}
```

```
@Repository
public interface OrderRepository
        extends JpaRepository<Order, Long> {
}
```

예를 들어

```
productRepository.findById(request.productId());
```

를 호출하면 Repository가 DB에서 해당 상품을 조회한다.

Spring Data JPA를 사용한다면 간단한 CRUD 메서드는 기본적으로 제공된다.

그래서 Service는

> 상품을 어떻게 SELECT할 것인가?

보다는

> 상품을 가져와서 무엇을 할 것인가?

에 집중할 수 있다.

---

## 8. 상품이 없으면 여기서 실패한다

클라이언트가 존재하지 않는 상품 번호를 보냈다고 해보자.

```
Product product = productRepository
        .findById(request.productId())
        .orElseThrow(
            () -> new ProductNotFoundException(request.productId())
        );
```

DB에서 상품을 찾지 못하면 `ProductNotFoundException`이 발생한다.

이 예외를 그대로 클라이언트에게 보여주는 것은 좋지 않다.

보통 `@RestControllerAdvice` 같은 공통 예외 처리기를 두고 HTTP 응답으로 변환한다.

예를 들면 다음과 같다.

```
{
  "code": "PRODUCT_NOT_FOUND",
  "message": "상품을 찾을 수 없습니다."
}
```

HTTP 상태 코드는 `404 Not Found`를 사용할 수 있다.

```
Service
  ↓
ProductRepository 조회
  ↓
상품 없음
  ↓
ProductNotFoundException
  ↓
Exception Handler
  ↓
404 Not Found
```

---

## 9. 재고 확인은 Product가 직접 할 수도 있다

상품을 찾았다고 주문이 항상 가능한 것은 아니다.

재고가 부족할 수도 있다.

```
@Entity
public class Product {

    @Id
    private Long id;

    private String name;

    private int stock;

    protected Product() {
    }

    public void decreaseStock(int quantity) {

        if (quantity <= 0) {
            throw new InvalidQuantityException(quantity);
        }

        if (stock < quantity) {
            throw new NotEnoughStockException(id);
        }

        this.stock -= quantity;
    }
}
```

Service에서 이렇게 호출한다.

```
product.decreaseStock(request.quantity());
```

여기서 재미있는 점은 Service가 직접

```
if (product.getStock() < quantity) {
    ...
}
```

라고 판단하지 않는다는 것이다.

상품의 재고 상태를 가장 잘 알고 있는 것은 `Product` 자신이다.

그래서

```
product.decreaseStock(quantity);
```

처럼 객체에게 행동을 맡길 수 있다.

이런 식으로 작성하면

```
Service
→ 주문이라는 유스케이스의 흐름을 담당

Product
→ 자신의 재고 상태와 재고 변경 규칙을 담당
```

처럼 책임을 나눌 수 있다.

물론 모든 로직을 반드시 Domain 객체에 넣어야 한다는 뜻은 아니다.

프로젝트 규모와 구조에 따라 Service에 두는 경우도 많다.

중요한 것은 **어떤 객체가 이 규칙을 책임지는 것이 자연스러운지 생각해 보는 것**이다.

---

## 10. 주문을 만들고 DB에 저장한다

재고 차감까지 정상적으로 끝났다면 주문을 생성한다.

```
Order order = Order.create(
        user,
        product,
        request.quantity()
);
```

그리고 Repository를 통해 저장한다.

```
Order savedOrder = orderRepository.save(order);
```

이 과정에서 실제 DB에는 주문 데이터가 INSERT 된다.

전체 흐름은 대략 다음과 같다.

```
Service
 ↓
Order 생성
 ↓
OrderRepository.save()
 ↓
JPA
 ↓
Database
```

---

## 11. 왜 `@Transactional`을 사용할까?

주문 생성에는 여러 작업이 함께 들어간다.

```
상품 조회
재고 차감
주문 생성
주문 저장
```

그런데 재고는 줄었는데 주문 저장이 실패하면 어떻게 될까?

```
재고: -2
주문: 저장 실패
```

데이터가 이상해진다.

그래서 주문 생성처럼 여러 데이터 변경이 하나의 작업으로 묶여야 하는 경우 Service 메서드에 `@Transactional`을 두는 경우가 많다.

```
@Transactional
public OrderResponse createOrder(OrderCreateRequest request) {
    // ...
}
```

트랜잭션 안에서 작업을 처리하면 중간에 예외가 발생했을 때 변경 내용을 롤백할 수 있다.

쉽게 생각하면

> 여기 있는 작업들은 성공하면 같이 성공하고, 실패하면 같이 취소한다.

라고 이해하면 된다.

실제 트랜잭션에는 격리 수준, 락, 동시성 같은 더 복잡한 내용이 있지만 처음에는 이 정도 개념부터 잡아도 충분하다.

---

## 12. Entity를 그대로 응답하지 않고 DTO를 만든다

주문 저장까지 끝났다면 클라이언트에게 결과를 돌려줘야 한다.

여기서 Entity를 그대로 반환하기보다는 Response DTO를 따로 만드는 경우가 많다.

```
public record OrderResponse(
        Long orderId,
        Long productId,
        int quantity,
        String status
) {

    public static OrderResponse from(Order order) {
        return new OrderResponse(
                order.getId(),
                order.getProductId(),
                order.getQuantity(),
                order.getStatus().name()
        );
    }
}
```

Service에서는 이렇게 변환한다.

```
return OrderResponse.from(savedOrder);
```

DTO를 따로 만들면 클라이언트에게 정말 필요한 값만 공개할 수 있다.

Entity에 필드가 20개 있다고 해서 그 20개를 전부 API 응답으로 보여줄 필요는 없다.

또 Entity 구조가 바뀌더라도 API 응답 구조를 어느 정도 분리해서 관리할 수 있다.

---

## 13. Java 객체가 다시 JSON으로 변환된다

Controller는 Service에서 받은 결과를 반환한다.

```
return ResponseEntity
        .status(HttpStatus.CREATED)
        .body(response);
```

여기에서 `response`는 Java 객체다.

```
OrderResponse
```

Spring은 이것을 다시 JSON으로 변환한다.

결과적으로 클라이언트는 다음과 같은 응답을 받는다.

```
HTTP/1.1 201 Created
Content-Type: application/json
```

```
{
  "orderId": 100,
  "productId": 10,
  "quantity": 2,
  "status": "CREATED"
}
```

처음 요청에서는 JSON이 Java 객체로 바뀌었고,

```
JSON
→ OrderCreateRequest
```

응답에서는 반대로 Java 객체가 JSON으로 바뀐다.

```
OrderResponse
→ JSON
```

---

# 지금까지의 전체 흐름 다시 따라가 보기

이제 처음부터 끝까지 한 번 연결해 보자.

사용자가 다음 요청을 보낸다.

```
POST /orders

{
  "userId": 1,
  "productId": 10,
  "quantity": 2
}
```

그러면 서버 안에서는 대략 이런 일이 일어난다.

### 1. 요청이 서버로 들어온다

```
POST /orders
```

### 2. DispatcherServlet이 요청을 받는다

Spring MVC가 요청을 처리할 Controller를 찾는다.

### 3. Controller와 매핑된다

```
@PostMapping
public ResponseEntity<OrderResponse> createOrder(...)
```

### 4. JSON을 Request DTO로 변환한다

```
JSON
↓
OrderCreateRequest
```

### 5. Validation을 수행한다

```
userId가 null인가?
productId가 null인가?
quantity가 1 이상인가?
```

### 6. Controller가 Service를 호출한다

```
orderService.createOrder(request);
```

### 7. 사용자를 조회한다

```
userRepository.findById(...)
```

### 8. 상품을 조회한다

```
productRepository.findById(...)
```

### 9. 상품이 재고를 확인하고 차감한다

```
product.decreaseStock(quantity);
```

### 10. 주문을 생성한다

```
Order.create(...)
```

### 11. 주문을 저장한다

```
orderRepository.save(order);
```

### 12. Response DTO로 변환한다

```
OrderResponse.from(savedOrder);
```

### 13. Controller가 201 Created를 반환한다

```
ResponseEntity.status(HttpStatus.CREATED)
```

### 14. Java 객체가 JSON으로 변환된다

```
OrderResponse
↓
JSON
```

### 15. 클라이언트가 응답을 받는다

```
{
  "orderId": 100,
  "productId": 10,
  "quantity": 2,
  "status": "CREATED"
}
```

한 줄로 줄이면 결국 이 흐름이다.

```
HTTP Request
→ Controller
→ DTO / Validation
→ Service
→ Domain
→ Repository
→ DB
→ Response DTO
→ HTTP Response
```

---

# 성공하는 경우보다 실패하는 경우가 더 다양하다

API를 처음 만들 때는 정상적으로 동작하는 코드부터 생각하게 된다.

하지만 실제 서비스에서는 실패하는 상황도 굉장히 많다.

주문 API만 해도 바로 몇 가지가 나온다.

```
잘못된 요청값
사용자가 존재하지 않음
상품이 존재하지 않음
재고 부족
중복 주문
DB 오류
권한 없음
```

예를 들어 재고가 부족하다면 Domain에서 다음 예외가 발생할 수 있다.

```
if (stock < quantity) {
    throw new NotEnoughStockException(id);
}
```

그리고 공통 예외 처리기가 이를 API 응답으로 변환한다.

```
{
  "code": "NOT_ENOUGH_STOCK",
  "message": "상품 재고가 부족합니다."
}
```

HTTP 상태 코드는 프로젝트 정책에 따라 `400 Bad Request`나 `409 Conflict` 등을 사용할 수 있다.

어떤 코드가 무조건 정답이라기보다는 **팀에서 상태 코드의 의미와 API 정책을 일관되게 정하는 것이 중요하다.**

---

# 각 계층은 무슨 일을 할까?

처음에는 Controller, Service, Repository가 왜 나뉘어 있는지 헷갈릴 수 있다.

아주 단순하게 정리하면 이렇게 생각할 수 있다.

## Controller

HTTP 요청과 응답을 담당한다.

```
어떤 URL인가?
어떤 HTTP Method인가?
어떤 요청 DTO를 받을 것인가?
어떤 HTTP Status를 반환할 것인가?
```

---

## Request DTO

클라이언트가 보낸 데이터를 담는다.

그리고 기본적인 입력값 검증을 할 수 있다.

```
null 여부
문자열 길이
숫자 범위
형식
```

---

## Service

하나의 기능이 어떤 순서로 진행될지 조율한다.

```
사용자 조회
→ 상품 조회
→ 재고 차감
→ 주문 생성
→ 주문 저장
```

트랜잭션의 경계를 잡는 역할도 자주 한다.

---

## Domain

자신의 상태와 관련된 규칙을 처리한다.

```
product.decreaseStock(quantity);
order.cancel();
coupon.use();
```

즉,

> 이 상태 변경이 가능한가?

를 객체가 직접 판단하도록 만들 수 있다.

---

## Repository

데이터 조회와 저장을 담당한다.

```
findById()
save()
delete()
```

Service가 DB 접근 방법까지 직접 신경 쓰지 않도록 분리해 준다.

---

## Response DTO

클라이언트에게 보여줄 데이터를 담는다.

Entity 전체가 아니라 API에서 필요한 값만 전달한다.

---

## Exception Handler

서버 내부에서 발생한 예외를 적절한 HTTP 응답으로 바꿔준다.

```
ProductNotFoundException
→ 404 Not Found

Validation Exception
→ 400 Bad Request
```

에러 응답 형식을 통일하는 역할도 할 수 있다.

---

# 처음 API를 만들 때 자주 하는 실수

## Controller에 모든 로직 넣기

처음에는 Controller 하나에 다 작성하는 것이 가장 쉬워 보인다.

```
@PostMapping
public ResponseEntity<?> createOrder(...) {

    // 사용자 조회

    // 상품 조회

    // 재고 확인

    // 주문 생성

    // DB 저장

    // 응답 생성
}
```

기능이 한두 개일 때는 큰 문제가 없어 보인다.

하지만 API가 많아지면 Controller가 금방 복잡해진다.

Controller는 HTTP 요청과 응답에 집중하고 실제 기능의 흐름은 Service로 넘기는 편이 관리하기 쉽다.

---

## Service에 모든 규칙 넣기

반대로 Service를 분리했다고 모든 문제가 해결되는 것도 아니다.

Service 안에서 다음과 같은 코드가 계속 생기기 시작할 수 있다.

```
if (product.getStock() < quantity) {
    ...
}

if (order.getStatus() == ...) {
    ...
}

if (coupon.getUsed()) {
    ...
}
```

이런 규칙 중 일부는 해당 Domain 객체가 직접 처리하는 것이 더 자연스러울 수 있다.

```
product.decreaseStock(quantity);

order.cancel();

coupon.use();
```

Service는 **흐름을 조율하고**, Domain은 **자기 상태에 대한 규칙을 처리한다**고 생각하면 구조를 잡는 데 도움이 된다.

---

## Entity를 그대로 API 응답으로 사용하기

처음에는 Entity를 그대로 반환하는 것이 편하다.

하지만 프로젝트가 커지면 Entity 구조와 API 스펙이 강하게 묶여 버릴 수 있다.

또 클라이언트에게 보여줄 필요 없는 필드까지 노출될 수 있다.

그래서 Request DTO와 Response DTO를 따로 두는 경우가 많다.

---

# 로그도 기능의 일부라고 생각하면 좋다

개발 환경에서는 디버거를 붙이면 쉽게 확인할 수 있다.

하지만 운영 서버에서는 그렇지 않다.

사용자가

> 주문했는데 실패했어요.

라고 했을 때 원인을 찾으려면 로그가 필요하다.

예를 들어 이런 정보가 도움이 된다.

```
orderId
userId
productId
요청 시각
어느 단계에서 실패했는지
예외 타입
```

다만 로그를 많이 남긴다고 무조건 좋은 것은 아니다.

비밀번호, 인증 토큰, 주민등록번호 같은 민감한 값은 로그에 남겨서는 안 된다.

운영 환경을 생각하면

> 문제가 발생했을 때 이 요청을 다시 추적할 수 있는가?

도 API를 만들면서 같이 고민할 필요가 있다.

---

# 마무리

백엔드를 처음 공부할 때는 API 하나를 Controller 메서드 하나라고 생각하기 쉽다.

하지만 실제로는 요청이 들어온 순간부터 응답이 나갈 때까지 여러 계층이 함께 움직인다.

```
HTTP Request
→ DispatcherServlet
→ Controller
→ Request DTO
→ Validation
→ Service
→ Domain
→ Repository
→ Database
→ Response DTO
→ HTTP Response
```

각 계층을 나누는 이유는 코드를 복잡하게 만들기 위해서가 아니다.

각자가 맡아야 할 책임을 나누기 위해서다.

처음 API를 만들 때 아래 정도만 한 번씩 생각해 봐도 코드 구조를 잡는 데 도움이 된다.

```
요청은 어떤 DTO로 받을까?

어떤 값은 Controller에 들어오기 전에 검증해야 할까?

이 기능은 어떤 순서로 처리되어야 할까?

어떤 규칙은 Domain 객체가 직접 처리하는 것이 좋을까?

어떤 데이터를 DB에서 조회해야 할까?

트랜잭션으로 묶어야 하는 작업은 무엇일까?

클라이언트에게 어떤 값만 응답할까?

실패하면 어떤 상태 코드와 에러를 내려줄까?
```

처음부터 완벽하게 계층을 나누는 것은 어렵다.

나도 코드를 작성하면서

> 이 로직은 Controller에 있어야 하나?
> Service에 있어야 하나?
> 아니면 객체가 직접 처리해야 하나?

를 계속 고민하게 된다.

그래도 **API 하나를 단순히 Controller 메서드 하나로 보지 않고, 요청부터 응답까지 하나의 흐름으로 보기 시작하면** 백엔드 코드가 훨씬 이해하기 쉬워진다.
