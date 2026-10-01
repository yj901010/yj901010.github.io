---
title: "Request DTO와 Response DTO는 왜 따로 만들까?"
date: 2026-08-07 09:23:57 +0900
last_modified_at: 2026-10-01
categories: [Backend, Spring]
tags: [dto, api, spring]
source_url: "https://velog.io/@joker901010/Request-DTO%EC%99%80-Response-DTO%EB%8A%94-%EC%99%9C-%EB%94%B0%EB%A1%9C-%EB%A7%8C%EB%93%A4%EA%B9%8C"
---
**Request DTO는 서버가 받기로 한 입력 계약이고, Response DTO는 서버가 공개하기로 한 출력 계약이다.**

둘 다 데이터를 전달하는 객체이지만, 데이터가 이동하는 방향과 목적이 다르기 때문에 분리하는 것이 안전하다.

---

## DTO가 무엇인지부터 간단히 알아보자

DTO는 **Data Transfer Object**의 약자로, 데이터를 전달하기 위한 객체다.

Spring MVC에서는 클라이언트가 보낸 HTTP 요청 본문을 Java 객체로 변환할 수 있다.

```
HTTP Request Body
→ HttpMessageConverter
→ Java Object
```

반대로 Controller가 반환한 Java 객체를 HTTP 응답 본문으로 변환할 수도 있다.

```
Java Object
→ HttpMessageConverter
→ HTTP Response Body
```

예를 들어 클라이언트가 다음과 같은 JSON을 보냈다고 해보자.

```
{
  "email": "junior@example.com",
  "password": "12345678",
  "name": "junior"
}
```

Spring은 이 JSON을 Java 객체로 변환해 Controller의 파라미터로 전달할 수 있다.

```
@PostMapping("/users")
public void createUser(@RequestBody UserCreateRequest request) {
}
```

반대로 Controller에서 객체를 반환하면 Spring이 JSON 응답으로 변환한다.

```
@GetMapping("/users/{id}")
public UserDetailResponse getUser(@PathVariable Long id) {
    return userService.getUser(id);
}
```

겉으로 보면 요청과 응답 모두 Java 객체를 사용한다.

하지만 역할은 다르다.

```
Request DTO
클라이언트가 서버에 보내는 값을 표현한다.

Response DTO
서버가 클라이언트에게 돌려주는 값을 표현한다.
```

---

## DTO 하나를 요청과 응답에 같이 사용하면 안 될까?

처음 DTO를 배우면 이런 생각이 들 수 있다.

> 회원 정보를 담는 DTO 하나를 만들어 요청과 응답에 모두 사용하면 안 될까?

예를 들어 다음과 같은 DTO가 있다고 해보자.

```
public record UserDto(
    Long id,
    String email,
    String password,
    String name,
    String role,
    LocalDateTime createdAt
) {
}
```

회원가입 요청에도 이 DTO를 사용한다.

```
@PostMapping("/users")
public UserDto createUser(@RequestBody UserDto request) {
    return userService.createUser(request);
}
```

회원 조회 응답에도 같은 DTO를 사용한다.

```
@GetMapping("/users/{id}")
public UserDto getUser(@PathVariable Long id) {
    return userService.getUser(id);
}
```

처음에는 DTO가 하나뿐이라 편해 보인다.

하지만 각각의 필드를 살펴보면 애매한 부분이 생긴다.

- 회원가입 요청에 `id`가 필요한가?
- 회원가입 요청에서 클라이언트가 `role`을 정해도 되는가?
- 회원가입할 때 `createdAt`을 클라이언트가 보내야 하는가?
- 회원 조회 응답에 `password`가 포함되어도 되는가?

하나의 DTO에 요청과 응답의 역할을 모두 넣으면, **어떤 필드가 언제 필요한지 알기 어려워진다.**

---

## 신청서와 결과 통지서로 생각해보자

Request DTO와 Response DTO의 차이는 신청서와 결과 통지서의 차이와 비슷하다.

회원가입 신청서에는 사용자가 작성해야 하는 값이 들어간다.

```
이메일
비밀번호
이름
```

회원가입이 완료된 후 서버가 보내는 결과에는 서버가 생성하거나 확인한 값이 들어간다.

```
회원 ID
이메일
이름
가입 일시
```

가입 신청서에 회원 ID를 적으라고 하면 이상하다.

회원 ID는 아직 회원이 만들어지기 전이므로 서버가 생성해야 하는 값이기 때문이다.

반대로 가입 결과에 비밀번호를 다시 포함하는 것도 위험하다.

비밀번호는 서버가 입력으로 받아 처리해야 하는 값이지, 클라이언트에게 다시 공개해야 하는 값이 아니다.

API도 마찬가지다.

```
Request DTO
서버가 작업을 수행하기 위해 필요한 입력값

Response DTO
작업 결과 중 클라이언트에게 공개할 값
```

---

## 회원가입 API로 직접 나누어보자

회원가입 요청을 예로 들어보자.

클라이언트는 다음과 같은 요청을 보낸다.

```
POST /users HTTP/1.1
Content-Type: application/json

{
  "email": "junior@example.com",
  "password": "12345678",
  "name": "junior"
}
```

회원가입에 필요한 값만 담아 Request DTO를 만든다.

```
public record UserCreateRequest(
    String email,
    String password,
    String name
) {
}
```

Controller에서는 이 DTO로 요청을 받는다.

```
@RestController
@RequestMapping("/users")
public class UserController {

    private final UserService userService;

    public UserController(UserService userService) {
        this.userService = userService;
    }

    @PostMapping
    public UserCreateResponse createUser(
            @RequestBody UserCreateRequest request
    ) {
        return userService.createUser(request);
    }
}
```

회원가입 결과는 별도의 Response DTO로 반환한다.

```
public record UserCreateResponse(
    Long id,
    String email,
    String name,
    LocalDateTime createdAt
) {
}
```

Service에서는 요청을 처리한 뒤 응답 객체를 생성한다.

```
@Service
public class UserService {

    public UserCreateResponse createUser(UserCreateRequest request) {
        User user = new User(
                request.email(),
                encodePassword(request.password()),
                request.name()
        );

        // 실제 코드에서는 Repository를 통해 저장한다.
        // User savedUser = userRepository.save(user);

        return new UserCreateResponse(
                1L,
                user.getEmail(),
                user.getName(),
                LocalDateTime.now()
        );
    }

    private String encodePassword(String password) {
        return "{encoded}" + password;
    }
}
```

이제 두 DTO의 역할이 분명해졌다.

```
UserCreateRequest
회원가입을 위해 클라이언트가 보내야 하는 값

UserCreateResponse
회원가입 후 서버가 클라이언트에게 알려줄 값
```

요청에는 비밀번호가 필요하다.

하지만 응답에는 비밀번호가 없어야 한다.

이 차이만으로도 Request DTO와 Response DTO를 분리해야 하는 이유를 쉽게 이해할 수 있다.

---

## Request DTO는 입력 계약을 표현한다

Request DTO는 단순히 데이터를 담는 객체가 아니다.

클라이언트가 어떤 값을 보내야 하는지, 각 값이 어떤 조건을 만족해야 하는지를 표현할 수 있다.

예를 들어 회원가입 요청에 검증 조건을 추가해보자.

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

Controller에서는 `@Valid`를 사용해 요청값을 검증할 수 있다.

```
@PostMapping
public UserCreateResponse createUser(
        @Valid @RequestBody UserCreateRequest request
) {
    return userService.createUser(request);
}
```

이 Request DTO만 봐도 회원가입 요청의 조건을 알 수 있다.

```
email
- 비어 있으면 안 된다.
- 이메일 형식이어야 한다.

password
- 비어 있으면 안 된다.
- 최소 8자 이상이어야 한다.

name
- 비어 있으면 안 된다.
```

따라서 Request DTO는 API 입력 형식과 검증 규칙을 보여주는 문서 역할도 한다.

---

## Response DTO는 공개 범위를 표현한다

Response DTO는 서버 내부 데이터 중 어떤 값을 클라이언트에게 공개할지 결정한다.

회원 상세 조회 응답을 예로 들어보자.

```
public record UserDetailResponse(
    Long id,
    String email,
    String name,
    String role,
    LocalDateTime createdAt
) {
}
```

이 DTO를 보면 클라이언트가 받을 수 있는 값을 알 수 있다.

```
id
email
name
role
createdAt
```

반면 서버의 `User` Entity에는 다음과 같은 값도 있을 수 있다.

```
password
refreshToken
deleted
internalMemo
updatedBy
```

이 값들은 서버 내부에서는 필요할 수 있지만, 외부 API에 그대로 노출되면 안 될 수 있다.

Entity를 직접 반환하거나 요청과 응답에서 같은 DTO를 사용하면, 의도하지 않은 필드가 응답에 포함될 위험이 커진다.

Response DTO를 별도로 만들면 다음과 같은 의미가 분명해진다.

> 이 API는 이 필드들만 외부에 공개한다.

즉, Response DTO는 응답 형식뿐 아니라 **서버 데이터의 공개 범위**를 표현한다.

---

## 같은 DTO를 사용하면 생길 수 있는 문제

### 1. 요청에 필요 없는 필드가 포함된다

다음 DTO를 회원가입 요청에 사용한다고 해보자.

```
public record UserDto(
    Long id,
    String email,
    String password,
    String name,
    String role,
    LocalDateTime createdAt
) {
}
```

회원가입 요청에서는 다음 값들이 필요하지 않다.

```
id
role
createdAt
```

이 값들은 보통 서버가 생성하거나 결정해야 한다.

그런데 같은 DTO를 사용하면 클라이언트가 보내지 않아도 되는 필드까지 요청 형식에 섞이게 된다.

---

### 2. 응답에 민감한 값이 포함될 수 있다

회원 조회 결과에 `password`가 포함되면 안 된다.

```
{
  "id": 1,
  "email": "junior@example.com",
  "password": "{encoded}12345678",
  "name": "junior"
}
```

암호화된 비밀번호라고 해도 외부로 노출하는 것은 바람직하지 않다.

응답 전용 DTO에 공개할 필드만 선언하면 이러한 실수를 예방하기 쉬워진다.

---

### 3. 검증 규칙이 애매해진다

회원가입 요청에서는 비밀번호가 필수다.

```
회원가입 요청
password 필수
```

하지만 회원 조회 응답에서는 비밀번호가 존재해서는 안 된다.

```
회원 조회 응답
password 미포함
```

같은 DTO를 사용하면 `password`에 `@NotBlank`를 붙여야 하는지, 응답에서는 어떻게 처리해야 하는지 애매해진다.

요청 DTO를 따로 만들면 요청에 필요한 검증 규칙만 선언할 수 있다.

---

### 4. 클라이언트가 보내면 안 되는 값을 보낼 수 있다

예를 들어 회원가입 요청 DTO에 `role`이 포함되어 있다고 해보자.

```
{
  "email": "junior@example.com",
  "password": "12345678",
  "name": "junior",
  "role": "ADMIN"
}
```

서버 코드가 이를 제대로 막지 않으면 일반 사용자가 관리자 권한을 요청하는 문제가 발생할 수 있다.

물론 최종적으로는 서버의 비즈니스 로직에서 권한을 검증해야 한다.

하지만 애초에 회원가입 Request DTO에 `role` 필드가 없다면 클라이언트가 해당 값을 전달할 이유 자체가 줄어든다.

Request DTO를 분리하면 **클라이언트가 입력할 수 있는 범위**를 명확히 제한할 수 있다.

---

### 5. 요청과 응답의 변경이 서로 영향을 준다

회원 목록 화면에 가입 일시가 필요해져서 DTO에 `createdAt`을 추가했다고 해보자.

하나의 DTO를 요청과 응답에서 함께 사용하면, 응답을 위해 추가한 필드가 요청 DTO에도 함께 나타난다.

반대로 회원가입 요청에 새로운 동의 여부를 추가했는데, 같은 DTO를 사용하면 조회 응답에도 불필요한 필드가 나타날 수 있다.

요청과 응답을 분리하면 변경 범위를 줄일 수 있다.

```
요청 형식 변경
→ Request DTO만 수정

응답 형식 변경
→ Response DTO만 수정
```

---

## Entity를 그대로 반환하면 안 될까?

다음처럼 Entity를 Controller에서 직접 반환할 수도 있다.

```
@GetMapping("/users/{id}")
public User getUser(@PathVariable Long id) {
    return userService.getUser(id);
}
```

기술적으로는 동작할 수 있다.

하지만 Entity는 데이터베이스와 도메인 로직을 표현하기 위한 객체다.

반면 Response DTO는 외부 API의 응답 형식을 표현한다.

Entity를 그대로 반환하면 다음과 같은 문제가 생길 수 있다.

- 내부 필드가 의도치 않게 노출될 수 있다.
- Entity 변경이 API 응답 변경으로 이어질 수 있다.
- 연관관계 직렬화 문제가 발생할 수 있다.
- 지연 로딩이나 순환 참조 문제가 생길 수 있다.
- API에 필요 없는 데이터까지 응답할 수 있다.

따라서 외부에 제공하는 API라면 Entity와 Response DTO를 분리하는 편이 안전하다.

---

## DTO 이름은 어떻게 지으면 좋을까?

DTO 이름에는 역할과 사용 목적이 드러나는 것이 좋다.

다음 이름은 범위가 너무 넓다.

```
UserDto
UserInfo
UserData
UserParam
```

프로젝트가 작을 때는 문제없어 보이지만, DTO가 많아지면 각각 어디에 사용하는 객체인지 알기 어려워진다.

조금 더 구체적으로 이름을 지을 수 있다.

```
UserCreateRequest
UserUpdateRequest
UserPasswordChangeRequest

UserCreateResponse
UserDetailResponse
UserSummaryResponse
```

이름을 구성하는 단어에도 의미가 있다.

```
Request
클라이언트가 서버에 보내는 값

Response
서버가 클라이언트에게 보내는 값

Create
생성 요청 또는 생성 결과

Update
수정 요청 또는 수정 결과

Detail
상세 조회 응답

Summary
목록이나 요약 응답
```

DTO 이름이 조금 길어지더라도 역할이 명확하다면 코드를 읽기 쉬워진다.

---

## 모든 API마다 DTO를 무조건 나누어야 할까?

Request DTO와 Response DTO를 분리하는 것이 일반적으로 안전하지만, 모든 상황에서 무조건 많은 클래스를 만들어야 한다는 뜻은 아니다.

아주 작은 내부 도구나 실험 코드에서는 하나의 DTO만으로 충분할 수도 있다.

예를 들어 다음 조건을 모두 만족한다면 하나의 DTO를 사용해도 큰 문제가 없을 수 있다.

- 요청과 응답 필드가 완전히 같다.
- 민감한 정보가 없다.
- 외부에 공개되는 API가 아니다.
- 변경 가능성이 낮다.
- 코드의 수명이 짧다.

하지만 다음 조건이 있다면 분리하는 편이 좋다.

- 비밀번호와 같은 민감한 값이 있다.
- 요청과 응답에 필요한 필드가 다르다.
- 외부 클라이언트가 사용하는 API다.
- 검증 규칙이 복잡하다.
- API가 장기간 유지될 예정이다.
- 요청과 응답의 변경 주기가 다르다.

중요한 것은 DTO 개수를 줄이는 것이 아니라, 각 객체의 역할을 명확하게 유지하는 것이다.

---

## Request DTO를 Service까지 그대로 넘겨도 될까?

작은 프로젝트에서는 Controller가 받은 Request DTO를 Service로 그대로 전달해도 괜찮다.

```
userService.createUser(request);
```

구조가 단순하고 코드도 이해하기 쉽다.

하지만 프로젝트가 커지면 Service 계층이 Controller의 요청 형식에 강하게 의존하는 문제가 생길 수 있다.

예를 들어 HTTP API가 아닌 배치 작업이나 메시지 소비 기능에서도 회원 생성 기능을 사용해야 할 수 있다.

이때는 Service 전용 입력 모델인 Command 객체를 만들기도 한다.

```
public record CreateUserCommand(
    String email,
    String password,
    String name
) {
}
```

Controller에서 Request DTO를 Command로 변환한다.

```
@PostMapping
public UserCreateResponse createUser(
        @Valid @RequestBody UserCreateRequest request
) {
    CreateUserCommand command = new CreateUserCommand(
            request.email(),
            request.password(),
            request.name()
    );

    return userService.createUser(command);
}
```

다만 처음부터 Request DTO, Command, Domain 객체를 모두 나누면 오히려 구조가 복잡해질 수 있다.

프로젝트 규모가 작다면 Request DTO를 Service로 전달하고, 계층 간 의존성이 문제가 되기 시작할 때 분리해도 된다.

---

## Response DTO는 어디에서 변환해야 할까?

Entity나 도메인 객체를 Response DTO로 변환하는 위치에는 여러 선택지가 있다.

### Service에서 변환

```
public UserDetailResponse getUser(Long id) {
    User user = userRepository.findById(id)
            .orElseThrow();

    return new UserDetailResponse(
            user.getId(),
            user.getEmail(),
            user.getName(),
            user.getRole(),
            user.getCreatedAt()
    );
}
```

### Controller에서 변환

```
@GetMapping("/{id}")
public UserDetailResponse getUser(@PathVariable Long id) {
    User user = userService.getUser(id);

    return UserDetailResponse.from(user);
}
```

### Response DTO에 변환 메서드 작성

```
public record UserDetailResponse(
    Long id,
    String email,
    String name,
    String role,
    LocalDateTime createdAt
) {

    public static UserDetailResponse from(User user) {
        return new UserDetailResponse(
                user.getId(),
                user.getEmail(),
                user.getName(),
                user.getRole(),
                user.getCreatedAt()
        );
    }
}
```

### 별도의 Mapper 사용

```
@Component
public class UserMapper {

    public UserDetailResponse toDetailResponse(User user) {
        return new UserDetailResponse(
                user.getId(),
                user.getEmail(),
                user.getName(),
                user.getRole(),
                user.getCreatedAt()
        );
    }
}
```

프로젝트가 커지면 MapStruct와 같은 매핑 도구를 사용하기도 한다.

어떤 방식이 항상 정답인 것은 아니다.

중요한 것은 팀에서 변환 위치와 방식을 정하고 일관되게 사용하는 것이다.

---

## 실무에서 기억하면 좋은 점

### 필요한 값만 응답한다

응답 DTO에 “나중에 사용할 수도 있는 값”을 미리 넣을 필요는 없다.

API 응답에 공개된 필드는 클라이언트가 사용하기 시작할 수 있다.

필드가 많아질수록 클라이언트의 의존성도 커지고, 나중에 필드를 제거하거나 의미를 변경하기 어려워진다.

따라서 현재 API에 필요한 값만 응답하는 것이 좋다.

---

### 프론트엔드에서 검증해도 서버 검증은 필요하다

프론트엔드에서 이메일 형식이나 비밀번호 길이를 검사하더라도 백엔드의 검증은 필요하다.

HTTP 요청은 브라우저 화면을 거치지 않고도 직접 보낼 수 있기 때문이다.

Request DTO의 Validation은 서버가 잘못된 입력을 차단하는 첫 번째 단계다.

---

### 민감한 값은 Response DTO에 넣지 않는다

다음과 같은 값은 응답에 포함할 때 특히 주의해야 한다.

```
비밀번호
액세스 토큰
리프레시 토큰
주민등록번호
내부 관리 메모
삭제 여부
내부 권한 정보
서버 내부 식별자
```

Entity에 존재한다는 이유만으로 응답에 포함해서는 안 된다.

Response DTO에는 클라이언트가 실제로 사용해야 하는 값만 선언해야 한다.

---

## 정리

Request DTO와 Response DTO는 모두 데이터를 전달하기 위한 객체다.

하지만 방향과 목적이 다르다.

### Request DTO

```
클라이언트가 서버에 보내는 입력값을 표현한다.
클라이언트가 보낼 수 있는 필드의 범위를 정한다.
입력값의 검증 규칙을 표현한다.
```

예시는 다음과 같다.

```
UserCreateRequest
UserUpdateRequest
UserPasswordChangeRequest
```

### Response DTO

```
서버가 클라이언트에게 반환할 결과를 표현한다.
외부에 공개할 필드의 범위를 정한다.
API 응답의 형식을 표현한다.
```

예시는 다음과 같다.

```
UserCreateResponse
UserDetailResponse
UserSummaryResponse
```

하나의 DTO를 요청과 응답에 함께 사용하면 처음에는 편할 수 있다.

하지만 시간이 지나면 다음과 같은 문제가 생길 수 있다.

```
요청에 필요 없는 필드가 포함된다.
클라이언트가 보내면 안 되는 값을 보낼 수 있다.
응답에 민감한 값이 노출될 수 있다.
Validation 규칙이 애매해진다.
요청과 응답의 변경이 서로 영향을 준다.
API의 의도가 불분명해진다.
```

따라서 DTO를 만들기 전에 다음 질문을 해보면 좋다.

```
이 DTO는 요청용인가, 응답용인가?

이 값은 클라이언트가 보내야 하는가?

이 값은 서버가 생성하는가?

이 필드는 외부에 공개해도 되는가?

이 검증 규칙은 요청에만 필요한가?

요청과 응답이 서로 다른 이유로 변경될 가능성이 있는가?
```

Request DTO와 Response DTO를 분리하는 이유는 단순히 클래스를 더 많이 만들기 위해서가 아니다.

**입력할 수 있는 값과 공개할 수 있는 값을 명확히 구분하고, API를 안전하게 변경하기 위해서다.**

다시 한 문장으로 정리하면 다음과 같다.

> Request DTO는 서버가 받기로 한 입력 계약이고, Response DTO는 서버가 공개하기로 한 출력 계약이다.
