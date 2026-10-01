---
title: "Spring Boot 예외 처리, 왜 한 곳에서 관리할까?"
date: 2026-08-10 09:45:35 +0900
last_modified_at: 2026-10-01
categories: [Backend, Spring]
tags: [exception, controller-advice, spring]
source_url: "https://velog.io/@joker901010/Spring-Boot-%EC%98%88%EC%99%B8-%EC%B2%98%EB%A6%AC-%EC%99%9C-%ED%95%9C-%EA%B3%B3%EC%97%90%EC%84%9C-%EA%B4%80%EB%A6%AC%ED%95%A0%EA%B9%8C"
---
백엔드 API를 만들다 보면 성공하는 경우만큼이나 실패하는 경우도 많이 생긴다.

예를 들어 이런 상황들이다.

- 존재하지 않는 사용자를 조회했다.
- 이미 가입된 이메일로 회원가입을 시도했다.
- 이메일 형식이 잘못됐다.
- 로그인하지 않은 사용자가 인증이 필요한 API를 호출했다.
- 서버에서 예상하지 못한 오류가 발생했다.

이런 상황에서 서버는 단순히 `"에러가 발생했습니다"`라고 응답하는 것이 아니라,

**어떤 문제가 발생했는지에 맞는 HTTP 상태 코드와 에러 응답을 내려줘야 한다.**

그런데 이 작업을 Controller마다 직접 처리하면 코드가 금방 복잡해진다.

Spring에서는 이런 문제를 해결하기 위해 예외 처리를 **한곳에 모아서 관리하는 방법**을 제공한다.

---

## 먼저 결론부터

Spring Boot에서 예외 처리는 보통 다음과 같이 역할을 나눈다.

```
Controller
    ↓
Service
    ↓
문제 발생
    ↓
예외 발생
    ↓
GlobalExceptionHandler
    ↓
HTTP 상태 코드 + 에러 응답
```

예를 들어 사용자가 존재하지 않는 상황이라면,

```
Service
"사용자가 없네?"
    ↓
UserNotFoundException 발생
    ↓
GlobalExceptionHandler
    ↓
404 Not Found

{
  "code": "USER_NOT_FOUND",
  "message": "사용자를 찾을 수 없습니다."
}
```

이런 식으로 처리할 수 있다.

핵심은 간단하다.

> **비즈니스 코드에서는 문제가 발생했다는 사실만 알리고,
> 그 문제를 어떤 HTTP 응답으로 바꿀지는 공통 예외 처리기가 담당한다.**

---

# 1. 예외 처리를 왜 따로 해야 할까?

가장 먼저 떠올릴 수 있는 방법은 `try-catch`다.

예를 들어 사용자를 조회하는 API가 있다고 해보자.

```
@GetMapping("/users/{id}")
public ResponseEntity<?> getUser(@PathVariable Long id) {

    try {
        UserResponse response = userService.getUser(id);
        return ResponseEntity.ok(response);

    } catch (UserNotFoundException e) {
        return ResponseEntity
                .status(HttpStatus.NOT_FOUND)
                .body("사용자를 찾을 수 없습니다.");
    }
}
```

이 코드는 문제가 없다.

실제로 정상적으로 동작한다.

사용자가 있으면 `200 OK`를 반환하고,

사용자가 없다면 예외를 잡아서 `404 Not Found`를 반환한다.

그렇다면 이렇게 하면 되는 것 아닐까?

---

## API가 하나라면 괜찮다

문제는 API가 많아졌을 때다.

사용자와 관련된 API만 해도 여러 개가 생길 수 있다.

```
GET    /users/{id}
PUT    /users/{id}
DELETE /users/{id}
GET    /users/{id}/orders
GET    /users/{id}/profile
```

이 API들에서 모두 사용자가 존재하지 않을 수 있다.

그러면 Controller마다 이런 코드가 반복되기 시작한다.

```
try {

} catch (UserNotFoundException e) {

}
```

API가 많아질수록 Controller는 점점 복잡해진다.

더 큰 문제는 **API마다 처리 방식이 달라질 수 있다는 것**이다.

예를 들어 같은 `UserNotFoundException`인데도

```
회원 조회 API     → 404
회원 수정 API     → 400
회원 삭제 API     → 404
```

처럼 개발자가 실수로 서로 다른 상태 코드를 반환할 수도 있다.

에러 응답 형태도 달라질 수 있다.

```
{
  "message": "사용자가 없습니다."
}
```

어떤 API에서는 이렇게 내려주고,

```
{
  "code": "USER_NOT_FOUND",
  "message": "사용자를 찾을 수 없습니다."
}
```

다른 API에서는 이렇게 내려줄 수도 있다.

프론트엔드 입장에서는 API마다 에러 구조가 달라져 처리하기 어려워진다.

---

# 2. 그래서 예외 처리를 한곳에 모은다

이 문제를 해결하는 방법이 **공통 예외 처리**다.

Spring에서는 주로

```
@RestControllerAdvice
```

와

```
@ExceptionHandler
```

를 이용한다.

처음 보면 이름이 어렵지만 역할은 단순하다.

### `@RestControllerAdvice`

여러 Controller에서 발생한 예외를 공통으로 처리할 수 있게 해준다.

### `@ExceptionHandler`

특정 예외가 발생했을 때 어떤 코드를 실행할지 지정한다.

둘을 조합하면 다음과 같은 구조를 만들 수 있다.

```
UserController ─────┐
OrderController ────┤
PaymentController ──┼──→ GlobalExceptionHandler
ProductController ──┤
                    ┘
```

각 Controller가 직접 예외를 처리하지 않고,

**공통 예외 처리 클래스 하나가 여러 Controller의 예외를 처리하는 것**이다.

---

# 3. 식당으로 비유하면 이해하기 쉽다

예외 처리를 식당의 고객 응대라고 생각해보자.

직원들이 문제가 생길 때마다 손님에게 직접 제각각 설명한다고 해보자.

```
주방 직원
"음식이 없습니다."

계산 직원
"ERROR입니다."

홀 직원
"잘못 요청하셨습니다."

매니저
"404입니다."
```

같은 식당인데도 직원마다 설명 방식이 다르다.

손님 입장에서는 혼란스럽다.

그래서 보통 고객 응대 방식을 정해둔다.

```
메뉴 없음        → "해당 메뉴를 찾을 수 없습니다."
잘못된 주문      → "주문 내용을 확인해주세요."
결제 실패        → "결제를 처리할 수 없습니다."
서버 문제        → "잠시 후 다시 시도해주세요."
```

백엔드의 공통 예외 처리도 비슷하다.

```
각 Service
    ↓
문제가 무엇인지 알림
    ↓
GlobalExceptionHandler
    ↓
정해진 형식으로 응답
```

`GlobalExceptionHandler`가 API의 **공통 안내 창구** 역할을 한다고 생각하면 된다.

---

# 4. 실제 코드로 만들어보자

사용자 조회 API를 만들어보자.

사용자가 존재하면 정보를 반환하고,

존재하지 않으면 `404 Not Found`를 반환하는 API다.

전체 흐름은 다음과 같다.

```
GET /users/999
        ↓
UserController
        ↓
UserService
        ↓
DB에서 사용자 검색
        ↓
사용자 없음
        ↓
UserNotFoundException
        ↓
GlobalExceptionHandler
        ↓
404 Not Found
```

하나씩 만들어보자.

---

## 4-1. 사용자 없음 예외 만들기

먼저 사용자를 찾지 못했을 때 사용할 예외를 만든다.

```
public class UserNotFoundException extends RuntimeException {

    public UserNotFoundException(Long userId) {
        super("User not found. userId=" + userId);
    }
}
```

이제 코드 어디에서든

```
throw new UserNotFoundException(userId);
```

라고 하면

> "사용자를 찾지 못했다"

라는 상황을 표현할 수 있다.

---

## 4-2. Service에서 예외 발생시키기

사용자를 조회한다.

```
@Service
public class UserService {

    private final UserRepository userRepository;

    public UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    public UserResponse getUser(Long id) {

        User user = userRepository.findById(id)
                .orElseThrow(() -> new UserNotFoundException(id));

        return new UserResponse(
                user.getId(),
                user.getName()
        );
    }
}
```

여기서 중요한 부분은 이것이다.

```
.orElseThrow(() -> new UserNotFoundException(id));
```

사용자가 있으면 그대로 가져온다.

사용자가 없다면

```
UserNotFoundException
```

이 발생한다.

Service에서는 아직

```
404를 반환할까?
400을 반환할까?
어떤 JSON을 내려줄까?
```

같은 것을 고민하지 않는다.

Service는 단지

> "사용자가 존재하지 않는다."

라는 **비즈니스 상황을 예외로 표현한다.**

---

# 5. Controller는 더 단순해진다

Controller를 보자.

```
@RestController
@RequestMapping("/users")
public class UserController {

    private final UserService userService;

    public UserController(UserService userService) {
        this.userService = userService;
    }

    @GetMapping("/{id}")
    public UserResponse getUser(@PathVariable Long id) {
        return userService.getUser(id);
    }
}
```

`try-catch`가 없다.

Controller는 그냥

```
요청을 받고
→ Service를 호출하고
→ 결과를 반환한다.
```

라는 자신의 역할에 집중한다.

그런데 Service에서 `UserNotFoundException`이 발생하면 어떻게 될까?

그때 공통 예외 처리기가 등장한다.

---

# 6. GlobalExceptionHandler 만들기

다음과 같이 만들 수 있다.

```
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(UserNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleUserNotFound(
            UserNotFoundException e
    ) {

        ErrorResponse response = new ErrorResponse(
                "USER_NOT_FOUND",
                "사용자를 찾을 수 없습니다."
        );

        return ResponseEntity
                .status(HttpStatus.NOT_FOUND)
                .body(response);
    }
}
```

여기서

```
@ExceptionHandler(UserNotFoundException.class)
```

의 의미는

> `UserNotFoundException`이 발생하면 이 메서드가 처리해라.

라는 뜻이다.

그리고

```
HttpStatus.NOT_FOUND
```

를 사용했으므로 HTTP 상태 코드는

```
404 Not Found
```

가 된다.

---

# 7. 에러 응답 객체도 하나 만들어두자

에러 응답을 문자열 하나로 보내기보다 일정한 JSON 형태로 만들어두는 것이 편하다.

```
public record ErrorResponse(
        String code,
        String message
) {
}
```

이제 실제 응답은 다음처럼 내려간다.

```
HTTP/1.1 404 Not Found
Content-Type: application/json
```

```
{
  "code": "USER_NOT_FOUND",
  "message": "사용자를 찾을 수 없습니다."
}
```

여기서 각각의 역할도 구분해서 보면 좋다.

```
404
→ HTTP 수준에서 어떤 종류의 실패인지 표현

USER_NOT_FOUND
→ 우리 서비스에서 정의한 구체적인 실패 이유

사용자를 찾을 수 없습니다.
→ 사람이 읽을 수 있는 설명
```

---

# 8. 전체 흐름을 다시 보면

처음에는 여러 코드가 나와서 복잡해 보일 수 있다.

하지만 실제 흐름은 단순하다.

```
① 클라이언트 요청

GET /users/999

        ↓

② Controller

userService.getUser(999)

        ↓

③ Service

DB에서 사용자 검색

        ↓

④ 사용자 없음

throw new UserNotFoundException(999)

        ↓

⑤ GlobalExceptionHandler

UserNotFoundException 처리

        ↓

⑥ HTTP 응답

404 Not Found

{
  "code": "USER_NOT_FOUND",
  "message": "사용자를 찾을 수 없습니다."
}
```

이 구조를 이해하는 것이 가장 중요하다.

---

# 9. 예외 처리를 모으면 무엇이 좋아질까?

## 첫 번째, Controller가 깔끔해진다

공통 예외 처리를 사용하지 않으면 Controller에 이런 코드가 반복된다.

```
try {

} catch (...) {

}
```

공통 예외 처리를 사용하면 Controller는 자신의 역할에 집중할 수 있다.

```
@GetMapping("/{id}")
public UserResponse getUser(@PathVariable Long id) {
    return userService.getUser(id);
}
```

---

## 두 번째, 같은 문제는 같은 응답을 줄 수 있다

`UserNotFoundException`은 여러 API에서 발생할 수 있다.

```
사용자 조회
사용자 수정
사용자 삭제
사용자 주문 조회
```

하지만 모두 같은 예외를 사용한다면

```
UserNotFoundException
        ↓
404 Not Found
```

라는 하나의 정책을 적용할 수 있다.

---

## 세 번째, 에러 응답 형식을 통일할 수 있다

모든 API가 다음 형식을 사용한다고 해보자.

```
{
  "code": "USER_NOT_FOUND",
  "message": "사용자를 찾을 수 없습니다."
}
```

그러면 프론트엔드에서도 에러를 처리하기 쉬워진다.

예를 들어

```
if (error.code === "USER_NOT_FOUND") {
    // 사용자 없음 화면 표시
}
```

처럼 처리할 수 있다.

---

## 네 번째, 나중에 정책을 바꾸기 쉽다

나중에 에러 응답에 `traceId`를 추가해야 할 수도 있다.

```
{
  "code": "USER_NOT_FOUND",
  "message": "사용자를 찾을 수 없습니다.",
  "traceId": "abc-123"
}
```

예외 응답 생성 코드가 여러 Controller에 흩어져 있다면 전부 수정해야 한다.

하지만 공통으로 관리하고 있다면 관련 코드만 수정하면 된다.

---

# 10. 예외마다 상태 코드도 달라야 한다

모든 실패를 똑같이 `500`으로 내려주면 안 된다.

실패에도 종류가 있기 때문이다.

대표적인 HTTP 상태 코드를 간단하게 보면 다음과 같다.

| 상황 | 상태 코드 | 의미 |
| --- | --- | --- |
| 요청값이 잘못됨 | 400 | Bad Request |
| 로그인이 필요함 | 401 | Unauthorized |
| 권한이 없음 | 403 | Forbidden |
| 데이터를 찾을 수 없음 | 404 | Not Found |
| 데이터가 충돌하거나 중복됨 | 409 | Conflict |
| 예상하지 못한 서버 오류 | 500 | Internal Server Error |

예를 들어 존재하지 않는 사용자를 조회했다면

```
404 Not Found
```

가 자연스럽다.

이미 가입된 이메일이라면

```
409 Conflict
```

를 사용할 수 있다.

서버 코드 자체에서 예상하지 못한 오류가 발생했다면

```
500 Internal Server Error
```

를 사용할 수 있다.

---

# 11. 중복 이메일도 같은 방식으로 처리할 수 있다

회원가입할 때 이미 사용 중인 이메일이 입력됐다고 해보자.

먼저 예외를 만든다.

```
public class DuplicateEmailException extends RuntimeException {

    public DuplicateEmailException(String email) {
        super("Duplicate email. email=" + email);
    }
}
```

그리고 공통 예외 처리기에 추가한다.

```
@ExceptionHandler(DuplicateEmailException.class)
public ResponseEntity<ErrorResponse> handleDuplicateEmail(
        DuplicateEmailException e
) {

    return ResponseEntity
            .status(HttpStatus.CONFLICT)
            .body(new ErrorResponse(
                    "DUPLICATE_EMAIL",
                    "이미 가입된 이메일입니다."
            ));
}
```

그러면 응답은 다음처럼 내려갈 수 있다.

```
HTTP/1.1 409 Conflict
```

```
{
  "code": "DUPLICATE_EMAIL",
  "message": "이미 가입된 이메일입니다."
}
```

예외가 하나 더 추가됐지만 Controller를 수정할 필요는 없다.

---

# 12. Validation 실패도 공통으로 처리할 수 있다

예외 처리는 우리가 직접 만든 예외에만 사용하는 것이 아니다.

Spring에서 요청값 검증에 실패했을 때 발생하는 예외도 공통으로 처리할 수 있다.

예를 들어 회원가입 요청 DTO가 있다고 해보자.

```
public record UserCreateRequest(

        @NotBlank
        @Email
        String email,

        @NotBlank
        @Size(min = 8)
        String password
) {
}
```

Controller에서는 `@Valid`를 사용한다.

```
@PostMapping("/users")
public UserResponse createUser(
        @Valid @RequestBody UserCreateRequest request
) {
    return userService.createUser(request);
}
```

사용자가 다음과 같이 요청했다고 해보자.

```
{
  "email": "abc",
  "password": "123"
}
```

이메일 형식도 잘못됐고 비밀번호도 너무 짧다.

이 경우 Spring의 Validation 과정에서 검증 실패 예외가 발생한다.

---

## Validation 예외도 GlobalExceptionHandler에서 처리한다

예를 들어 다음과 같이 처리할 수 있다.

```
@ExceptionHandler(MethodArgumentNotValidException.class)
public ResponseEntity<ValidationErrorResponse> handleValidation(
        MethodArgumentNotValidException e
) {

    List<FieldErrorResponse> errors = e.getBindingResult()
            .getFieldErrors()
            .stream()
            .map(error -> new FieldErrorResponse(
                    error.getField(),
                    error.getDefaultMessage()
            ))
            .toList();

    ValidationErrorResponse response =
            new ValidationErrorResponse(
                    "VALIDATION_ERROR",
                    "입력값을 확인해주세요.",
                    errors
            );

    return ResponseEntity
            .badRequest()
            .body(response);
}
```

필드 에러 객체는 이렇게 만들 수 있다.

```
public record FieldErrorResponse(
        String field,
        String message
) {
}
```

전체 Validation 응답은 이렇게 만들 수 있다.

```
public record ValidationErrorResponse(
        String code,
        String message,
        List<FieldErrorResponse> errors
) {
}
```

그러면 응답을 다음과 같이 통일할 수 있다.

```
{
  "code": "VALIDATION_ERROR",
  "message": "입력값을 확인해주세요.",
  "errors": [
    {
      "field": "email",
      "message": "올바른 이메일 형식이어야 합니다."
    },
    {
      "field": "password",
      "message":일할 수 있다.

```json
{
  "code": "VALIDATION_ERROR",
  "message": "입 "8자 이상 입력해주세요."
    }
  ]
}
```

프론트엔드는 `errors`를 보고 각 입력창 아래에 에러 메시지를 표시할 수 있다.

---

# 13. 그러면 모든 Exception을 하나로 처리하면 되지 않을까?

이런 코드를 작성할 수도 있다.

```
@ExceptionHandler(Exception.class)
public ResponseEntity<ErrorResponse> handleException(Exception e) {

    return ResponseEntity
            .status(HttpStatus.INTERNAL_SERVER_ERROR)
            .body(new ErrorResponse(
                    "ERROR",
                    "오류가 발생했습니다."
            ));
}
```

얼핏 보면 모든 예외를 한 번에 처리하니 편해 보인다.

하지만 문제가 있다.

예를 들어

```
사용자가 없음
이메일 중복
Validation 실패
권한 없음
```

같은 상황까지 전부

```
500 Internal Server Error
```

가 되어버릴 수 있다.

하지만 이것들은 모두 서버가 고장 난 상황은 아니다.

따라서 예측 가능한 예외는 의미에 맞게 따로 처리하는 편이 좋다.

```
UserNotFoundException
→ 404

DuplicateEmailException
→ 409

MethodArgumentNotValidException
→ 400
```

그리고 정말 예상하지 못한 예외만 마지막에

```
Exception
→ 500
```

으로 처리한다.

---

# 14. Exception.class는 마지막 안전망으로 사용한다

다음과 같은 Handler를 마지막에 둘 수 있다.

```
@ExceptionHandler(Exception.class)
public ResponseEntity<ErrorResponse> handleUnexpected(
        Exception e
) {

    log.error("Unexpected exception", e);

    return ResponseEntity
            .status(HttpStatus.INTERNAL_SERVER_ERROR)
            .body(new ErrorResponse(
                    "INTERNAL_SERVER_ERROR",
                    "서버 내부 오류가 발생했습니다."
            ));
}
```

이 Handler의 목적은

> **개발자가 예상하지 못한 오류가 발생했을 때 사용하는 마지막 안전망**

이라고 생각하면 된다.

예측할 수 있는 비즈니스 예외까지 모두 여기에서 처리하는 것은 좋은 구조가 아니다.

---

# 15. 사용자에게 모든 에러 내용을 보여주면 안 된다

예외에는 개발자가 문제를 찾기 위해 필요한 자세한 정보가 들어갈 수 있다.

예를 들어 서버 로그에는 다음과 같은 정보가 필요할 수 있다.

```
userId
orderId
실패한 API
발생한 Exception
stack trace
```

하지만 이 정보를 그대로 사용자에게 내려주는 것은 좋지 않다.

예를 들어 이런 응답은 피하는 것이 좋다.

```
{
  "code": "USER_NOT_FOUND",
  "message": "User not found. userId=999, query=select * from users where id=999"
}
```

DB 쿼리나 내부 구현 정보가 외부에 노출되고 있다.

사용자에게는 필요한 정보만 전달한다.

```
{
  "code": "USER_NOT_FOUND",
  "message": "사용자를 찾을 수 없습니다."
}
```

그리고 개발자가 확인해야 하는 자세한 내용은 서버 로그에 남긴다.

```
사용자 응답
→ 사용자가 다음 행동을 판단하는 데 필요한 정보

서버 로그
→ 개발자가 문제의 원인을 찾는 데 필요한 정보
```

이 둘은 목적이 다르다.

---

# 16. 예외를 잡고 그냥 무시하면 안 된다

다음 코드를 보자.

```
try {
    paymentService.pay(orderId);

} catch (Exception e) {
    log.error("Payment failed", e);
}
```

에러 로그는 남겼다.

하지만 문제는 그다음이다.

예외를 처리한 뒤 아무 일도 없었던 것처럼 코드가 계속 진행될 수 있다.

호출한 쪽에서는 결제가 실패했다는 사실을 모를 수도 있다.

이런 상황을 흔히

> **예외를 삼켰다**

라고 표현한다.

예외를 잡았다면 다음 중 하나는 해야 한다.

```
① 문제를 실제로 복구한다.

또는

② 적절한 예외를 다시 발생시킨다.

또는

③ 호출한 쪽에서 실패임을 알 수 있도록 처리한다.
```

단순히 로그만 남기고 무시하는 것은 주의해야 한다.

---

# 17. 에러 코드는 프론트엔드와의 약속이 될 수 있다

에러 응답에 이런 값을 사용하고 있다고 해보자.

```
USER_NOT_FOUND
DUPLICATE_EMAIL
VALIDATION_ERROR
ACCESS_DENIED
```

프론트엔드는 이 값을 이용해서 화면 동작을 결정할 수 있다.

예를 들어

```
USER_NOT_FOUND
→ "존재하지 않는 사용자입니다." 표시

DUPLICATE_EMAIL
→ 이메일 입력창에 중복 메시지 표시

ACCESS_DENIED
→ 접근 권한 없음 화면으로 이동
```

즉,

```
{
  "code": "USER_NOT_FOUND"
}
```

의 `code`는 단순한 문자열이 아니라 **프론트엔드와 백엔드 사이의 약속**이 될 수 있다.

따라서 한번 사용하기 시작한 에러 코드를 특별한 이유 없이 자주 변경하지 않는 것이 좋다.

---

# 18. Spring은 실제로 어떻게 예외를 처리할까?

여기까지 이해했다면 Spring 내부 동작도 조금 살펴볼 수 있다.

Controller에서 예외가 발생했다고 해서 우리가 직접

```
globalExceptionHandler.handle(...)
```

를 호출하는 것은 아니다.

Spring이 요청을 처리하는 과정에서 예외를 발견하고 적절한 예외 처리기를 찾아준다.

아주 단순화하면 다음과 같다.

```
HTTP 요청
    ↓
Controller 실행
    ↓
Service 실행
    ↓
예외 발생
    ↓
Spring이 예외 처리 방법 탐색
    ↓
@ExceptionHandler 발견
    ↓
해당 메서드 실행
    ↓
HTTP 에러 응답 생성
```

그래서 Controller에서는 다음 코드만 있어도 된다.

```
@GetMapping("/{id}")
public UserResponse getUser(@PathVariable Long id) {
    return userService.getUser(id);
}
```

중간에 예외가 발생하면 Spring의 예외 처리 시스템이 공통 Handler로 연결해주는 것이다.

처음 공부할 때는 Spring 내부의 세부 구현까지 외울 필요는 없다.

우선

> **Controller 밖으로 나온 예외를 Spring이 적절한 `@ExceptionHandler`와 연결해준다.**

정도로 이해하면 충분하다.

---

# 19. `@ExceptionHandler`와 `@RestControllerAdvice` 차이

둘이 비슷해 보여 처음에는 헷갈릴 수 있다.

### `@ExceptionHandler`

특정 예외를 처리하는 메서드를 지정한다.

```
@ExceptionHandler(UserNotFoundException.class)
public ResponseEntity<ErrorResponse> handleUserNotFound(
        UserNotFoundException e
) {
    ...
}
```

---

### `@RestControllerAdvice`

여러 Controller에서 사용할 공통 예외 처리 클래스를 만든다.

```
@RestControllerAdvice
public class GlobalExceptionHandler {

}
```

둘을 같이 사용하면

```
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(UserNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleUserNotFound(
            UserNotFoundException e
    ) {
        ...
    }
}
```

와 같은 구조가 된다.

쉽게 기억하면 된다.

```
@RestControllerAdvice
→ "여러 Controller의 예외를 여기서 처리할게."

@ExceptionHandler
→ "이 예외가 발생하면 이 메서드를 실행해."
```

---

# 20. 처음에는 이 정도 구조면 충분하다

예외 처리를 처음 공부한다면 너무 복잡하게 설계할 필요는 없다.

다음 정도로 시작해도 충분하다.

```
exception
├── GlobalExceptionHandler.java
├── UserNotFoundException.java
└── DuplicateEmailException.java

response
└── ErrorResponse.java
```

그리고 역할을 다음처럼 나눈다.

```
Controller
→ 요청을 받고 Service를 호출한다.

Service
→ 비즈니스 로직을 수행한다.
→ 문제가 발생하면 의미 있는 예외를 발생시킨다.

GlobalExceptionHandler
→ 발생한 예외를 HTTP 상태 코드와 에러 응답으로 바꾼다.

ErrorResponse
→ 클라이언트에게 전달할 에러 형식을 정의한다.
```

처음부터 거대한 예외 구조를 만드는 것보다,

필요한 예외가 생길 때 하나씩 추가하는 방식이 이해하기도 쉽고 관리하기도 편하다.

---

# 21. 실무에서 기억할 것

예외 처리를 만들 때는 다음 다섯 가지 정도를 먼저 생각하면 좋다.

### 1. 이건 어떤 종류의 실패인가?

```
잘못된 입력인가?
데이터가 없는 것인가?
중복인가?
권한 문제인가?
서버 자체의 문제인가?
```

### 2. 어떤 HTTP 상태 코드가 자연스러운가?

```
400?
401?
403?
404?
409?
500?
```

### 3. 프론트엔드가 구분해야 하는 에러인가?

구분해야 한다면

```
USER_NOT_FOUND
DUPLICATE_EMAIL
```

같은 에러 코드를 정의할 수 있다.

### 4. 사용자에게 어떤 메시지를 보여줄 것인가?

내부 구현을 노출하지 않으면서 사용자가 상황을 이해할 수 있어야 한다.

### 5. 개발자가 문제를 찾으려면 무엇을 로그로 남겨야 하는가?

사용자 응답과 서버 로그는 목적이 다르다는 것을 기억해야 한다.

---

# 정리

예외 처리를 한곳에서 관리하는 가장 큰 이유는

**실패를 일관된 방식으로 처리하기 위해서다.**

Controller마다 직접 `try-catch`를 작성할 수도 있다.

하지만 API가 많아질수록

```
중복 코드가 생기고
에러 응답이 제각각이 되고
상태 코드 정책도 달라질 수 있다.
```

그래서 보통 다음처럼 역할을 나눈다.

```
Controller
→ 요청과 응답 흐름에 집중

Service
→ 비즈니스 로직 처리
→ 문제가 발생하면 예외 발생

GlobalExceptionHandler
→ 예외를 HTTP 상태 코드와 에러 응답으로 변환
```

예를 들어

```
사용자를 찾을 수 없음
        ↓
UserNotFoundException
        ↓
GlobalExceptionHandler
        ↓
404 Not Found

{
  "code": "USER_NOT_FOUND",
  "message": "사용자를 찾을 수 없습니다."
}
```

라는 하나의 흐름으로 만들 수 있다.

처음에는 `@RestControllerAdvice`나 `@ExceptionHandler`라는 이름이 낯설 수 있다.

하지만 역할만 보면 어렵지 않다.

> **예외는 서버 안에서 발생한 문제이고,
> 예외 처리는 그 문제를 클라이언트가 이해할 수 있는 HTTP 응답으로 바꾸는 과정이다.**

그리고 `GlobalExceptionHandler`는 그 변환 작업을 한곳에서 담당하는 역할이라고 이해하면 된다.

Spring Boot에서 예외 처리를 공부할 때는 우선 이 흐름부터 확실히 이해하는 것이 좋다.

```
문제 발생
→ 예외 발생
→ 공통 예외 처리
→ 상태 코드 + 에러 응답
```

이 흐름이 익숙해지고 나면 Validation 예외, 인증·인가 예외, 공통 에러 코드, 로그 정책처럼 더 복잡한 예외 처리도 훨씬 쉽게 이해할 수 있다.
