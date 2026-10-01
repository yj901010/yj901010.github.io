---
title: "의존성 방향은 왜 중요할까?"
date: 2026-08-21 11:36:17 +0900
last_modified_at: 2026-10-01
categories: [Architecture, Design]
tags: [dependency, architecture, design]
source_url: "https://velog.io/@joker901010/%EC%9D%98%EC%A1%B4%EC%84%B1-%EB%B0%A9%ED%96%A5%EC%9D%80-%EC%99%9C-%EC%A4%91%EC%9A%94%ED%95%A0%EA%B9%8C"
---
Spring으로 백엔드 개발을 시작하면 거의 항상 이런 구조를 만나게 된다.

```
Controller
→ Service
→ Repository
```

처음에는 그냥 이렇게 외우기도 한다.

> Controller에서 Service를 호출하고,
> Service에서 Repository를 호출하면 된다.

그런데 개발하다 보면 조금씩 궁금해진다.

```
왜 꼭 이 방향이어야 하지?

Repository가 Service를 호출하면 안 되나?

Service에서 ResponseEntity를 반환하면 왜 안 좋다고 하지?

인터페이스는 왜 만드는 거지?
```

이 질문들은 결국 하나의 주제로 연결된다.

**의존성 방향이다.**

---

## 한 줄로 먼저 이해하기

의존성 방향을 신경 쓰는 이유는 간단하다.

> 코드 하나가 바뀌었을 때 그 변경이 불필요한 곳까지 퍼지는 것을 막기 위해서다.

예를 들어 API 응답 형식만 바꿨는데 Repository까지 수정해야 한다면 조금 이상하다.

반대로 DB 조회 방법이 바뀌었는데 Controller까지 전부 수정해야 한다면 이것도 좋지 않다.

각 계층이 자신이 알아야 할 것만 알고 있으면 변경 범위를 줄일 수 있다.

---

## Spring이 해주는 것과 개발자가 해야 하는 것

Spring Framework에서는 Dependency Injection, 즉 **의존성 주입**을 제공한다.

예를 들어 `UserController`가 `UserService`를 사용한다고 해보자.

```
@RestController
public class UserController {

    private final UserService userService;

    public UserController(UserService userService) {
        this.userService = userService;
    }
}
```

`UserController`가 직접 `new UserService()`를 하지 않아도 Spring이 필요한 객체를 주입해준다.

하지만 여기서 중요한 점이 하나 있다.

Spring은

```
UserController에게 UserService를 넣어주는 것
```

은 해주지만,

```
UserController가 UserService를 알아야 하는지

UserService가 UserController를 알아야 하는지
```

까지 결정해주지는 않는다.

즉,

```
Spring:
객체 사이의 의존성을 주입해준다.

개발자:
객체 사이의 의존성 방향을 설계한다.
```

라고 이해하면 된다.

---

# 먼저 의존성이 무엇인지부터 보자

A 클래스가 B 클래스를 사용한다면 일반적으로

```
A → B
```

라고 표현한다.

예를 들어 Controller에서 Service를 호출한다.

```
@RestController
public class UserController {

    private final UserService userService;

    public UserController(UserService userService) {
        this.userService = userService;
    }
}
```

그러면

```
UserController → UserService
```

라고 할 수 있다.

Service가 Repository를 사용한다면

```
UserService → UserRepository
```

가 된다.

그래서 흔히 사용하는 구조가 만들어진다.

```
Controller
    ↓
Service
    ↓
Repository
```

---

# 왜 이 방향이 자연스러울까?

회원 조회 API를 하나 생각해보자.

사용자가 다음 요청을 보낸다.

```
GET /users/1
```

요청은 가장 먼저 Controller가 받는다.

Controller는 Service에게

```
1번 회원을 조회해줘.
```

라고 요청한다.

Service는 회원을 찾기 위해 Repository에게

```
DB에서 1번 회원을 찾아줘.
```

라고 요청한다.

전체 흐름은 다음과 같다.

```
HTTP 요청
   ↓
Controller
   ↓
Service
   ↓
Repository
   ↓
Database
```

각 계층이 담당하는 역할도 다르다.

```
Controller
HTTP 요청과 응답 처리

Service
비즈니스 로직 처리

Repository
데이터 접근
```

이 역할을 기준으로 보면 의존성 방향도 자연스럽게 만들어진다.

---

# 회사 업무로 생각하면 조금 더 쉽다

의존성 방향을 회사 업무와 비슷하게 생각해볼 수도 있다.

고객이 창구 직원에게 요청한다.

```
고객
↓
창구 직원
↓
업무 담당자
↓
자료 담당자
```

창구 직원은 고객과 이야기한다.

업무 담당자는 실제 업무를 처리한다.

자료 담당자는 필요한 자료를 찾아준다.

이때 자료 담당자가 갑자기 이런 것을 신경 쓰기 시작한다고 생각해보자.

```
이 자료는 고객 화면에서는 빨간색으로 표시해주세요.

모바일에서는 이름을 두 줄로 표시해주세요.
```

조금 이상하다.

자료 담당자는 자료를 찾아주는 역할이다.

고객 화면이 어떻게 생겼는지는 창구 쪽에서 결정하는 것이 자연스럽다.

백엔드도 비슷하다.

```
Controller
HTTP를 안다.

Service
비즈니스 규칙을 안다.

Repository
데이터 저장 방법을 안다.
```

각 계층은 자신에게 필요한 것만 아는 것이 좋다.

---

# 실제 코드로 살펴보자

회원 조회 기능을 만들어보자.

먼저 Repository가 있다.

```
public interface UserRepository extends JpaRepository<User, Long> {
}
```

Repository는 DB에서 데이터를 조회한다.

Service는 Repository를 사용한다.

```
@Service
public class UserService {

    private final UserRepository userRepository;

    public UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    public User getUser(Long id) {
        return userRepository.findById(id)
                .orElseThrow(() -> new UserNotFoundException(id));
    }
}
```

Service는

```
사용자를 찾는다.

사용자가 없으면 예외를 발생시킨다.
```

라는 업무를 담당한다.

Controller는 Service를 사용한다.

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

        User user = userService.getUser(id);

        return new UserResponse(
                user.getId(),
                user.getName()
        );
    }
}
```

Controller는 Service에서 받은 결과를 HTTP 응답 형식으로 바꾼다.

전체 의존성은 다음과 같다.

```
UserController
      ↓
UserService
      ↓
UserRepository
```

---

# 반대 방향으로 의존하면 어떤 문제가 생길까?

이제 조금 이상한 구조를 생각해보자.

Repository가 API 응답 DTO를 직접 알고 있다고 해보자.

```
public interface UserRepository extends JpaRepository<User, Long> {

    @Query("""
        select new com.example.api.UserResponse(u.id, u.name)
        from User u
        where u.id = :id
    """)
    Optional<UserResponse> findUserResponseById(Long id);
}
```

코드 자체는 동작할 수 있다.

문제는 Repository가 무엇을 알고 있느냐이다.

```
Repository
↓
UserResponse
```

그런데 `UserResponse`가 Controller에서 사용하는 **외부 API 응답 모델**이라면 Repository가 API 구조를 알아버린다.

예를 들어 API 요구사항이 바뀌었다고 해보자.

기존 응답이

```
{
  "id": 1,
  "name": "Lee"
}
```

였는데

```
{
  "id": 1,
  "nickname": "Lee",
  "profileImage": "..."
}
```

로 변경되었다.

단순한 API 응답 변경인데 Repository 쿼리까지 수정해야 할 수도 있다.

즉,

```
API 변경
↓
Repository 변경
```

으로 변경 영향이 퍼진다.

이것이 의존성 방향을 신경 쓰는 이유 중 하나다.

---

# 그렇다면 Repository에서 DTO를 반환하면 무조건 나쁜 걸까?

그렇지는 않다.

실무에서는 조회 성능 때문에 필요한 컬럼만 가져오는 경우가 많다.

예를 들어

```
UserQueryResult
UserSummaryProjection
UserRow
```

같은 **조회 전용 모델**을 만들 수 있다.

```
public record UserQueryResult(
        Long id,
        String name
) {
}
```

Repository가 이 객체를 반환한다.

```
public interface UserRepository extends JpaRepository<User, Long> {

    Optional<UserQueryResult> findUserQueryResultById(Long id);
}
```

이 객체는 API 응답을 위한 객체가 아니라

```
DB 조회 결과를 표현하기 위한 객체
```

다.

즉 중요한 것은

```
Repository에서 DTO를 사용하면 안 된다.
```

가 아니다.

중요한 질문은 이것이다.

> 이 DTO는 어느 계층의 책임을 표현하는 객체인가?

API 응답 객체라면 Repository가 직접 알지 않는 편이 좋다.

조회 전용 Projection이라면 Repository에서 사용하는 것이 자연스러울 수 있다.

---

# Service가 Controller를 알게 되면 어떻게 될까?

이번에는 Service에서 `ResponseEntity`를 반환한다고 해보자.

```
@Service
public class UserService {

    public ResponseEntity<UserResponse> getUser(Long id) {

        // 비즈니스 로직

        return ResponseEntity.ok(
                new UserResponse(id, "Lee")
        );
    }
}
```

처음에는 편해 보인다.

하지만 Service가 이제 HTTP를 알아야 한다.

```
Service
↓
ResponseEntity
↓
HTTP
```

Service가 이런 것까지 처리하기 시작할 수도 있다.

```
return ResponseEntity
        .status(HttpStatus.OK)
        .header("X-Test", "value")
        .body(response);
```

그런데 HTTP 상태 코드나 Header는 비즈니스 규칙일까?

대부분은 아니다.

이것들은 웹 계층의 관심사다.

따라서 Service에서는 비즈니스 결과를 반환하고

```
@Service
public class UserService {

    public User getUser(Long id) {
        // 비즈니스 로직
        return user;
    }
}
```

Controller가 HTTP 응답을 만드는 편이 역할이 명확하다.

```
@GetMapping("/{id}")
public ResponseEntity<UserResponse> getUser(@PathVariable Long id) {

    User user = userService.getUser(id);

    UserResponse response = new UserResponse(
            user.getId(),
            user.getName()
    );

    return ResponseEntity.ok(response);
}
```

이렇게 하면 Service는 HTTP에서 자유로워진다.

---

# 왜 HTTP에서 자유로운 것이 좋을까?

Service를 사용하는 곳이 Controller뿐이라는 보장은 없다.

나중에는 다음과 같은 곳에서도 같은 기능을 사용할 수 있다.

```
REST Controller

Batch

Scheduler

Kafka Consumer

CLI

테스트 코드
```

그런데 Service가 `ResponseEntity`를 반환한다면 어떻게 될까?

```
Batch
→ Service
→ ResponseEntity?
```

이상하다.

Batch 프로그램에서는 HTTP 응답이 필요하지 않다.

반대로 Service가 순수하게 업무 결과만 반환한다면

```
Controller
   ↓

Batch
   ↓
Service

Kafka Consumer
   ↓
```

여러 곳에서 자연스럽게 사용할 수 있다.

---

# 결국 핵심은 변경이 어디까지 퍼지는가이다

예를 들어 API 응답에 필드 하나가 추가되었다.

좋은 구조라면 주로 Controller 또는 API DTO 근처만 바뀐다.

```
API 변경
↓
Controller / Response DTO
```

하지만 Repository가 Response DTO를 알고 있다면

```
API 변경
↓
Response DTO
↓
Repository
```

까지 변경될 수 있다.

비슷하게 DB 접근 기술이 바뀌었다고 생각해보자.

```
JPA
→ MyBatis
```

이 변화가 Controller까지 영향을 준다면 계층 사이의 결합도가 높은 것이다.

가능하면 변경은 관련 있는 영역 안에서 끝나는 것이 좋다.

---

# 순환 의존성도 의존성 방향 문제다

의존성이 복잡해지다 보면 이런 구조가 생기기도 한다.

```
UserService
→ OrderService

OrderService
→ UserService
```

코드로 보면 다음과 같다.

```
@Service
public class UserService {

    private final OrderService orderService;

    public UserService(OrderService orderService) {
        this.orderService = orderService;
    }
}
```

그리고

```
@Service
public class OrderService {

    private final UserService userService;

    public OrderService(UserService userService) {
        this.userService = userService;
    }
}
```

그러면 Spring 입장에서는

```
UserService를 만들려면 OrderService가 필요하다.

OrderService를 만들려면 UserService가 필요하다.
```

라는 상황이 발생한다.

```
UserService
    ↓
OrderService
    ↓
UserService
```

이것이 순환 의존성이다.

---

# 순환 의존성이 생겼다면 먼저 구조를 확인하자

순환 의존성이 생기면 `@Lazy` 등을 사용해서 당장 문제를 피하고 싶을 수 있다.

하지만 먼저 이런 질문을 해보는 것이 좋다.

```
두 Service가 서로 너무 많은 것을 알고 있지 않은가?

공통 로직을 다른 객체로 분리할 수 없는가?

한쪽에서만 다른 Service를 호출하도록 만들 수 없는가?

두 기능을 조정하는 별도의 객체가 필요한 것은 아닌가?
```

예를 들어

```
UserService ↔ OrderService
```

가 아니라

```
OrderFacade
├─ UserService
└─ OrderService
```

처럼 두 작업을 조정하는 역할을 별도로 둘 수도 있다.

항상 이렇게 해야 한다는 의미는 아니다.

중요한 것은

> 순환 의존성이 생겼다면 단순한 Spring 설정 문제가 아니라 역할이 꼬인 것은 아닌지 확인해보자.

라는 것이다.

---

# "인터페이스에 의존하라"는 말은 무슨 뜻일까?

의존성 방향을 공부하다 보면 이런 말을 자주 듣는다.

> 구현체보다 인터페이스에 의존하라.

예를 들어 회원가입 후 이메일을 보낸다고 해보자.

```
@Service
public class SignupService {

    private final EmailSender emailSender;

    public SignupService(EmailSender emailSender) {
        this.emailSender = emailSender;
    }
}
```

현재는 이메일만 보내기 때문에 문제가 없어 보인다.

그런데 나중에 요구사항이 바뀔 수 있다.

```
이메일
→ 문자

이메일
→ Push

이메일
→ Slack
```

이럴 때 Service가 이메일이라는 구체적인 기술에 직접 의존하고 있다면 변경의 영향을 받게 된다.

---

# 역할을 인터페이스로 만들 수 있다

Service 입장에서 정말 필요한 것은

```
이메일을 보내는 것
```

일까?

조금 더 본질적으로 보면

```
회원가입 완료 알림을 보내는 것
```

이다.

그러면 이런 인터페이스를 만들 수 있다.

```
public interface WelcomeNotificationSender {

    void send(String email);
}
```

이메일 구현체는 다음과 같이 만든다.

```
@Component
public class EmailWelcomeNotificationSender
        implements WelcomeNotificationSender {

    @Override
    public void send(String email) {
        // 이메일 전송
    }
}
```

Service는 구현체 대신 인터페이스에 의존한다.

```
@Service
public class SignupService {

    private final WelcomeNotificationSender notificationSender;

    public SignupService(
            WelcomeNotificationSender notificationSender
    ) {
        this.notificationSender = notificationSender;
    }

    public void signup(SignupRequest request) {

        // 회원가입 처리

        notificationSender.send(request.email());
    }
}
```

이제 `SignupService`가 아는 것은

```
Email
```

이 아니라

```
WelcomeNotificationSender
```

라는 역할이다.

---

# 테스트에서도 장점이 생긴다

인터페이스를 사용하면 테스트에서 실제 이메일 발송기를 사용할 필요가 없다.

예를 들어 가짜 구현체를 넣을 수 있다.

```
class FakeWelcomeNotificationSender
        implements WelcomeNotificationSender {

    @Override
    public void send(String email) {
        // 실제 이메일을 보내지 않는다.
    }
}
```

또는 Mockito를 사용할 수도 있다.

```
WelcomeNotificationSender sender =
        mock(WelcomeNotificationSender.class);

SignupService signupService =
        new SignupService(sender);
```

Service는 실제 이메일 서버가 없어도 테스트할 수 있다.

---

# 그렇다고 모든 클래스에 인터페이스를 만들어야 할까?

그것도 아니다.

이런 구조를 볼 때가 있다.

```
UserService
UserServiceImpl

OrderService
OrderServiceImpl

ProductService
ProductServiceImpl
```

구현체가 하나뿐이고 교체 가능성도 없는데 습관적으로 모든 Service에 인터페이스를 만드는 경우다.

인터페이스는 목적이 있을 때 만드는 것이 좋다.

예를 들어 다음과 같은 경우다.

```
구현체가 여러 개 존재할 가능성이 있다.

외부 시스템과 연결되는 부분이다.

구현 기술을 교체할 가능성이 있다.

테스트 대역을 만들 필요가 있다.

도메인과 외부 기술 사이의 경계를 만들고 싶다.
```

인터페이스 자체가 좋은 설계를 만들어주는 것은 아니다.

**어디에 경계를 만들 것인지가 더 중요하다.**

---

# 의존성 방향이 잘 잡히면 테스트도 쉬워진다

다음 Service가 있다고 해보자.

```
@Service
public class UserService {

    private final UserRepository userRepository;

    public UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    public User getUser(Long id) {
        return userRepository.findById(id)
                .orElseThrow(() ->
                        new UserNotFoundException(id));
    }
}
```

테스트에서는 Repository를 Mock으로 바꿀 수 있다.

```
class UserServiceTest {

    private final UserRepository userRepository =
            mock(UserRepository.class);

    private final UserService userService =
            new UserService(userRepository);

    @Test
    void 사용자가_없으면_예외가_발생한다() {

        given(userRepository.findById(1L))
                .willReturn(Optional.empty());

        assertThatThrownBy(
                () -> userService.getUser(1L)
        ).isInstanceOf(UserNotFoundException.class);
    }
}
```

Service 테스트를 위해 실제 DB를 띄울 필요가 없다.

Repository 역할만 Mock으로 대체하면 된다.

---

# 반대로 의존성이 강하게 묶여 있다면?

예를 들어 Service 내부에서 직접 DB 연결을 만든다고 해보자.

```
public class UserService {

    public User getUser(Long id) {

        Connection connection =
                DriverManager.getConnection(...);

        // 직접 SQL 실행
    }
}
```

Service를 테스트하려면 실제 DB 연결까지 필요해질 가능성이 높다.

또는 Service가 HTTP 객체를 직접 받는다고 해보자.

```
public User getUser(
        HttpServletRequest request,
        Long id
) {
}
```

테스트할 때 비즈니스 로직과 상관없는 HTTP 객체까지 준비해야 한다.

의존성 방향을 잘 나누면 테스트할 때 필요한 것만 바꿀 수 있다.

```
Controller 테스트
→ Service 대체

Service 테스트
→ Repository 대체

외부 API 테스트
→ 외부 Client 대체
```

---

# 초보자라면 이것부터 기억하자

처음부터 Clean Architecture나 Hexagonal Architecture까지 전부 이해할 필요는 없다.

Spring 백엔드를 처음 공부한다면 우선 아래 구조부터 익숙해지면 된다.

```
Controller
   ↓
Service
   ↓
Repository
```

그리고 역할을 이렇게 생각하면 된다.

### Controller

```
HTTP 요청을 받는다.

요청 값을 검증한다.

Service를 호출한다.

HTTP 응답을 만든다.
```

### Service

```
업무 규칙을 처리한다.

필요한 Repository나 다른 기능을 사용한다.
```

### Repository

```
DB에서 데이터를 조회한다.

DB에 데이터를 저장한다.
```

---

# 코드를 작성하면서 확인할 질문

새로운 의존성을 추가하려고 할 때 다음 질문을 해보면 좋다.

```
이 클래스가 정말 이 객체를 알아야 할까?
```

예를 들어 Service에서 `ResponseEntity`를 사용하고 있다면

```
Service가 HTTP를 알아야 할까?
```

라고 생각해본다.

Repository가 `UserResponse`를 사용하고 있다면

```
Repository가 API 응답 형식을 알아야 할까?
```

라고 생각해본다.

Entity가 `CreateUserRequest`를 알고 있다면

```
DB Entity가 HTTP Request DTO를 알아야 할까?
```

라고 생각해볼 수 있다.

이 질문 하나만 습관적으로 해도 계층이 뒤섞이는 것을 상당히 줄일 수 있다.

---

# 패키지 구조와도 연결된다

의존성 방향은 패키지 구조와도 관련이 있다.

예를 들어 이런 구조를 사용할 수 있다.

```
user
├── controller
│   └── UserController
│
├── service
│   └── UserService
│
├── repository
│   └── UserRepository
│
└── domain
    └── User
```

조금 더 역할을 나눈 구조라면

```
api
application
domain
infrastructure
```

처럼 만들 수도 있다.

어떤 패키지 구조가 정답인 것은 아니다.

중요한 것은 팀 안에서

```
어느 계층이 어느 계층을 알아도 되는가?
```

라는 규칙을 정하고 유지하는 것이다.

---

# 의존성 방향이 중요한 진짜 이유

결국 의존성 방향은 화살표를 예쁘게 그리는 문제가 아니다.

```
Controller
→ Service
→ Repository
```

를 외우는 것이 목적도 아니다.

더 중요한 것은 다음 질문이다.

```
코드가 변경되었을 때 어디까지 영향을 받을까?

비즈니스 로직이 HTTP나 DB 기술에 너무 묶여 있지는 않을까?

각 클래스가 자신의 역할보다 너무 많은 것을 알고 있지는 않을까?

테스트할 때 필요한 의존성을 쉽게 바꿀 수 있을까?
```

---

# 정리

Spring은 의존성을 주입해준다.

하지만 **누가 누구를 의존해야 하는지는 개발자가 설계해야 한다.**

기본적인 레이어드 아키텍처에서는 보통 다음 방향을 사용한다.

```
Controller
   ↓
Service
   ↓
Repository
```

각 계층의 역할은 대략 다음과 같다.

```
Controller
→ HTTP

Service
→ 비즈니스 로직

Repository
→ 데이터 접근
```

반대로 이런 코드가 보인다면 한 번쯤 생각해볼 필요가 있다.

```
Repository → API Response DTO

Service → ResponseEntity

Service → HttpServletRequest

Entity → Request DTO

Service A ↔ Service B
```

무조건 잘못된 코드라는 뜻은 아니다.

하지만

> 정말 이 계층이 이것을 알아야 하는가?

를 확인해볼 필요가 있다는 신호다.

나는 의존성 방향을 이렇게 이해하면 가장 쉽다고 생각한다.

> **의존성 방향은 코드가 변경되었을 때 그 충격이 어디까지 퍼질지를 결정한다.**

처음에는 단순히

```
Controller
→ Service
→ Repository
```

를 기준으로 시작해도 충분하다.

그리고 코드를 작성하면서 계속 하나만 물어보면 된다.

> **이 클래스가 정말 이 의존성을 알아야 할까?**

이 질문이 익숙해지기 시작하면 단순히 Spring 코드를 작성하는 것을 넘어, 코드의 구조를 설계하기 시작하게 된다.
