---
title: "Spring Validation은 어디서 해야 할까?"
date: 2026-08-10 09:22:29 +0900
last_modified_at: 2026-10-01
categories: [Backend, Spring]
tags: [validation, spring, dto]
source_url: "https://velog.io/@joker901010/Spring-Validation%EC%9D%80-%EC%96%B4%EB%94%94%EC%84%9C-%ED%95%B4%EC%95%BC-%ED%95%A0%EA%B9%8C"
---
백엔드 개발을 하다 보면 한 번쯤 이런 고민을 하게 된다.

> Validation은 Controller에서 해야 할까?
> 아니면 Service에서 해야 할까?

Spring에서는 `@Valid`를 사용하면 요청 값을 쉽게 검증할 수 있다.

그래서 처음에는 이렇게 생각하기 쉽다.

> “DTO에 `@NotBlank`, `@Email`, `@Size` 붙이고 Controller에서 `@Valid` 쓰면 Validation은 끝 아닌가?”

하지만 실제 서비스를 개발하다 보면 `@Valid`만으로 처리할 수 없는 검증이 훨씬 많다.

예를 들어 다음과 같은 경우다.

- 이미 가입된 이메일인가?
- 현재 사용자가 이 게시글을 수정할 권한이 있는가?
- 상품 재고가 충분한가?
- 해당 쿠폰을 현재 주문에 사용할 수 있는가?
- 현재 주문 상태에서 취소가 가능한가?

이런 검증은 단순히 요청 값만 보고 판단할 수 없다.

DB 상태나 현재 사용자 정보, 그리고 서비스의 비즈니스 규칙을 알아야 한다.

그래서 Validation은 한 곳에 몰아넣기보다 **검증의 성격에 따라 위치를 나누는 것이 좋다.**

## 먼저 결론부터

Validation은 크게 다음처럼 나눠 생각하면 이해하기 쉽다.

- **Frontend**: 사용자에게 빠르게 알려주기 위한 검증
- **Request DTO / Controller**: 형식, 필수 값, 길이, 범위 같은 입력 검증
- **Service / Domain**: DB 상태, 권한, 상태 변경 가능 여부 같은 비즈니스 검증
- **DB**: `UNIQUE`, `FOREIGN KEY`, `NOT NULL` 같은 최종 데이터 무결성 보장

핵심은 이것이다.

> 값 자체만 보고 판단할 수 있다면 Request DTO에서 검증하고,
> 시스템의 상태나 비즈니스 규칙이 필요하다면 Service나 Domain에서 검증한다.

---

## Spring에서는 Validation을 어떻게 사용할까?

Spring Framework에서는 `@RequestBody`와 함께 `@Valid` 또는 `@Validated`를 사용해 요청 객체를 검증할 수 있다.

예를 들어 회원가입 API가 있다고 해보자.

```
@PostMapping("/users")
public UserResponse createUser(
        @Valid @RequestBody UserCreateRequest request
) {
    return userService.createUser(request);
}
```

요청 DTO에는 Jakarta Bean Validation의 제약 조건을 선언할 수 있다.

```
public record UserCreateRequest(

    @NotBlank
    @Email
    String email,

    @NotBlank
    @Size(min = 8, max = 50)
    String password,

    @NotBlank
    @Size(max = 20)
    String name

) {
}
```

이렇게 하면 다음과 같은 요청은 Controller 단계에서 걸러낼 수 있다.

```
{
  "email": "not-email",
  "password": "123",
  "name": ""
}
```

이 경우에는 DB를 조회할 필요도 없다.

이메일 형식이 잘못됐고, 비밀번호가 너무 짧으며, 이름도 비어 있기 때문이다.

이런 검증은 요청 값 자체만 보면 판단할 수 있다.

---

## 형식 검증과 비즈니스 검증을 구분하자

Validation을 이해할 때 가장 도움이 되는 기준은 **형식 검증과 비즈니스 검증을 구분하는 것**이다.

OWASP Input Validation Cheat Sheet에서도 입력 검증을 크게 문법적인 검증과 의미적인 검증으로 나누어 설명한다.

### 1. 형식 검증

값 자체만 보고 정상인지 판단할 수 있는 검증이다.

예를 들면 다음과 같다.

- 값이 비어 있지 않은가?
- 이메일 형식인가?
- 문자열이 20자 이하인가?
- 숫자가 1 이상인가?
- 허용된 범위 안의 값인가?
- 필수 값이 전달되었는가?

이런 검증은 Request DTO에서 처리하기 좋다.

```
public record ProductCreateRequest(

    @NotBlank
    String name,

    @Min(1)
    int price,

    @NotNull
    ProductCategory category

) {
}
```

이 검증은 다른 시스템 상태를 알 필요가 없다.

`price`가 1 이상인지 확인하는 데 DB 조회가 필요하지 않기 때문이다.

---

### 2. 비즈니스 검증

반대로 시스템의 현재 상태나 비즈니스 정책을 알아야 판단할 수 있는 검증도 있다.

예를 들면 다음과 같다.

- 이미 존재하는 이메일인가?
- 해당 상품이 현재 판매 중인가?
- 재고가 충분한가?
- 현재 사용자가 해당 게시글 작성자인가?
- 쿠폰 사용 조건을 만족하는가?
- 현재 주문 상태에서 취소할 수 있는가?

이런 검증은 Service나 Domain에서 처리하는 편이 자연스럽다.

```
if (!product.isOnSale()) {
    throw new ProductNotOnSaleException(productId);
}

if (product.getStock() < request.quantity()) {
    throw new NotEnoughStockException(productId);
}
```

요청 형식 자체는 정상일 수 있다.

하지만 현재 시스템 상태에서는 요청을 처리할 수 없는 것이다.

---

# 회원가입 예제로 다시 살펴보기

회원가입 요청 DTO를 다음처럼 만들었다고 해보자.

```
public record UserCreateRequest(

    @NotBlank
    @Email
    String email,

    @NotBlank
    @Size(min = 8, max = 50)
    String password,

    @NotBlank
    @Size(max = 20)
    String name

) {
}
```

Controller에서는 `@Valid`를 사용한다.

```
@RestController
@RequestMapping("/users")
public class UserController {

    private final UserService userService;

    public UserController(UserService userService) {
        this.userService = userService;
    }

    @PostMapping
    public UserResponse createUser(
            @Valid @RequestBody UserCreateRequest request
    ) {
        return userService.createUser(request);
    }
}
```

여기까지는 입력값 자체에 대한 검증이다.

그렇다면 이런 질문은 어디에서 확인해야 할까?

> 이미 가입된 이메일인가?

이것은 DTO만 봐서는 알 수 없다.

DB를 조회해야 한다.

따라서 Service에서 확인할 수 있다.

```
@Service
public class UserService {

    private final UserRepository userRepository;

    public UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    public UserResponse createUser(UserCreateRequest request) {

        if (userRepository.existsByEmail(request.email())) {
            throw new DuplicateEmailException(request.email());
        }

        User user = new User(
                request.email(),
                encodePassword(request.password()),
                request.name()
        );

        User savedUser = userRepository.save(user);

        return new UserResponse(
                savedUser.getId(),
                savedUser.getEmail(),
                savedUser.getName()
        );
    }

    private String encodePassword(String password) {
        return "{encoded}" + password;
    }
}
```

이 예제에서는 검증 위치가 다음처럼 나뉜다.

| 검증 | 위치 |
| --- | --- |
| email이 비어 있지 않은가 | Request DTO |
| email 형식이 맞는가 | Request DTO |
| password가 8자 이상인가 | Request DTO |
| name이 비어 있지 않은가 | Request DTO |
| 이미 가입된 email인가 | Service |
| 비밀번호를 암호화해서 저장하는가 | Service |

이렇게 생각하면 Validation 위치를 결정하기 훨씬 쉬워진다.

---

# 게시글 수정에서는 어떻게 할까?

이번에는 게시글 수정 API를 생각해보자.

요청 DTO는 다음과 같다.

```
public record PostUpdateRequest(

    @NotBlank
    @Size(max = 100)
    String title,

    @NotBlank
    String content

) {
}
```

제목과 내용이 비어 있는지는 Request DTO에서 바로 확인할 수 있다.

하지만 다음 검증은 어떨까?

> 현재 로그인한 사용자가 이 게시글의 작성자인가?

이것은 요청 DTO만으로 판단할 수 없다.

게시글 정보를 조회해야 하고, 현재 사용자 정보와 비교해야 한다.

```
public PostResponse updatePost(
        Long postId,
        Long currentUserId,
        PostUpdateRequest request
) {

    Post post = postRepository.findById(postId)
            .orElseThrow(() -> new PostNotFoundException(postId));

    if (!post.isWrittenBy(currentUserId)) {
        throw new ForbiddenPostAccessException(postId, currentUserId);
    }

    post.update(
            request.title(),
            request.content()
    );

    return PostResponse.from(post);
}
```

이 검증은 단순 입력값 검사가 아니다.

“게시글은 작성자만 수정할 수 있다”라는 **비즈니스 규칙**이다.

---

# 비유해서 이해해보기

Validation을 공연장 입장 과정으로 생각하면 조금 더 이해하기 쉽다.

공연장 입구에서는 먼저 기본적인 것을 확인한다.

- 티켓이 있는가?
- 날짜가 적혀 있는가?
- 좌석 번호가 있는가?

이런 것은 티켓만 보면 바로 확인할 수 있다.

백엔드에서는 Request DTO Validation과 비슷하다.

```
Request
   ↓
@NotBlank
@Email
@Size
@Min
```

하지만 공연장 안으로 들어가려면 추가 확인이 필요할 수도 있다.

- 이미 사용된 티켓인가?
- VIP 구역에 들어갈 권한이 있는가?
- 해당 공연이 취소된 상태는 아닌가?

이런 것은 티켓만 보고 판단하기 어렵다.

시스템 내부의 정보를 확인해야 한다.

백엔드에서는 Service나 Domain의 비즈니스 검증과 비슷하다.

즉,

```
외부 요청
   ↓
입력 형식 검증
   ↓
비즈니스 규칙 검증
   ↓
DB 저장
```

처럼 단계별로 잘못된 요청을 걸러낸다고 생각하면 된다.

---

# Frontend에서도 Validation을 하는데 Backend에서 또 해야 할까?

프론트엔드에서도 입력값 검증을 할 수 있다.

예를 들어 회원가입 화면에서 이메일 형식을 바로 확인할 수 있다.

```
if (!email.includes("@")) {
    alert("이메일 형식이 올바르지 않습니다.");
}
```

사용자는 서버에 요청을 보내기 전에 잘못된 값을 확인할 수 있기 때문에 사용자 경험이 좋아진다.

하지만 프론트엔드 Validation만 믿으면 안 된다.

프론트엔드는 우회할 수 있기 때문이다.

사용자가 직접 HTTP 요청을 보내면 된다.

```
POST /users HTTP/1.1
Content-Type: application/json
```

```
{
  "email": "not-email",
  "password": "1",
  "name": ""
}
```

따라서 프론트엔드에서 검증하더라도 백엔드 Validation은 반드시 필요하다.

나는 둘의 역할을 이렇게 구분하는 편이다.

> Frontend Validation은 사용자에게 빠르게 알려주기 위한 검증이고,
> Backend Validation은 시스템과 데이터를 보호하기 위한 최종 검증이다.

같은 규칙이 프론트엔드와 백엔드에 중복될 수 있지만 문제가 아니다.

백엔드는 프론트엔드를 신뢰해서는 안 된다.

---

# 그렇다면 Controller와 Service 중 어디에서 검증해야 할까?

다음 기준으로 나누면 비교적 쉽게 결정할 수 있다.

## Request DTO / Controller에서 하기 좋은 검증

- 필수 값 여부
- 문자열 길이
- 숫자 범위
- 이메일 형식
- 값의 기본적인 패턴
- enum으로 변환 가능한 값인지
- 요청 데이터의 기본 구조

대표적으로 다음과 같은 Annotation을 사용할 수 있다.

```
@NotNull
@NotBlank
@Size
@Min
@Max
@Email
@Pattern
@Positive
```

---

## Service에서 하기 좋은 검증

다음과 같이 시스템 상태가 필요한 검증이다.

- DB에 존재하는 값인가?
- 이미 중복된 데이터인가?
- 현재 사용자에게 권한이 있는가?
- 현재 상태에서 수행 가능한 요청인가?
- 여러 값 사이의 비즈니스 관계가 올바른가?
- 다른 도메인 정보를 확인해야 하는가?
- 외부 API의 상태를 확인해야 하는가?

한 문장으로 줄이면 다음과 같다.

> 요청 값 하나만 보고 판단할 수 없다면 비즈니스 검증일 가능성이 높다.

---

# 모든 비즈니스 검증을 Service에 넣어야 할까?

여기서 한 단계 더 생각해볼 수 있다.

초기에는 Service에 다음과 같은 코드가 들어갈 수 있다.

```
if (!order.canCancel()) {
    throw new OrderCannotCancelException(order.getId());
}
```

이 정도는 충분히 자연스럽다.

하지만 서비스가 커지면서 같은 검증이 여러 곳에서 반복되기 시작할 수 있다.

```
if (order.getStatus() != OrderStatus.PAID) {
    ...
}
```

이 코드가 여러 Service에 복사된다면 문제가 생긴다.

나중에 주문 취소 정책이 변경됐을 때 모든 코드를 찾아 수정해야 하기 때문이다.

이런 규칙은 도메인 객체가 직접 판단하도록 만들 수도 있다.

```
public void cancel() {

    if (!canCancel()) {
        throw new OrderCannotCancelException(id);
    }

    this.status = OrderStatus.CANCELED;
}
```

또는 복잡한 정책이라면 별도의 정책 객체로 분리할 수도 있다.

즉,

> “비즈니스 검증은 Service에서 한다.”

라고 외우기보다는

> “비즈니스 규칙은 가능한 한 한 곳에서 관리한다.”

라고 이해하는 것이 더 좋다.

Service가 그 역할을 할 수도 있고, Domain 객체나 별도의 Policy 객체가 맡을 수도 있다.

---

# DB Validation은 필요 없을까?

애플리케이션에서 검증한다고 해서 DB 제약조건이 필요 없는 것은 아니다.

오히려 DB의 제약조건은 매우 중요하다.

예를 들어 회원 이메일은 중복되면 안 된다고 해보자.

Service에서 다음 검증을 할 수 있다.

```
if (userRepository.existsByEmail(request.email())) {
    throw new DuplicateEmailException(request.email());
}
```

하지만 동시에 두 요청이 들어온다면 어떻게 될까?

두 요청 모두 `existsByEmail()`을 실행했을 때 아직 데이터가 저장되지 않았다면 둘 다 `false`를 받을 수 있다.

그 이후 두 요청이 동시에 INSERT를 시도할 수도 있다.

따라서 DB에도 `UNIQUE` 제약조건이 있어야 한다.

```
ALTER TABLE users
ADD CONSTRAINT uk_users_email UNIQUE (email);
```

이렇게 보면 애플리케이션 검증과 DB 제약조건은 역할이 조금 다르다.

애플리케이션 Validation은

- 비즈니스 흐름을 명확하게 만들고
- 사용자에게 이해하기 좋은 오류를 제공한다.

DB Constraint는

- 어떤 상황에서도 데이터 무결성이 깨지지 않도록 마지막으로 방어한다.

따라서 둘 중 하나만 선택하는 문제가 아니다.

```
Application Validation
        ↓
비즈니스적으로 요청을 검증
        ↓
Database Constraint
        ↓
최종 데이터 무결성 보장
```

이라고 생각하는 것이 좋다.

---

# Validation 실패 응답도 설계해야 한다

Validation을 추가했다면 실패했을 때 어떤 응답을 줄지도 생각해야 한다.

예를 들어 이메일과 비밀번호 Validation이 실패했다면 다음과 같은 응답을 사용할 수 있다.

```
{
  "code": "VALIDATION_ERROR",
  "message": "입력값이 올바르지 않습니다.",
  "errors": [
    {
      "field": "email",
      "message": "이메일 형식이 올바르지 않습니다."
    },
    {
      "field": "password",
      "message": "비밀번호는 최소 8자 이상이어야 합니다."
    }
  ]
}
```

HTTP Status는 일반적으로 `400 Bad Request`를 사용할 수 있다.

```
HTTP/1.1 400 Bad Request
Content-Type: application/json
```

여기서 중요한 것은 **Validation 실패 응답의 형태를 일관되게 유지하는 것**이다.

프론트엔드는 `field` 값을 보고 어떤 입력창에 오류 메시지를 보여줄지 결정할 수 있다.

```
field: email
→ 이메일 입력창 아래에 메시지 표시

field: password
→ 비밀번호 입력창 아래에 메시지 표시
```

Validation은 서버 내부에서 예외를 발생시키는 것으로 끝나는 것이 아니다.

클라이언트가 오류를 어떻게 처리할지도 함께 고려해야 한다.

---

# 실무에서 조심할 점

## 1. 모든 검증을 Annotation으로 해결하려고 하지 말자

`@NotBlank`, `@Size`, `@Email` 같은 Annotation은 매우 편리하다.

하지만 복잡한 비즈니스 규칙까지 억지로 Annotation으로 만들 필요는 없다.

예를 들어

> “VIP 회원이면서 최근 30일 동안 주문 금액이 10만 원 이상이고 특정 프로모션 기간에만 쿠폰을 사용할 수 있다.”

같은 정책을 DTO Annotation에 넣기 시작하면 오히려 코드를 이해하기 어려워질 수 있다.

Annotation은 단순한 입력 검증에 사용하는 것이 가장 이해하기 쉽다.

---

## 2. DTO Validation이 통과했다고 요청이 정상인 것은 아니다

다음 요청은 형식적으로는 완벽할 수 있다.

```
{
  "email": "test@example.com",
  "password": "password123",
  "name": "홍길동"
}
```

하지만 이미 가입된 이메일이라면 회원가입은 실패해야 한다.

즉,

```
형식적으로 올바른 요청
≠
비즈니스적으로 올바른 요청
```

이다.

---

## 3. DB Constraint만 믿지 말자

DB에서 `UNIQUE` 제약조건 오류가 발생한 뒤 그것만 클라이언트에 그대로 전달할 수도 있다.

하지만 애플리케이션에서 미리 의미 있는 검증을 하면 다음과 같이 더 이해하기 좋은 오류를 줄 수 있다.

```
이미 사용 중인 이메일입니다.
```

따라서 DB Constraint는 마지막 방어선으로 두고, 필요한 비즈니스 검증은 애플리케이션에서도 수행하는 것이 좋다.

---

## 4. 같은 비즈니스 검증을 여러 곳에 복사하지 말자

예를 들어 “주문 취소 가능 여부”를 여러 Service에서 각각 구현한다고 해보자.

```
if (order.getStatus() == OrderStatus.PAID) {
    ...
}
```

비슷한 코드가 곳곳에 생기면 정책 변경이 어려워진다.

가능하면 다음처럼 한 곳에 모은다.

```
if (!order.canCancel()) {
    throw new OrderCannotCancelException(order.getId());
}
```

비즈니스 규칙은 가능한 한 한 곳에서 관리하는 것이 좋다.

---

## 5. Validation 메시지에 너무 많은 내부 정보를 노출하지 말자

특히 인증이나 보안과 관련된 검증에서는 오류 메시지도 신경 써야 한다.

예를 들어 다음 메시지는 좋지 않을 수 있다.

```
비밀번호는 맞지만 관리자 권한이 없습니다.
```

공격자에게 불필요한 정보를 알려줄 수 있기 때문이다.

상황에 따라 다음처럼 표현할 수 있다.

```
인증 정보가 올바르지 않습니다.
```

사용자에게 필요한 정보와 외부에 노출해서는 안 되는 정보를 구분해야 한다.

---

# Validation 위치를 결정하는 간단한 기준

API를 만들 때 다음 질문을 순서대로 해보면 도움이 된다.

### 1. 이 값 하나만 보고 정상 여부를 판단할 수 있는가?

그렇다면 Request DTO Validation으로 처리하기 좋다.

```
@NotBlank
@Size
@Email
@Min
```

### 2. DB를 조회해야 판단할 수 있는가?

그렇다면 비즈니스 검증일 가능성이 높다.

```
중복 이메일
재고
존재 여부
```

### 3. 현재 사용자나 권한 정보가 필요한가?

Service나 보안 계층에서 처리해야 할 가능성이 높다.

```
게시글 수정 권한
관리자 권한
리소스 소유권
```

### 4. 현재 객체 상태에 따라 가능한 행동이 달라지는가?

Domain 규칙으로 관리할 수 있다.

```
결제 완료 주문만 환불 가능
배송 시작 전 주문만 취소 가능
```

### 5. 어떤 경우에도 깨지면 안 되는 데이터 규칙인가?

DB Constraint도 함께 설정한다.

```
UNIQUE
FOREIGN KEY
NOT NULL
CHECK
```

---

# 정리

Validation은 Controller와 Service 중 하나를 선택하는 문제가 아니다.

검증의 종류에 따라 역할을 나누는 것이 중요하다.

```
Frontend
    ↓
사용자에게 빠른 피드백

Request DTO / Controller
    ↓
형식, 필수값, 길이, 범위 검증

Service / Domain
    ↓
DB 상태, 권한, 상태 변경, 비즈니스 정책 검증

Database
    ↓
최종 데이터 무결성 보장
```

처음에는 `@Valid`를 붙이면 Validation이 끝난다고 생각하기 쉽다.

하지만 실제 백엔드에서는 **입력값 자체가 올바른지 확인하는 검증**과 **현재 시스템에서 그 요청이 허용되는지 확인하는 검증**을 구분해야 한다.

나는 Validation을 다음처럼 이해하고 있다.

> Validation은 잘못된 요청이 시스템 깊은 곳까지 들어가지 않도록 여러 단계에서 막아주는 안전장치다.

API를 만들면서 Validation 위치가 고민된다면 다음 질문 하나부터 시작해보자.

> “이 검증은 요청 값만 보고 판단할 수 있는가?”

그렇다면 Request DTO에 둘 가능성이 높다.

반대로 DB, 사용자 권한, 현재 상태, 비즈니스 정책을 알아야 판단할 수 있다면 Service나 Domain에서 처리할 가능성이 높다.

이 기준만 잡아도 Validation 코드를 어디에 두어야 할지 훨씬 명확해진다.
