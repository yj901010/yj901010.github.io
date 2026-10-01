---
title: "DTO를 쓰는 이유"
date: 2026-08-04 13:21:23 +0900
last_modified_at: 2026-10-01
categories: [Backend, Spring]
tags: [dto, spring, api]
source_url: "https://velog.io/@joker901010/DTO%EB%A5%BC-%EC%93%B0%EB%8A%94-%EC%9D%B4%EC%9C%A0"
---
DTO는 요청과 응답에 필요한 데이터만 담아 전달하는 객체다.

백엔드 API에서는 DTO를 사용해 외부에 공개되는 요청·응답 형식을 Entity와 내부 구현으로부터 분리할 수 있다.

---

## 처음에는 DTO가 이해되지 않았다

Spring으로 처음 API를 만들 때는 이런 생각이 들었다.

> “Entity가 이미 있는데 DTO를 왜 또 만들어야 하지?”

Entity에도 `email`, `name`, `password` 같은 필드가 있다.

그렇다면 요청을 받을 때도 Entity를 사용하고, 응답을 보낼 때도 Entity를 그대로 반환하면 되지 않을까?

실제로 간단한 프로젝트에서는 다음과 같이 작성할 수도 있다.

```
@Entity
public class User {

    @Id
    private Long id;

    private String email;

    private String password;

    private String name;
}
```

```
@GetMapping("/users/{id}")
public User getUser(@PathVariable Long id) {
    return userRepository.findById(id)
            .orElseThrow();
}
```

코드만 보면 간단하다.

조회한 `User` Entity를 Controller에서 그대로 반환하면 Spring이 JSON으로 변환해준다.

하지만 프로젝트가 조금만 커지면 이 방식에는 여러 문제가 생긴다.

---

## Spring은 Java 객체를 어떻게 JSON으로 바꿀까?

Spring MVC는 HTTP 요청과 응답의 본문을 읽고 쓰기 위해 `HttpMessageConverter`를 사용한다.

클라이언트가 다음과 같은 JSON을 보냈다고 해보자.

```
{
  "email": "junior@example.com",
  "password": "12345678",
  "name": "junior"
}
```

Controller에서 `@RequestBody`를 사용하면 Spring은 JSON 요청 본문을 Java 객체로 변환한다.

```
@PostMapping
public void createUser(@RequestBody UserCreateRequest request) {
}
```

반대로 `@RestController`의 메서드가 Java 객체를 반환하면 Spring은 해당 객체를 JSON 응답 본문으로 변환한다.

요청 JSON은 요청 DTO로 변환되고, 처리 결과인 응답 DTO는 다시 JSON으로 변환되어 클라이언트에 전달된다.

```
@GetMapping("/users/{id}")
public UserResponse getUser(@PathVariable Long id) {
    return new UserResponse(
            id,
            "junior@example.com",
            "junior"
    );
}
```

응답은 다음과 같은 JSON이 된다.

```
{
  "id": 1,
  "email": "junior@example.com",
  "name": "junior"
}
```

즉, DTO가 JSON으로 바뀌는 특별한 객체라서 사용하는 것은 아니다.

Entity든 DTO든 일반 Java 객체라면 조건에 따라 JSON으로 변환될 수 있다.

중요한 것은 **어떤 객체를 외부 요청과 응답의 형식으로 사용할 것인가**다.

DTO는 이 외부 형식을 명확하게 정의하기 위해 사용한다.

---

## Entity를 그대로 반환하면 생기는 문제

### 1. 민감한 정보가 노출될 수 있다

회원 Entity에는 API 응답으로 보내면 안 되는 값이 들어 있을 수 있다.

```
@Entity
public class User {

    @Id
    private Long id;

    private String email;

    private String password;

    private String refreshToken;

    private boolean deleted;

    private String internalMemo;
}
```

이 Entity를 그대로 반환하면 다음과 같은 내부 정보까지 JSON에 포함될 수 있다.

- 비밀번호
- 리프레시 토큰
- 삭제 여부
- 내부 관리 메모
- 생성자와 수정자 정보

물론 `@JsonIgnore`를 사용해 특정 필드를 제외할 수도 있다.

하지만 Entity에 API 응답을 위한 설정이 계속 추가되면 Entity가 영속성, 도메인 로직, API 표현까지 한꺼번에 책임지게 된다.

차라리 응답에 필요한 값만 담은 DTO를 만드는 편이 더 명확하다.

```
public record UserResponse(
        Long id,
        String email,
        String name
) {
}
```

이 DTO에는 처음부터 `password`와 `refreshToken`이 없다.

실수로 응답에 포함될 가능성 자체를 줄일 수 있다.

---

### 2. 데이터베이스 구조가 API 응답에 드러난다

Entity는 일반적으로 데이터베이스 저장 구조와 밀접한 관계를 가진다.

반면 API 응답은 클라이언트와 약속한 외부 형식이다.

두 구조는 변경되는 이유가 다르다.

예를 들어 내부 사정으로 Entity의 필드명을 변경했다고 해보자.

```
private String name;
```

이 필드를 다음과 같이 변경할 수 있다.

```
private String displayName;
```

Entity를 그대로 반환하고 있었다면 응답 JSON의 필드명도 함께 바뀔 수 있다.

```
{
  "displayName": "junior"
}
```

하지만 프론트엔드는 기존 응답을 기대하고 있을 수 있다.

```
{
  "name": "junior"
}
```

데이터베이스나 내부 모델을 변경했을 뿐인데 외부 API까지 영향을 받게 되는 것이다.

응답 DTO를 따로 두면 내부 구조가 바뀌더라도 외부 응답 형식을 유지하기 쉬워진다.

```
public record UserResponse(
        Long id,
        String email,
        String name
) {
}
```

```
return new UserResponse(
        user.getId(),
        user.getEmail(),
        user.getDisplayName()
);
```

내부에서는 `displayName`을 사용하더라도 API에서는 기존의 `name`을 유지할 수 있다.

---

### 3. 연관관계 때문에 응답이 지나치게 커질 수 있다

JPA Entity는 다른 Entity와 연관관계를 가질 수 있다.

```
@Entity
public class User {

    @OneToMany(mappedBy = "user")
    private List<Order> orders;
}
```

```
@Entity
public class Order {

    @ManyToOne
    private User user;
}
```

이 구조에서 Entity를 그대로 JSON으로 변환하면 다음과 같이 서로를 계속 참조할 수 있다.

```
User
 └─ Orders
     └─ User
         └─ Orders
             └─ User
```

이 때문에 순환 참조 오류가 발생하거나, 필요 이상으로 큰 응답이 만들어질 수 있다.

또한 지연 로딩된 연관관계를 직렬화하는 과정에서 예상하지 못한 추가 쿼리가 발생할 수도 있다.

응답 DTO를 사용하면 API에 필요한 데이터만 선택할 수 있다.

```
public record UserDetailResponse(
        Long id,
        String name,
        int orderCount
) {
}
```

```
{
  "id": 1,
  "name": "junior",
  "orderCount": 3
}
```

클라이언트가 주문 전체 목록을 필요로 하지 않는다면 주문 개수만 내려주면 된다.

Entity 구조를 그대로 노출할 필요가 없다.

---

### 4. 화면에 필요한 응답 모양을 만들기 어렵다

데이터베이스의 저장 구조와 화면에서 필요한 데이터의 모양은 항상 같지 않다.

예를 들어 회원 목록 화면에서는 다음 정보만 필요할 수 있다.

```
{
  "id": 1,
  "name": "junior",
  "orderCount": 3
}
```

반면 회원 상세 화면에서는 더 많은 정보가 필요할 수 있다.

```
{
  "id": 1,
  "email": "junior@example.com",
  "name": "junior",
  "status": "ACTIVE",
  "createdAt": "2026-08-04T10:00:00"
}
```

하나의 `User` Entity를 모든 화면의 응답으로 사용하려고 하면 필요하지 않은 필드까지 노출되거나, 화면마다 여러 직렬화 설정을 추가해야 한다.

DTO를 사용하면 각 API의 목적에 맞는 응답을 만들 수 있다.

```
public record UserSummaryResponse(
        Long id,
        String name,
        int orderCount
) {
}
```

```
public record UserDetailResponse(
        Long id,
        String email,
        String name,
        String status,
        LocalDateTime createdAt
) {
}
```

이름만 보더라도 각 DTO가 어떤 응답에 사용되는지 알 수 있다.

---

## Entity와 DTO는 무엇이 다를까?

Entity와 DTO의 차이는 창고 재고표와 고객 영수증의 차이와 비슷하다.

창고 재고표에는 내부 관리에 필요한 정보가 들어 있다.

- 상품 ID
- 입고 날짜
- 창고 위치
- 매입 원가
- 재고 수량
- 내부 관리 메모

하지만 고객에게 주는 영수증에는 고객에게 필요한 정보만 들어간다.

- 상품명
- 구매 수량
- 결제 금액
- 구매 날짜

창고 재고표를 고객에게 그대로 주면 안 된다.

고객에게 필요하지 않은 내부 정보가 포함되어 있기 때문이다.

Entity가 내부 관리용 재고표라면, DTO는 외부 사용자에게 필요한 정보만 정리한 영수증과 비슷하다.

백엔드에서도 비슷하다.

**Entity**

- 데이터베이스 저장과 조회에 사용된다.
- 영속성 상태와 연관관계를 가질 수 있다.
- 내부 도메인 모델의 일부로 사용될 수 있다.

**DTO**

- 요청과 응답 등 데이터를 전달하기 위해 사용된다.
- 특정 API에 필요한 필드만 담을 수 있다.
- 외부에 공개할 데이터의 형태를 명확하게 정의한다.

API 사용자는 서버의 테이블 구조를 알고 싶은 것이 아니다.

그 API가 약속한 요청과 응답의 모양을 알고 싶은 것이다.

---

## 회원 생성 API로 이해해보기

회원 생성 API를 만들어보자.

클라이언트는 다음과 같은 요청을 보낸다.

```
POST /users
Content-Type: application/json
```

```
{
  "email": "junior@example.com",
  "password": "12345678",
  "name": "junior"
}
```

이 요청을 받을 DTO를 만든다.

```
public record UserCreateRequest(
        String email,
        String password,
        String name
) {
}
```

Controller에서는 `@RequestBody`를 사용해 요청 JSON을 DTO로 받는다.

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
            @RequestBody UserCreateRequest request
    ) {
        return userService.createUser(request);
    }
}
```

Service에서는 요청 DTO의 값을 사용해 회원을 생성한다.

```
@Service
public class UserService {

    private final UserRepository userRepository;
    private final PasswordEncoder passwordEncoder;

    public UserService(
            UserRepository userRepository,
            PasswordEncoder passwordEncoder
    ) {
        this.userRepository = userRepository;
        this.passwordEncoder = passwordEncoder;
    }

    public UserResponse createUser(UserCreateRequest request) {
        User user = new User(
                request.email(),
                passwordEncoder.encode(request.password()),
                request.name()
        );

        User savedUser = userRepository.save(user);

        return new UserResponse(
                savedUser.getId(),
                savedUser.getEmail(),
                savedUser.getName()
        );
    }
}
```

응답 DTO는 따로 만든다.

```
public record UserResponse(
        Long id,
        String email,
        String name
) {
}
```

응답은 다음과 같다.

```
{
  "id": 1,
  "email": "junior@example.com",
  "name": "junior"
}
```

요청에는 회원을 생성하기 위해 `password`가 필요하다.

하지만 응답에는 `password`가 절대 포함되면 안 된다.

이 차이만 봐도 요청 DTO와 응답 DTO를 분리하는 이유를 이해할 수 있다.

---

## 요청 DTO와 응답 DTO는 목적이 다르다

요청과 응답은 데이터가 이동하는 방향부터 다르다.

요청 DTO는 클라이언트가 서버에 보내는 값을 표현한다.

```
public record UserCreateRequest(
        String email,
        String password,
        String name
) {
}
```

응답 DTO는 서버가 처리한 결과를 클라이언트에 전달한다.

```
public record UserResponse(
        Long id,
        String email,
        String name
) {
}
```

요청에는 없지만 응답에는 존재하는 값이 있다.

- `id`
- `createdAt`
- `status`

반대로 요청에는 필요하지만 응답에는 없어야 하는 값도 있다.

- `password`
- `passwordConfirm`
- `verificationCode`

요청과 응답을 하나의 DTO로 합치면 다음과 같은 형태가 될 수 있다.

```
public record UserDto(
        Long id,
        String email,
        String password,
        String passwordConfirm,
        String name,
        String status
) {
}
```

이 DTO를 보면 각 필드가 언제 사용되는지 바로 알기 어렵다.

- 회원가입 요청에서 `id`는 필요한가?
- 회원 조회 응답에 `passwordConfirm`이 필요한가?
- `status`는 클라이언트가 입력하는 값인가?
- 사용하지 않는 필드는 항상 `null`로 두는가?

처음에는 클래스 수를 줄인 것처럼 보인다.

하지만 시간이 지나면 DTO의 의미가 모호해지고 잘못된 필드를 사용하는 실수가 생길 수 있다.

그래서 보통 목적에 따라 DTO를 나눈다.

```
UserCreateRequest
UserUpdateRequest
UserResponse
UserSummaryResponse
UserDetailResponse
```

이름만 보더라도 어떤 API에서 사용하는 객체인지 짐작할 수 있다.

---

## DTO에 요청 검증 규칙을 표현할 수 있다

요청 DTO의 장점 중 하나는 입력값 검증 조건을 명확하게 표현할 수 있다는 것이다.

회원가입 요청 DTO에 Bean Validation 어노테이션을 추가해보자.

```
public record UserCreateRequest(

        @NotBlank
        @Email
        String email,

        @NotBlank
        @Size(min = 8)
        String password,

        @NotBlank
        String name
) {
}
```

Controller에서는 `@Valid`를 붙인다.

```
@PostMapping
public UserResponse createUser(
        @Valid @RequestBody UserCreateRequest request
) {
    return userService.createUser(request);
}
```

다음과 같은 요청이 들어왔다고 해보자.

```
{
  "email": "not-email",
  "password": "123",
  "name": ""
}
```

이 요청은 여러 검증 조건을 만족하지 못한다.

- `email`이 이메일 형식이 아니다.
- `password`가 8자보다 짧다.
- `name`이 비어 있다.

Spring은 요청 본문을 DTO로 변환한 뒤 검증을 수행할 수 있다.

검증에 실패하면 Controller의 비즈니스 로직이 실행되기 전에 잘못된 요청을 처리할 수 있다.

---

## 검증을 Entity가 아니라 요청 DTO에 두는 이유

Entity에 검증 어노테이션을 붙이는 것도 가능하다.

하지만 같은 회원 데이터라도 API마다 요구하는 입력값이 다를 수 있다.

회원가입 요청에서는 다음 값이 필요하다.

```
email
password
name
```

반면 회원 이름 수정 요청에서는 `name`만 필요할 수 있다.

```
public record UserNameUpdateRequest(

        @NotBlank
        String name
) {
}
```

비밀번호 변경 API라면 기존 비밀번호와 새로운 비밀번호가 필요할 수 있다.

```
public record UserPasswordUpdateRequest(

        @NotBlank
        String currentPassword,

        @NotBlank
        @Size(min = 8)
        String newPassword
) {
}
```

하나의 Entity에 모든 요청의 검증 규칙을 넣으려고 하면 조건이 복잡해진다.

반면 요청 DTO별로 검증 규칙을 작성하면 각 API가 어떤 입력을 요구하는지 명확하게 드러난다.

DTO는 단순히 값을 옮기는 객체이면서, 동시에 API의 입력 명세를 코드로 표현하는 역할도 한다.

---

## Java record를 DTO로 사용하는 이유

최근 Java 프로젝트에서는 DTO를 `record`로 작성하는 경우가 많다.

일반적인 DTO는 값을 저장하고 전달하는 목적이 크다.

기존 클래스 방식으로 DTO를 만들면 필드, 생성자, getter 등을 작성해야 한다.

```
public class UserResponse {

    private final Long id;
    private final String email;
    private final String name;

    public UserResponse(Long id, String email, String name) {
        this.id = id;
        this.email = email;
        this.name = name;
    }

    public Long getId() {
        return id;
    }

    public String getEmail() {
        return email;
    }

    public String getName() {
        return name;
    }
}
```

record를 사용하면 같은 목적의 객체를 더 간결하게 표현할 수 있다.

```
public record UserResponse(
        Long id,
        String email,
        String name
) {
}
```

record는 생성 이후 값을 변경할 수 없는 데이터 묶음을 표현하기에 적합하다.

다만 record만 DTO로 사용할 수 있는 것은 아니다.

일반 클래스도 DTO로 사용할 수 있으며, 프로젝트의 Java 버전과 객체 생성 방식에 따라 적절한 형태를 선택하면 된다.

---

## DTO 변환은 어디에서 해야 할까?

DTO를 사용하면 Entity와 DTO 사이를 변환하는 코드가 필요하다.

```
UserResponse response = new UserResponse(
        user.getId(),
        user.getEmail(),
        user.getName()
);
```

이 변환을 어디에서 수행할지는 프로젝트마다 다르다.

### Service에서 변환하기

```
public UserResponse getUser(Long id) {
    User user = userRepository.findById(id)
            .orElseThrow();

    return new UserResponse(
            user.getId(),
            user.getEmail(),
            user.getName()
    );
}
```

작은 프로젝트에서는 이해하기 쉽고 간단하다.

하지만 Service 메서드가 많아지면 변환 코드가 반복될 수 있다.

### DTO에 정적 팩토리 메서드 두기

```
public record UserResponse(
        Long id,
        String email,
        String name
) {

    public static UserResponse from(User user) {
        return new UserResponse(
                user.getId(),
                user.getEmail(),
                user.getName()
        );
    }
}
```

Service에서는 다음과 같이 사용할 수 있다.

```
return UserResponse.from(user);
```

변환 위치를 쉽게 찾을 수 있다는 장점이 있다.

### Mapper 클래스 사용하기

```
@Component
public class UserMapper {

    public UserResponse toResponse(User user) {
        return new UserResponse(
                user.getId(),
                user.getEmail(),
                user.getName()
        );
    }
}
```

변환 로직이 많거나 여러 곳에서 재사용된다면 Mapper를 분리할 수 있다.

### MapStruct 사용하기

필드가 많은 DTO 변환이 반복된다면 MapStruct 같은 매핑 도구를 사용할 수도 있다.

어떤 방식이 항상 정답인 것은 아니다.

중요한 것은 프로젝트 안에서 변환 규칙과 위치를 일관되게 유지하는 것이다.

---

## DTO를 사용한다고 모든 문제가 해결되지는 않는다

DTO는 유용하지만 무조건 많이 만든다고 좋은 것은 아니다.

### DTO에 비즈니스 로직을 너무 많이 넣지 않기

DTO의 기본 목적은 데이터를 전달하는 것이다.

다음과 같은 복잡한 판단까지 DTO가 담당하기 시작하면 책임이 불분명해질 수 있다.

- 회원 등급 결정
- 할인 금액 계산
- 주문 가능 여부 판단
- 결제 상태 변경
- 재고 차감

이런 핵심 비즈니스 규칙은 Service나 도메인 객체 등 적절한 위치에 두는 편이 좋다.

다만 값의 형식을 단순히 변환하거나 여러 필드로부터 표시용 문자열을 만드는 정도는 프로젝트의 기준에 따라 허용할 수 있다.

중요한 것은 DTO가 핵심 비즈니스 로직의 중심이 되지 않도록 하는 것이다.

---

### DTO를 지나치게 잘게 나누지 않기

API마다 무조건 새로운 DTO를 만들면 클래스 수가 지나치게 많아질 수 있다.

두 API의 목적과 필드 구성이 사실상 같다면 같은 DTO를 재사용할 수도 있다.

반대로 다음과 같은 차이가 생기면 분리를 고려해야 한다.

- 요청과 응답의 방향이 다르다.
- 민감 정보 포함 여부가 다르다.
- 검증 규칙이 다르다.
- 화면별로 필요한 필드가 다르다.
- 향후 서로 다르게 변경될 가능성이 높다.

DTO를 분리하는 기준은 단순히 필드가 한 개 다르냐가 아니다.

**서로 다른 이유로 변경되는 객체인지**를 생각해보는 것이 좋다.

---

## DTO 이름은 목적이 드러나게 짓는 것이 좋다

다음과 같은 이름은 범위가 너무 넓다.

```
UserDto
UserData
UserInfo
```

프로젝트가 작을 때는 문제가 없어 보일 수 있다.

하지만 API가 많아지면 `UserDto`가 요청용인지, 응답용인지, 목록용인지 알기 어려워진다.

조금 더 구체적인 이름을 사용하면 역할이 명확해진다.

```
UserCreateRequest
UserUpdateRequest
UserResponse
UserSummaryResponse
UserDetailResponse
```

`Request`와 `Response`를 붙이는 것만으로도 데이터가 이동하는 방향을 알 수 있다.

`Create`, `Update`, `Summary`, `Detail` 같은 단어를 사용하면 DTO의 목적도 함께 드러난다.

좋은 이름은 별도의 설명 없이도 객체가 사용되는 상황을 알려준다.

---

## DTO는 계층마다 반드시 만들어야 할까?

DTO는 흔히 계층 사이에서 데이터를 전달하는 객체라고 설명된다.

하지만 그렇다고 해서 Controller, Service, Repository 사이마다 무조건 별도의 DTO를 만들어야 하는 것은 아니다.

예를 들어 다음과 같이 모든 계층마다 객체를 하나씩 만들 수 있다.

```
UserCreateHttpRequest
UserCreateServiceCommand
UserCreateDomainData
UserCreatePersistenceData
UserCreateResponse
```

복잡한 시스템에서는 이런 분리가 필요할 수 있다.

하지만 작은 CRUD 프로젝트에서 무조건 이렇게 나누면 변환 코드와 클래스만 늘어날 수 있다.

처음에는 다음 두 가지를 분리하는 것만으로도 충분한 경우가 많다.

```
외부 요청 DTO
외부 응답 DTO
```

그리고 서비스가 복잡해지면서 HTTP 요청 형식과 비즈니스 로직의 입력을 별도로 관리할 필요가 생길 때 Command나 Query 객체를 추가할 수 있다.

중요한 것은 DTO를 많이 만드는 것이 아니다.

외부 계약과 내부 모델을 어디까지 분리해야 하는지 판단하는 것이다.

---

## 공식 문서와 연결해서 이해하기

Spring MVC는 `HttpMessageConverter`를 사용해 HTTP 요청과 응답의 본문을 Java 객체와 변환한다.

- `@RequestBody`를 사용하면 요청 본문을 Java 객체로 변환할 수 있다.
- `@ResponseBody`가 적용된 메서드의 반환값은 응답 본문으로 직렬화될 수 있다.
- `@RestController`는 `@Controller`와 `@ResponseBody`의 역할을 함께 제공한다.
- `@RequestBody`에 `@Valid` 또는 `@Validated`를 함께 사용하면 변환된 요청 객체를 검증할 수 있다.

Java의 record는 고정된 데이터 묶음을 간결하게 표현하기 위한 클래스 형태다.

그래서 요청과 응답처럼 값을 전달하는 것이 주목적인 DTO에 자주 사용된다.

다만 공식 문서가 “API에서는 반드시 DTO를 사용해야 한다”고 규정하는 것은 아니다.

DTO는 Spring MVC의 필수 기능이라기보다, 애플리케이션의 외부 계약과 내부 모델을 분리하기 위해 사용하는 설계 방식에 가깝다.

---

## 정리

DTO는 데이터를 전달하기 위한 객체다.

백엔드 API에서 DTO를 사용하는 대표적인 이유는 다음과 같다.

- Entity를 외부에 직접 노출하지 않기 위해
- 민감한 정보가 응답으로 나가는 것을 막기 위해
- 요청과 응답의 형식을 분리하기 위해
- API에 필요한 필드만 주고받기 위해
- API별 입력값 검증 규칙을 표현하기 위해
- 데이터베이스 구조의 변화가 API에 직접 전파되는 것을 줄이기 위해
- 화면과 기능에 맞는 응답 형식을 만들기 위해
- 연관관계로 인한 순환 참조와 과도한 응답을 피하기 위해

나는 DTO를 다음과 같이 이해하게 되었다.

> DTO는 서버의 내부 모델과 외부 API 계약 사이에 두는 완충지대다.

처음에는 DTO가 비슷한 클래스를 하나 더 만드는 불필요한 작업처럼 보였다.

하지만 API가 많아지고 Entity가 복잡해질수록 요청, 응답, 데이터베이스 모델을 분리해야 하는 이유가 분명해진다.

백엔드 코드를 작성할 때는 다음과 같은 질문을 해볼 수 있다.

- 이 값은 클라이언트가 요청으로 보내는 값인가?
- 이 값이 응답으로 나가도 안전한가?
- Entity의 구조를 외부에서 알 필요가 있는가?
- 이 API에 정말 필요한 필드만 포함되어 있는가?
- 이 요청의 검증 규칙은 어디에 표현되어 있는가?
- 내부 구조가 바뀌어도 API 응답 형식을 유지할 수 있는가?
- 요청과 응답이 서로 다른 이유로 변경될 가능성이 있는가?

이 질문에 답하다 보면 DTO를 왜 사용하는지 자연스럽게 이해할 수 있다.
