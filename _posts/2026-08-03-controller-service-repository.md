---
title: "Controller, Service, Repository는 왜 나눌까?"
date: 2026-08-03 09:00:02 +0900
last_modified_at: 2026-10-01
categories: [Backend, Spring]
tags: [controller, service, repository]
source_url: "https://velog.io/@joker901010/Controller-Service-Repository%EB%8A%94-%EC%99%9C-%EB%82%98%EB%88%8C%EA%B9%8C"
---
Spring으로 처음 API를 만들다 보면 이런 생각이 들 수 있다.

> “Controller에서 요청도 받고, DB도 조회하고, 응답까지 바로 만들면 더 간단하지 않을까?”

실제로 작은 기능 하나만 만들 때는 Controller 안에서 모든 작업을 처리하는 편이 더 빠르게 느껴질 수 있다.

하지만 기능이 늘어나기 시작하면 Controller가 점점 길어지고, HTTP 요청 처리 코드와 비즈니스 로직, 데이터베이스 접근 코드가 한곳에 섞이게 된다.

그래서 Spring 애플리케이션에서는 보통 역할에 따라 코드를 다음과 같이 나눈다.

- Controller: HTTP 요청과 응답 처리
- Service: 비즈니스 로직과 작업 흐름 처리
- Repository: 데이터베이스 접근

한 문장으로 표현하면 다음과 같다.

> Controller는 요청의 입구이고, Service는 판단의 중심이며, Repository는 데이터의 통로다.

---

## Spring에서는 각 계층을 어떻게 표현할까?

Spring에서는 클래스의 역할을 나타내기 위해 다음과 같은 애너테이션을 사용한다.

```
@Controller
@Service
@Repository
```

이 애너테이션들은 모두 `@Component`를 기반으로 한다.

따라서 세 애너테이션 모두 해당 클래스를 Spring이 관리하는 Bean으로 등록할 수 있다.

다만 의미는 서로 다르다.

```
@Controller
웹 요청을 처리하는 클래스

@Service
비즈니스 작업을 처리하는 클래스

@Repository
데이터 저장소에 접근하는 클래스
```

단순히 폴더를 구분하기 위해 사용하는 것이 아니라, 클래스가 어떤 책임을 가지고 있는지 코드에 표현하는 것이다.

---

## Controller에서 모든 일을 처리하면 안 될까?

회원 정보를 조회하는 API를 만든다고 해보자.

처음에는 다음처럼 작성할 수도 있다.

```
@RestController
public class UserController {

    @GetMapping("/users/{id}")
    public UserResponse getUser(@PathVariable Long id) {
        // DB에서 회원 조회
        // 회원 존재 여부 확인
        // 비즈니스 규칙 검사
        // 응답 객체 생성

        return new UserResponse(id, "junior");
    }
}
```

파일 하나만 보면 흐름이 단순해 보인다.

하지만 실제 기능이 추가되면 Controller 안에는 다음과 같은 코드가 함께 들어가게 된다.

- 요청 값 검증
- 데이터베이스 조회
- 회원 존재 여부 확인
- 회원 상태 확인
- 권한 검사
- 응답 객체 변환
- 예외 처리

이런 코드가 계속 쌓이면 Controller가 너무 많은 책임을 가지게 된다.

또한 같은 회원 조회 로직이 다른 곳에서도 필요할 수 있다.

- 관리자 API에서 회원 조회
- 주문 API에서 회원 상태 확인
- 배치 작업에서 회원 조회
- 알림 처리에서 회원 정보 확인

회원 조회 로직이 Controller에 들어 있다면 HTTP 요청과 강하게 묶이기 때문에 다른 기능에서 재사용하기 어렵다.

처음에는 빠르게 작성한 코드가 나중에는 가장 수정하기 어려운 코드가 될 수 있다.

---

## 식당에 비유해서 이해해보기

Controller, Service, Repository의 관계를 식당에 비유해보자.

### Controller는 주문을 받는 직원이다

손님이 주문하면 주문 내용을 확인하고 주방에 전달한다.

```
“회원 1번 정보를 보여주세요.”
```

Controller는 요청에서 회원 번호를 꺼내 Service에 전달한다.

### Service는 주방장이다

주방장은 주문 내용을 보고 실제로 어떤 작업을 해야 하는지 판단한다.

```
“회원이 존재하는지 확인하고,
현재 조회할 수 있는 상태인지 검사한 뒤,
응답에 필요한 정보를 만들어야겠다.”
```

Service는 비즈니스 규칙과 전체 작업 흐름을 담당한다.

### Repository는 재료 창고 담당자다

Repository는 데이터가 저장된 곳에서 필요한 정보를 가져온다.

```
“데이터베이스에서 회원 1번을 조회하겠습니다.”
```

Repository는 데이터베이스와 직접 대화한다.

각자의 역할이 나뉘어 있으면 문제가 생겼을 때 확인해야 할 위치도 명확해진다.

```
URL이나 요청 처리가 이상하다
→ Controller 확인

회원 상태 판단이나 정책이 이상하다
→ Service 확인

조회 쿼리나 DB 저장이 이상하다
→ Repository 확인
```

---

# 회원 조회 API를 계층별로 나눠보기

간단한 회원 조회 API를 Controller, Service, Repository로 나눠보자.

전체 흐름은 다음과 같다.

```
HTTP Request
    ↓
UserController
    ↓
UserService
    ↓
UserRepository
    ↓
Database
    ↓
UserRepository
    ↓
UserService
    ↓
UserController
    ↓
HTTP Response
```

---

## 1. Controller

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

Controller는 HTTP 요청을 받는다.

```
GET /users/1
```

요청 URL에서 `id`를 꺼내 Service에 전달한다.

```
userService.getUser(id);
```

회원이 존재하는지 판단하거나 DB를 직접 조회하지 않는다.

Controller는 웹 요청을 해석하고 응답을 반환하는 역할에 집중한다.

---

## 2. Service

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

Service는 회원 조회 작업의 흐름을 담당한다.

현재 코드에서는 다음 작업을 처리한다.

1. Repository를 통해 회원을 조회한다.
2. 회원이 없으면 예외를 발생시킨다.
3. 조회된 회원을 응답 DTO로 변환한다.

단순한 조회에서는 로직이 많지 않아 보일 수 있다.

하지만 실제 서비스에서는 다음과 같은 비즈니스 규칙이 추가될 수 있다.

- 탈퇴한 회원은 조회할 수 없다.
- 차단된 회원은 일부 정보만 보여준다.
- 관리자만 민감한 정보를 조회할 수 있다.
- 조회 기록을 남긴다.
- 여러 Repository의 데이터를 조합한다.

이러한 판단과 흐름은 Service에서 처리하는 것이 자연스럽다.

---

## 3. Repository

```
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
}
```

Repository는 데이터베이스 접근을 담당한다.

`JpaRepository`를 상속하면 기본적인 저장과 조회 기능을 사용할 수 있다.

```
findById(id);
save(user);
delete(user);
findAll();
```

필요하다면 Query Method, JPQL, QueryDSL 등을 사용해 조회 조건을 추가할 수도 있다.

```
Optional<User> findByEmail(String email);
```

Repository는 데이터를 어떻게 가져오고 저장할지를 담당하지만, 그 데이터로 어떤 비즈니스 판단을 할지는 결정하지 않는다.

---

## 4. Entity와 DTO

데이터베이스의 회원 정보를 나타내는 Entity를 간단히 만들어보자.

```
@Entity
public class User {

    @Id
    private Long id;

    private String name;

    protected User() {
    }

    public Long getId() {
        return id;
    }

    public String getName() {
        return name;
    }
}
```

API 응답에는 Entity를 그대로 반환하지 않고 별도의 DTO를 사용할 수 있다.

```
public record UserResponse(
        Long id,
        String name
) {
}
```

Entity는 데이터베이스와 가까운 객체이고, DTO는 요청이나 응답에 필요한 데이터를 전달하는 객체다.

둘을 분리하면 데이터베이스 구조가 바뀌거나 API 응답 형식이 바뀌었을 때 서로에게 미치는 영향을 줄일 수 있다.

---

# 각 계층은 어떤 일을 해야 할까?

## Controller의 역할

Controller는 웹 요청과 응답에 가까운 일을 담당한다.

- URL 매핑
- HTTP Method 처리
- `PathVariable` 받기
- `RequestParam` 받기
- `RequestBody` 받기
- 요청 값 검증
- Service 호출
- HTTP 상태 코드와 응답 반환

예를 들면 다음과 같은 코드가 Controller에 위치한다.

```
@PostMapping
public ResponseEntity<UserResponse> createUser(
        @Valid @RequestBody SignupRequest request
) {
    UserResponse response = userService.signup(request);

    return ResponseEntity
            .status(HttpStatus.CREATED)
            .body(response);
}
```

Controller는 요청을 Service가 사용할 수 있는 형태로 전달하고, 결과를 HTTP 응답으로 바꾸는 역할에 집중하는 것이 좋다.

---

## Service의 역할

Service는 비즈니스 로직과 작업 흐름을 담당한다.

- 회원이 존재하는지 확인
- 이미 가입된 이메일인지 검사
- 주문 가능한 상태인지 판단
- 사용자 권한 확인
- 가격과 할인 금액 계산
- 여러 Repository 호출 조합
- 트랜잭션 범위 설정
- 예외 상황 판단
- 외부 시스템 호출 순서 결정

예를 들어 주문을 생성할 때는 다음과 같은 작업이 필요할 수 있다.

```
회원 조회
→ 상품 조회
→ 재고 확인
→ 가격 계산
→ 주문 저장
→ 재고 차감
```

이처럼 여러 작업을 하나의 비즈니스 흐름으로 묶는 것이 Service의 역할이다.

---

## Repository의 역할

Repository는 데이터 저장소와 대화한다.

- 데이터 조회
- 데이터 저장
- 데이터 수정
- 데이터 삭제
- Query Method 작성
- JPQL 작성
- QueryDSL 조회
- Entity와 테이블 매핑을 통한 데이터 접근

Repository는 다음 질문에 답하는 계층이다.

> “필요한 데이터를 어디에서 어떻게 가져올 것인가?”

반면 다음과 같은 판단은 Repository보다 Service에 두는 편이 좋다.

> “이 회원이 현재 주문할 수 있는 상태인가?”

Repository는 데이터를 가져오고, Service가 그 데이터를 바탕으로 정책을 판단한다.

---

# Controller가 너무 많은 일을 하면 생기는 문제

다음 코드를 보자.

```
@GetMapping("/users/{id}")
public UserResponse getUser(@PathVariable Long id) {
    User user = userRepository.findById(id)
            .orElseThrow(() -> new UserNotFoundException(id));

    if (!user.isActive()) {
        throw new InactiveUserException(id);
    }

    return new UserResponse(
            user.getId(),
            user.getName()
    );
}
```

코드는 동작할 수 있다.

하지만 Controller 안에 서로 다른 책임이 섞여 있다.

- HTTP 요청 처리
- DB 조회
- 회원 존재 여부 검사
- 활성 회원인지 판단
- 응답 DTO 생성

특히 다음 코드는 단순 조회가 아니라 비즈니스 규칙에 해당한다.

```
if (!user.isActive()) {
    throw new InactiveUserException(id);
}
```

이 로직이 Controller에 있으면 다른 API나 배치 작업에서 그대로 재사용하기 어렵다.

Service로 옮기면 다음과 같이 작성할 수 있다.

```
@Service
public class UserService {

    private final UserRepository userRepository;

    public UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    public UserResponse getUser(Long id) {
        User user = findActiveUser(id);

        return new UserResponse(
                user.getId(),
                user.getName()
        );
    }

    private User findActiveUser(Long id) {
        User user = userRepository.findById(id)
                .orElseThrow(() -> new UserNotFoundException(id));

        if (!user.isActive()) {
            throw new InactiveUserException(id);
        }

        return user;
    }
}
```

Controller는 다시 단순해진다.

```
@GetMapping("/{id}")
public UserResponse getUser(@PathVariable Long id) {
    return userService.getUser(id);
}
```

이제 활성 회원을 찾는 규칙은 Service 안에 있으므로 HTTP API가 아닌 다른 작업에서도 활용할 수 있다.

---

# Service에 모든 로직을 넣으면 될까?

Controller를 얇게 만들겠다고 모든 코드를 Service 하나에 몰아넣는 것도 좋은 구조는 아니다.

예를 들어 `UserService` 하나가 다음 기능을 모두 담당한다고 생각해보자.

- 회원가입
- 로그인
- 회원 정보 수정
- 비밀번호 변경
- 포인트 적립
- 쿠폰 발급
- 환영 메일 발송
- 알림 전송
- 프로필 이미지 업로드

기능이 많아지면 `UserService` 역시 지나치게 커질 수 있다.

이럴 때는 기능이나 책임에 따라 클래스를 더 나눌 수 있다.

예를 들어 메일 발송은 별도의 컴포넌트로 분리할 수 있다.

```
@Component
public class MailSender {

    public void sendWelcomeMail(String email) {
        // 메일 발송
    }
}
```

Service는 회원가입의 전체 흐름을 조합한다.

```
@Service
public class SignupService {

    private final UserRepository userRepository;
    private final MailSender mailSender;

    public SignupService(
            UserRepository userRepository,
            MailSender mailSender
    ) {
        this.userRepository = userRepository;
        this.mailSender = mailSender;
    }

    public void signup(SignupRequest request) {
        User user = new User(request.email());

        User savedUser = userRepository.save(user);

        mailSender.sendWelcomeMail(savedUser.getEmail());
    }
}
```

여기서 Service는 직접 메일을 보내는 방법을 구현하지 않는다.

대신 회원을 저장한 뒤 환영 메일을 발송한다는 비즈니스 흐름을 관리한다.

즉, Service 계층이 있다고 해서 하나의 Service에 모든 코드를 넣어도 된다는 뜻은 아니다.

---

# 실무에서 자주 조심해야 할 점

## 1. Controller는 가능한 한 얇게 유지한다

Controller에는 이미 HTTP 관련 코드가 많이 들어간다.

```
@RequestBody
@PathVariable
@RequestParam
@Valid
ResponseEntity
HttpStatus
```

여기에 DB 조회와 비즈니스 판단까지 들어가면 한 메서드가 너무 많은 일을 하게 된다.

Controller는 다음 흐름 정도로 읽히는 것이 좋다.

```
요청을 받는다
→ 필요한 값을 꺼낸다
→ Service를 호출한다
→ 응답을 반환한다
```

---

## 2. Service 메서드는 비즈니스 의미가 드러나게 작성한다

다음 메서드 이름은 데이터베이스 작업처럼 느껴진다.

```
selectUser(id);
insertOrder(order);
updatePoint(point);
```

Service에서는 기술적인 표현보다 어떤 업무를 수행하는지가 드러나는 이름이 좋다.

```
getUserProfile(userId);
placeOrder(command);
earnPoints(userId, amount);
cancelOrder(orderId);
```

Service 코드를 읽었을 때 데이터베이스 조작보다 비즈니스 행동이 보이는 것이 좋다.

---

## 3. Repository에는 비즈니스 정책을 넣지 않는다

Repository는 데이터를 조회하고 저장하는 역할이다.

다음과 같은 판단은 Service에서 처리하는 편이 자연스럽다.

```
탈퇴한 회원은 로그인할 수 없다.
재고가 부족하면 주문할 수 없다.
주문 완료 후에는 주소를 변경할 수 없다.
관리자만 회원의 상세 정보를 볼 수 있다.
```

Repository는 판단에 필요한 데이터를 제공한다.

```
Optional<User> findById(Long id);
```

Service는 조회한 데이터를 바탕으로 정책을 판단한다.

```
if (user.isWithdrawn()) {
    throw new WithdrawnUserException(userId);
}
```

---

## 4. 모든 기능에 계층을 기계적으로 추가할 필요는 없다

계층 분리는 목적이 아니라 수단이다.

아주 작은 기능에서 Service가 아무런 판단도 하지 않고 Repository 메서드 하나만 그대로 호출한다면, Service가 불필요하게 느껴질 수도 있다.

```
public List<CategoryResponse> getCategories() {
    return categoryRepository.findAll()
            .stream()
            .map(CategoryResponse::from)
            .toList();
}
```

이런 구조가 항상 잘못된 것은 아니다.

프로젝트의 규모, 변경 가능성, 팀의 규칙에 따라 선택할 수 있다.

다만 다음과 같은 가능성이 있다면 Service 계층을 두는 편이 유지보수에 도움이 된다.

- 비즈니스 규칙이 추가될 가능성이 있다.
- 여러 Repository를 조합할 수 있다.
- 트랜잭션이 필요할 수 있다.
- API 외의 다른 곳에서도 로직을 재사용할 수 있다.
- 테스트할 비즈니스 조건이 존재한다.

중요한 것은 계층의 개수가 아니라 각 클래스가 맡은 책임이 명확한지다.

---

# 역할을 나누면 테스트도 쉬워진다

Controller, Service, Repository를 분리하면 계층별로 무엇을 테스트해야 하는지도 명확해진다.

## Controller 테스트

Controller 테스트에서는 HTTP 요청과 응답을 확인한다.

- URL 매핑이 올바른가?
- 요청 값이 정상적으로 변환되는가?
- Validation이 동작하는가?
- 응답 상태 코드가 올바른가?
- 응답 JSON 형식이 올바른가?

예를 들면 다음과 같다.

```
GET /users/1 요청
→ HTTP 200 응답

잘못된 요청 값 전달
→ HTTP 400 응답
```

Controller 테스트에서는 Service를 Mock으로 대체하고 웹 계층만 확인할 수 있다.

---

## Service 테스트

Service 테스트에서는 비즈니스 규칙을 확인한다.

- 회원이 없으면 예외가 발생하는가?
- 탈퇴한 회원은 조회할 수 없는가?
- 이미 가입된 이메일이면 회원가입이 실패하는가?
- 쿠폰 할인 금액이 올바르게 계산되는가?
- 재고가 부족하면 주문이 실패하는가?
- 주문과 재고 변경이 하나의 트랜잭션으로 처리되는가?

Service 테스트에서는 Repository를 Mock으로 대체하고 비즈니스 판단에 집중할 수 있다.

---

## Repository 테스트

Repository 테스트에서는 실제 데이터 접근이 올바른지 확인한다.

- Query Method가 원하는 데이터를 조회하는가?
- JPQL 조건이 올바른가?
- Entity 매핑이 정상인가?
- 저장과 수정이 정상적으로 반영되는가?
- 데이터베이스 제약조건이 동작하는가?

계층을 분리하면 테스트 실패가 발생했을 때 어느 부분에 문제가 있는지도 찾기 쉬워진다.

---

# 계층 분리의 핵심은 책임 분리다

Controller, Service, Repository를 나누는 이유는 단순히 프로젝트 폴더를 보기 좋게 만들기 위해서가 아니다.

각 계층이 서로 다른 책임을 담당하도록 하기 위해서다.

```
Controller
HTTP 요청을 받고 응답을 반환한다.

Service
비즈니스 규칙을 판단하고 작업 흐름을 관리한다.

Repository
데이터 저장소를 조회하고 변경한다.
```

역할을 나누면 다음과 같은 장점이 생긴다.

- 코드를 읽기 쉬워진다.
- 변경해야 할 위치를 찾기 쉬워진다.
- 비즈니스 로직을 재사용하기 쉬워진다.
- 계층별 테스트를 작성하기 쉬워진다.
- HTTP와 DB 같은 기술 코드가 비즈니스 로직과 덜 섞인다.
- 한 클래스가 너무 많은 책임을 가지는 것을 막을 수 있다.

다만 Controller, Service, Repository를 만들었다는 사실만으로 좋은 구조가 되는 것은 아니다.

Controller에 비즈니스 로직이 들어갈 수도 있고, Service가 지나치게 커질 수도 있으며, Repository에 정책 판단이 들어갈 수도 있다.

중요한 것은 애너테이션의 이름이 아니라 코드가 어떤 책임을 가지고 있는지다.

코드를 작성할 때 다음 세 가지 질문을 해보면 도움이 된다.

```
이 코드는 HTTP 요청이나 응답을 다루는가?
→ Controller

이 코드는 업무 규칙이나 작업 순서를 판단하는가?
→ Service

이 코드는 데이터베이스를 조회하거나 변경하는가?
→ Repository
```

처음부터 모든 코드를 완벽하게 나누기는 어렵다.

하지만 기능이 추가될 때마다 각 코드가 어떤 책임을 가지고 있는지 생각하다 보면, Controller, Service, Repository를 어디까지 나눠야 하는지 조금씩 감을 잡을 수 있다.

마지막으로 다시 한 문장으로 정리하면 다음과 같다.

> Controller는 요청의 입구이고, Service는 판단의 중심이며, Repository는 데이터의 통로다.
