---
title: "비즈니스 로직은 Service에만 넣으면 될까?"
date: 2026-08-21 14:38:11 +0900
last_modified_at: 2026-10-01
categories: [Backend, Design]
tags: [service, domain, business-logic]
source_url: "https://velog.io/@joker901010/%EB%B9%84%EC%A6%88%EB%8B%88%EC%8A%A4-%EB%A1%9C%EC%A7%81%EC%9D%80-Service%EC%97%90%EB%A7%8C-%EB%84%A3%EC%9C%BC%EB%A9%B4-%EB%90%A0%EA%B9%8C"
---
처음에는 **“비즈니스 로직은 Service에 둔다”**라고 이해해도 괜찮다.

하지만 기능이 복잡해질수록 모든 로직을 Service에 몰아넣기보다는 역할에 따라 나누는 것이 좋다.

```
Controller
→ HTTP 요청과 응답을 처리한다.

Service
→ 하나의 기능이 실행되는 전체 흐름을 조율한다.

Domain 객체
→ 자기 상태와 관련된 규칙과 행동을 담당한다.

Repository
→ 데이터를 조회하고 저장한다.
```

쉽게 말하면,

> **Service는 일을 어떤 순서로 진행할지 알고,
> Domain 객체는 자기 상태에서 무엇을 할 수 있는지 안다.**

---

## 처음에는 왜 Service에 비즈니스 로직을 넣을까?

Spring을 처음 배우면 보통 이런 구조부터 접하게 된다.

```
Controller
    ↓
Service
    ↓
Repository
    ↓
Database
```

각 계층의 역할도 대략 이렇게 배운다.

```
Controller → 요청과 응답
Service    → 비즈니스 로직
Repository → 데이터 접근
```

그래서 자연스럽게 이런 생각을 하게 된다.

> “비즈니스 로직은 전부 Service에 넣으면 되는구나.”

완전히 틀린 말은 아니다.

오히려 Controller나 Repository에 비즈니스 규칙을 직접 넣는 것보다는 Service에 모으는 편이 훨씬 낫다.

문제는 서비스가 점점 커졌을 때 생긴다.

---

## 예를 들어 주문 취소 기능을 만들어보자

주문 취소에는 생각보다 여러 작업이 필요할 수 있다.

```
1. 주문을 조회한다.
2. 주문이 취소 가능한 상태인지 확인한다.
3. 주문을 취소 상태로 변경한다.
4. 결제를 취소한다.
5. 재고를 복구한다.
6. 사용한 쿠폰을 복구한다.
7. 고객에게 알림을 보낸다.
```

이 모든 내용을 Service 하나에 넣는다면 처음에는 이렇게 작성할 수 있다.

```
@Service
public class OrderService {

    public void cancelOrder(Long orderId) {
        // 주문 조회
        // 주문 상태 확인
        // 결제 상태 확인
        // 취소 가능 여부 확인
        // 주문 상태 변경
        // 재고 복구
        // 쿠폰 복구
        // 결제 취소
        // 알림 발송
    }
}
```

처음에는 별문제가 없어 보인다.

하지만 기능이 계속 추가되면 Service가 점점 많은 것을 알게 된다.

```
주문 상태 규칙
결제 규칙
재고 규칙
쿠폰 규칙
알림 규칙
```

결국 Service가

> “업무 흐름을 조율하는 클래스”

가 아니라

> “모든 비즈니스 규칙을 알고 있는 거대한 클래스”

가 될 수 있다.

그래서 비즈니스 로직을 볼 때는 한 번 더 질문해보는 것이 좋다.

```
이 로직은 전체 기능의 흐름인가?

아니면 특정 객체의 상태와 관련된 규칙인가?
```

이 질문이 Service와 Domain의 역할을 구분하는 데 도움이 된다.

---

# Service와 Domain을 쉽게 구분해보자

주문 취소 상황을 다시 생각해보자.

고객이 주문 취소를 요청했다.

그러면 시스템은 다음과 같은 일을 처리해야 한다.

```
주문 조회
→ 취소 가능 여부 확인
→ 주문 취소
→ 결제 취소
→ 재고 복구
→ 알림 발송
```

이 **전체 순서를 조율하는 역할**은 Service와 잘 어울린다.

```
OrderService

"주문 취소라는 업무를 진행한다."
```

그런데 다음 질문은 조금 다르다.

> “이 주문은 지금 취소할 수 있는 상태인가?”

예를 들어 주문 상태가 다음과 같다고 해보자.

```
CREATED
PAID
SHIPPING
CANCELLED
```

그리고 규칙이 다음과 같다면,

```
CREATED → 취소 가능
PAID    → 취소 가능
SHIPPING → 취소 불가능
```

이 규칙은 `OrderService`보다 **Order 자체와 더 밀접한 규칙**이다.

그래서 이렇게 생각할 수 있다.

```
OrderService

무엇을 어떤 순서로 실행할지 결정한다.
```

```
Order

내 현재 상태에서 어떤 행동이 가능한지 판단한다.
```

---

# 코드로 비교해보자

먼저 모든 판단을 Service에서 하는 코드를 보자.

```
@Service
public class OrderService {

    private final OrderRepository orderRepository;

    public OrderService(OrderRepository orderRepository) {
        this.orderRepository = orderRepository;
    }

    @Transactional
    public void cancelOrder(Long orderId) {

        Order order = orderRepository.findById(orderId)
                .orElseThrow(() -> new OrderNotFoundException(orderId));

        if (order.getStatus() != OrderStatus.CREATED
                && order.getStatus() != OrderStatus.PAID) {
            throw new OrderCannotCancelException(orderId);
        }

        order.setStatus(OrderStatus.CANCELLED);
    }
}
```

코드는 정상적으로 동작한다.

그런데 자세히 보면 Service가 두 가지 규칙을 알고 있다.

```
CREATED 또는 PAID 상태에서만 취소할 수 있다.

취소하면 상태를 CANCELLED로 변경해야 한다.
```

둘 다 **Order의 상태와 직접 관련된 규칙**이다.

그렇다면 이 규칙을 Order가 직접 가지게 만들 수도 있다.

---

## Order에게 취소 책임을 주기

```
@Entity
public class Order {

    @Id
    private Long id;

    @Enumerated(EnumType.STRING)
    private OrderStatus status;

    protected Order() {
    }

    public boolean canCancel() {
        return status == OrderStatus.CREATED
                || status == OrderStatus.PAID;
    }

    public void cancel() {

        if (!canCancel()) {
            throw new OrderCannotCancelException(id);
        }

        this.status = OrderStatus.CANCELLED;
    }

    public Long getId() {
        return id;
    }

    public OrderStatus getStatus() {
        return status;
    }
}
```

이제 주문 자신이 알고 있다.

```
내가 취소 가능한 상태인가?

취소되면 내 상태는 어떻게 변하는가?
```

그러면 Service는 훨씬 단순해진다.

```
@Service
public class OrderService {

    private final OrderRepository orderRepository;

    public OrderService(OrderRepository orderRepository) {
        this.orderRepository = orderRepository;
    }

    @Transactional
    public void cancelOrder(Long orderId) {

        Order order = orderRepository.findById(orderId)
                .orElseThrow(() -> new OrderNotFoundException(orderId));

        order.cancel();
    }
}
```

코드의 역할도 훨씬 명확해진다.

```
OrderService
→ 주문을 찾아서 취소 작업을 진행한다.

Order
→ 자신이 취소 가능한지 판단하고 상태를 변경한다.
```

그리고 코드도 자연스럽게 읽힌다.

```
order.cancel();
```

말 그대로

> “주문을 취소한다.”

라는 의미가 된다.

---

# 그렇다면 Controller에는 무엇을 넣어야 할까?

Controller의 주된 역할은 **HTTP 요청을 받고 응답을 만드는 것**이다.

예를 들어 다음과 같은 것들이 Controller의 관심사다.

```
@PathVariable
@RequestBody
@RequestParam
ResponseEntity
HTTP Status Code
Header
```

그래서 다음과 같이 Controller가 직접 주문을 조회하고 상태까지 판단하는 코드는 피하는 편이 좋다.

```
@PostMapping("/orders/{orderId}/cancel")
public ResponseEntity<Void> cancelOrder(
        @PathVariable Long orderId
) {

    Order order = orderRepository.findById(orderId)
            .orElseThrow(() -> new OrderNotFoundException(orderId));

    if (order.getStatus() != OrderStatus.CREATED
            && order.getStatus() != OrderStatus.PAID) {
        throw new OrderCannotCancelException(orderId);
    }

    order.setStatus(OrderStatus.CANCELLED);

    return ResponseEntity.noContent().build();
}
```

이 Controller는 너무 많은 것을 알고 있다.

```
HTTP 요청 처리
DB 조회
주문 취소 규칙
주문 상태 변경
HTTP 응답 생성
```

또 하나의 문제가 있다.

주문 취소 기능이 HTTP에서만 호출된다는 보장은 없다.

나중에는 다음과 같은 곳에서도 사용할 수 있다.

```
고객용 주문 취소 API
관리자 주문 취소 API
배치 프로그램
메시지 큐 Consumer
고객센터 시스템
```

비즈니스 로직이 Controller 안에 들어 있다면 다른 곳에서 재사용하기가 어려워진다.

그래서 Controller는 가능한 한 간단하게 유지하는 편이 좋다.

```
@PostMapping("/orders/{orderId}/cancel")
public ResponseEntity<Void> cancelOrder(
        @PathVariable Long orderId
) {

    orderService.cancelOrder(orderId);

    return ResponseEntity.noContent().build();
}
```

Controller는 요청을 받고 Service에게 일을 요청한다.

그 정도면 충분한 경우가 많다.

---

# Repository는 무엇을 해야 할까?

Repository의 역할은 **데이터 접근**이다.

예를 들어 Spring Data JPA에서는 다음과 같이 사용할 수 있다.

```
@Repository
public interface OrderRepository
        extends JpaRepository<Order, Long> {
}
```

Repository에서는 다음과 같은 질문에 집중한다.

```
주문을 어떻게 조회할 것인가?

특정 상태의 주문을 어떻게 찾을 것인가?

해당 데이터가 존재하는가?
```

예를 들면 다음과 같다.

```
findById(orderId);

findByStatus(status);

existsByUserId(userId);
```

반대로 Repository가 이런 판단까지 하면 역할이 애매해진다.

```
default Order findCancelableOrder(Long orderId) {

    Order order = findById(orderId)
            .orElseThrow(() -> new OrderNotFoundException(orderId));

    if (order.getStatus() != OrderStatus.CREATED
            && order.getStatus() != OrderStatus.PAID) {
        throw new OrderCannotCancelException(orderId);
    }

    return order;
}
```

`findById()`처럼 데이터를 가져오는 것은 Repository의 역할이다.

하지만

```
CREATED와 PAID 상태에서만 취소할 수 있다.
```

라는 것은 데이터 조회 방법이 아니라 **주문의 비즈니스 규칙**에 가깝다.

만약 정책이 바뀐다면 어떨까?

```
기존

CREATED
PAID

상태에서만 취소 가능
```

이었던 것이

```
PAYMENT_PENDING도 취소 가능
```

으로 바뀔 수도 있다.

이 규칙 때문에 Repository를 수정해야 한다면 역할이 조금 이상해진다.

그래서 보통은 이렇게 생각하는 것이 편하다.

```
Repository

"어떻게 데이터를 가져올까?"
```

```
Domain / Service

"가져온 데이터로 무엇을 할 수 있을까?"
```

---

# Service에는 어떤 로직을 두는 것이 좋을까?

Service는 **하나의 기능을 완성하기 위해 여러 객체를 연결하고 순서를 조율하는 역할**과 잘 어울린다.

예를 들어 주문 생성 기능이 있다고 해보자.

```
회원 조회
상품 조회
주문 생성
재고 차감
주문 저장
결제 요청
쿠폰 사용
알림 발송
```

여기에는 하나의 객체만 등장하는 것이 아니다.

```
User
Product
Order
Coupon
Payment
Repository
외부 결제 API
```

이런 객체들을 연결해서

> “주문 생성이라는 하나의 기능”

을 완성하는 것은 Service가 담당하기 좋다.

예를 들면 다음과 같다.

```
@Transactional
public OrderResponse createOrder(
        OrderCreateRequest request
) {

    User user = userRepository
            .findById(request.userId())
            .orElseThrow(
                    () -> new UserNotFoundException(request.userId())
            );

    Product product = productRepository
            .findById(request.productId())
            .orElseThrow(
                    () -> new ProductNotFoundException(request.productId())
            );

    Order order = Order.create(
            user,
            product,
            request.quantity()
    );

    product.decreaseStock(request.quantity());

    Order savedOrder = orderRepository.save(order);

    return OrderResponse.from(savedOrder);
}
```

Service는 여러 객체를 연결한다.

```
User
Product
Order
UserRepository
ProductRepository
OrderRepository
```

또 여러 DB 작업을 하나의 트랜잭션으로 묶어야 하는 경우에도 Service가 자연스럽다.

```
@Transactional
public void cancelOrder(Long orderId) {

    // 주문 조회
    // 주문 취소
    // 재고 복구
    // 쿠폰 복구
}
```

즉 Service에는 이런 질문이 잘 어울린다.

> “이 기능을 처리하려면 무엇을 어떤 순서로 실행해야 하지?”

---

# Domain 객체에는 어떤 로직을 두는 것이 좋을까?

Domain 객체는 **자기 상태와 직접 관련된 규칙과 행동**을 가지는 것이 자연스럽다.

예를 들어 Order는 자신이 취소 가능한지 판단할 수 있다.

```
public boolean canCancel() {
    return status == OrderStatus.CREATED
            || status == OrderStatus.PAID;
}
```

그리고 자신을 취소 상태로 변경할 수도 있다.

```
public void cancel() {

    if (!canCancel()) {
        throw new OrderCannotCancelException(id);
    }

    this.status = OrderStatus.CANCELLED;
}
```

Product도 자신의 재고를 관리할 수 있다.

```
public void decreaseStock(int quantity) {

    if (quantity <= 0) {
        throw new InvalidQuantityException(quantity);
    }

    if (stock < quantity) {
        throw new NotEnoughStockException(id);
    }

    this.stock -= quantity;
}
```

이렇게 만들면 Service에서 이런 코드를 사용할 수 있다.

```
product.decreaseStock(quantity);

order.cancel();
```

반대로 Service에서 직접 다음처럼 작성할 필요가 줄어든다.

```
if (product.getStock() < quantity) {
    ...
}

product.setStock(
        product.getStock() - quantity
);
```

객체에게 행동을 맡기면 코드가 실제 업무 용어와 비슷하게 읽힌다.

```
product.decreaseStock(quantity);
order.cancel();
coupon.use(user);
```

그리고 관련 규칙도 한 객체 안에 모이게 된다.

---

# Service와 Domain 중 어디에 둘지 헷갈린다면

처음에는 정확히 나누기가 어렵다.

그럴 때는 몇 가지 질문을 해보면 된다.

## 1. 특정 객체의 상태만으로 판단할 수 있는가?

그렇다면 Domain 객체에 둘 수 있는지 먼저 생각해본다.

```
order.canCancel();

coupon.isExpired();

product.hasEnoughStock(quantity);
```

---

## 2. 여러 객체나 Repository를 함께 사용해야 하는가?

그렇다면 Service에 가까운 경우가 많다.

```
회원 조회
주문 조회
상품 조회
결제 API 호출
알림 발송
여러 DB 변경 처리
```

---

## 3. HTTP 요청이나 응답과 관련 있는가?

그렇다면 Controller의 관심사일 가능성이 높다.

```
@PathVariable
@RequestBody
ResponseEntity
Status Code
Header
```

---

## 4. 데이터를 어떻게 조회할지에 대한 문제인가?

그렇다면 Repository와 가깝다.

```
findByEmail();

existsByUserId();

findOrdersByStatus();
```

---

## 5. 특정 객체의 상태를 변경하는 규칙인가?

Domain 객체가 직접 할 수 있는지 생각해본다.

```
order.cancel();

user.changePassword();

product.decreaseStock();

coupon.use();
```

이런 식으로 메서드 이름이 실제 업무 행동처럼 자연스럽게 읽힌다면 Domain 객체에 잘 어울리는 경우가 많다.

---

# 모든 로직을 Service에 넣으면 안 될까?

꼭 그렇지는 않다.

예를 들어 정말 단순한 CRUD 서비스라면 다음과 같은 구조도 충분할 수 있다.

```
Controller
→ Service
→ Repository
```

그리고 Entity는 데이터를 담는 역할만 수행할 수도 있다.

```
@Entity
public class Order {

    private Long id;

    private OrderStatus status;

    public OrderStatus getStatus() {
        return status;
    }

    public void setStatus(OrderStatus status) {
        this.status = status;
    }
}
```

프로젝트가 단순하다면 이것만으로도 충분하다.

문제는 비즈니스 규칙이 많아졌을 때다.

Service에서 계속 이런 코드가 생기기 시작한다.

```
if (order.getStatus() == OrderStatus.PAID) {
    order.setStatus(OrderStatus.CANCELLED);
}
```

그리고 비슷한 코드가 여러 Service에 생긴다면 문제가 커질 수 있다.

```
OrderService
AdminOrderService
PaymentService
BatchOrderService
```

각 Service가 주문 상태 규칙을 조금씩 알고 있게 된다.

그러면 나중에 취소 정책 하나가 바뀌었을 때 여러 곳을 수정해야 할 수도 있다.

그래서 규칙이 복잡해질수록

```
order.cancel();
```

처럼 관련 객체가 자신의 규칙을 직접 가지게 만드는 방법을 생각해볼 수 있다.

이처럼 데이터만 가지고 있고 행동은 대부분 Service에 존재하는 구조를 이야기할 때 **Anemic Domain Model**이라는 표현도 사용한다.

처음부터 이 용어를 외울 필요는 없다.

초보자라면 우선

> “Entity가 무조건 getter/setter만 가지고 있어야 하는 것은 아니다.”

정도로 이해해도 충분하다.

---

# 그렇다고 Domain 객체에 모든 것을 넣으면 안 된다

이번에는 반대로 Domain 객체가 너무 많은 일을 하는 것도 피해야 한다.

예를 들어 다음과 같은 코드는 조심해야 한다.

```
order.cancel(
        orderRepository,
        paymentClient
);
```

Order 내부에서

```
DB 조회
Repository 호출
외부 결제 API 호출
알림 API 호출
```

까지 하기 시작하면 Order가 외부 기술을 너무 많이 알게 된다.

Domain 객체는 가능한 한

```
자기 상태
자기 규칙
자기 행동
```

에 집중하는 편이 좋다.

반면 Repository나 외부 API처럼 여러 의존성을 연결하는 작업은 Service가 맡는 것이 자연스럽다.

---

# Spring에서는 어떻게 보고 있을까?

Spring에서는 `@Controller`, `@Service`, `@Repository` 같은 stereotype annotation으로 각각의 역할을 표현할 수 있다.

크게 보면 다음처럼 이해할 수 있다.

```
@Controller

웹 요청을 처리한다.
```

```
@Service

애플리케이션의 비즈니스 작업을 처리한다.
```

```
@Repository

데이터 접근을 담당한다.
```

또 Spring의 선언적 트랜잭션을 사용할 때 여러 데이터 변경을 하나의 작업 단위로 묶는 코드를 Service 계층에서 관리하는 경우가 많다.

예를 들어,

```
@Transactional
public void cancelOrder(Long orderId) {
    ...
}
```

처럼 하나의 유스케이스 단위로 트랜잭션을 설정할 수 있다.

다만 여기서 중요한 점은

> `@Service`가 붙었다고 해서 모든 비즈니스 규칙을 반드시 Service 안에 작성해야 한다는 의미는 아니라는 것이다.

Service는 전체 유스케이스를 조율하고,

Domain 객체는 자신과 직접 관련된 규칙을 가지도록 나눌 수 있다.

---

# 처음 프로젝트를 만든다면 이렇게 시작해도 좋다

처음부터

```
도메인 모델
애플리케이션 서비스
도메인 서비스
Aggregate
Value Object
```

같은 개념을 모두 적용하려고 할 필요는 없다.

오히려 처음에는 단순하게 시작하는 것이 이해하기 쉽다.

```
Controller
→ Service
→ Repository
```

그리고 코드를 작성하면서 Service가 커지기 시작하면 한 번 살펴본다.

```
이 조건문은 누구의 규칙이지?

이 상태 변경은 누가 알고 있어야 하지?

이 계산은 특정 객체가 직접 해도 되지 않을까?
```

예를 들어 Service에 이런 코드가 계속 나타난다면,

```
if (order.getStatus() == ...) {
    ...
}

if (product.getStock() < ...) {
    ...
}

if (coupon.getExpiredAt() ...) {
    ...
}
```

Domain 객체로 옮길 수 있는 규칙이 없는지 생각해볼 수 있다.

```
order.cancel();

product.decreaseStock(quantity);

coupon.use();
```

이런 식으로 조금씩 책임을 분리하면 된다.

---

# 정리

비즈니스 로직을 무조건 한 계층에 몰아넣을 필요는 없다.

각 로직의 성격에 따라 역할을 나누는 것이 중요하다.

```
Controller
→ HTTP 요청과 응답을 처리한다.

Service
→ 하나의 기능이 실행되는 전체 흐름을 조율한다.
→ 여러 Repository나 외부 시스템을 연결한다.
→ 트랜잭션의 단위가 되기도 한다.

Domain 객체
→ 자기 상태와 관련된 규칙을 판단한다.
→ 자신의 상태를 변경한다.
→ 계산이나 검증 같은 행동을 수행한다.

Repository
→ 데이터를 조회하고 저장한다.
```

처음에는

> **“비즈니스 로직은 Service에 둔다.”**

라고 이해해도 충분하다.

다만 Service 안에 조건문과 상태 변경 코드가 계속 늘어나기 시작한다면 한 번 더 생각해보자.

> **“이 규칙은 특정 객체가 스스로 판단해도 되지 않을까?”**

결국 핵심은 이것이다.

> **Service는 일을 진행하는 방법을 알고,
> Domain 객체는 자기 상태의 의미를 안다.**

코드를 작성하다가 어디에 로직을 넣어야 할지 고민된다면 다음 네 가지 질문부터 해보자.

```
HTTP 요청이나 응답에 대한 코드인가?
→ Controller

데이터를 어떻게 조회할지에 대한 코드인가?
→ Repository

여러 객체를 연결해 하나의 기능을 수행하는 코드인가?
→ Service

특정 객체의 상태와 규칙에 대한 코드인가?
→ Domain
```

이 기준만 잡혀도 Controller, Service, Repository, Entity에 어떤 코드를 넣어야 할지 훨씬 판단하기 쉬워진다.
