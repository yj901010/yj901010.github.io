---
title: "엔터티를 API 응답으로 반환하면 안되는 이유"
date: 2026-08-05 09:05:40 +0900
last_modified_at: 2026-10-01
categories: [Backend, JPA]
tags: [entity, dto, jpa, api]
source_url: "https://velog.io/@joker901010/%EC%97%94%ED%84%B0%ED%8B%B0%EB%A5%BC-API-%EC%9D%91%EB%8B%B5%EC%9C%BC%EB%A1%9C-%EB%B0%98%ED%99%98%ED%95%98%EB%A9%B4-%EC%95%88%EB%90%98%EB%8A%94-%EC%9D%B4%EC%9C%A0"
---
JPA Entity를 API 응답으로 그대로 반환하면 서버 내부 정보가 노출될 수 있고, Entity 변경이 API 변경으로 이어질 수 있다.

또한 연관관계로 인해 응답 데이터가 예상보다 커지거나, 순환 참조와 Lazy Loading 문제가 발생할 수 있다.

그래서 일반적으로 API 응답에는 Entity 대신 **Response DTO**를 사용한다.

---

## Entity를 반환해도 정상적으로 동작하는 이유

Spring MVC에서는 Controller가 반환한 Java 객체를 JSON으로 변환해 HTTP 응답 본문에 담을 수 있다.

예를 들어 다음과 같이 `User` 객체를 반환할 수 있다.

```
@GetMapping("/users/{id}")
public User getUser(@PathVariable Long id) {
    return userRepository.findById(id)
            .orElseThrow();
}
```

이 코드는 정상적으로 동작할 수 있다.

Spring MVC가 `HttpMessageConverter`를 이용해 Controller의 반환값을 HTTP 응답으로 변환하기 때문이다.

일반적인 Spring Boot 프로젝트에서는 Jackson이 Java 객체를 JSON으로 직렬화한다.

즉, 다음과 같은 흐름으로 처리된다.

```
Controller가 User 객체 반환
        ↓
Jackson이 User 객체를 JSON으로 변환
        ↓
클라이언트에게 JSON 응답 전송
```

처음에는 이 방식이 매우 편해 보인다.

- DTO를 만들 필요가 없다.
- Entity를 DTO로 변환하는 코드가 필요 없다.
- Controller 코드가 짧아진다.

그래서 다음과 같은 생각이 들 수 있다.

> Entity를 바로 반환하면 코드도 짧고 간단한데, 굳이 DTO를 만들어야 할까?

하지만 여기서 중요한 질문이 하나 있다.

> 이 Entity에 들어 있는 모든 필드를 클라이언트에게 공개해도 되는가?

대부분의 실제 서비스에서는 그렇지 않다.

Spring은 객체를 JSON으로 잘 변환해 준다. 하지만 **어떤 정보를 외부에 공개할 것인지는 개발자가 직접 결정해야 한다.**

---

## Entity와 Response DTO의 차이

Entity는 일반적으로 데이터베이스와 서버 내부 로직에 가까운 객체다.

반면 Response DTO는 클라이언트에게 공개할 API 응답을 표현하는 객체다.

간단히 구분하면 다음과 같다.

```
Entity
- 데이터베이스 저장 구조와 가까움
- 서버 내부 처리에 사용됨
- 외부에 공개하면 안 되는 필드가 포함될 수 있음

Response DTO
- 클라이언트에게 보낼 응답 구조
- 공개하기로 결정한 필드만 포함
- API 목적에 맞게 별도로 설계
```

Entity를 회사의 내부 문서라고 생각하면 이해하기 쉽다.

내부 문서에는 다음과 같은 정보가 들어갈 수 있다.

- 고객 이름
- 고객 등급
- 담당 직원 ID
- 삭제 여부
- 내부 관리 상태
- 담당자 메모
- 마지막 수정자

하지만 고객에게 보내는 안내문에는 필요한 내용만 들어가야 한다.

- 고객 이름
- 고객 등급
- 사용할 수 있는 혜택

내부 문서를 그대로 고객에게 보내면 필요하지 않은 정보까지 노출될 수 있다.

백엔드에서도 마찬가지다.

**Entity를 그대로 응답하는 것은 서버 내부 데이터 모델을 외부에 그대로 공개하는 것과 비슷하다.**

---

## 예제로 살펴보기

다음과 같은 회원 Entity가 있다고 해보자.

```
@Entity
public class User {

    @Id
    private Long id;

    private String email;

    private String password;

    private String name;

    private String role;

    private boolean deleted;

    private String internalMemo;

    protected User() {
    }

    public Long getId() {
        return id;
    }

    public String getEmail() {
        return email;
    }

    public String getPassword() {
        return password;
    }

    public String getName() {
        return name;
    }

    public String getRole() {
        return role;
    }

    public boolean isDeleted() {
        return deleted;
    }

    public String getInternalMemo() {
        return internalMemo;
    }
}
```

이 Entity를 Controller에서 그대로 반환하면 다음과 같은 JSON이 응답될 수 있다.

```
{
  "id": 1,
  "email": "junior@example.com",
  "password": "$2a$10$encrypted-password",
  "name": "junior",
  "role": "USER",
  "deleted": false,
  "internalMemo": "VIP 전환 검토"
}
```

클라이언트가 실제로 필요한 값은 `id`, `email`, `name`뿐일 수 있다.

하지만 Entity를 그대로 반환하면서 다음 정보까지 함께 노출됐다.

- 암호화된 비밀번호
- 사용자 권한
- 삭제 여부
- 내부 관리 메모

비밀번호는 암호화되어 있더라도 외부 응답으로 내려가면 안 된다.

화면에서 해당 값을 표시하지 않는다고 안전한 것도 아니다. API 응답에 포함된 데이터는 브라우저 개발자 도구나 네트워크 요청을 통해 확인할 수 있다.

따라서 민감한 정보는 프론트엔드에서 숨기는 것이 아니라, **서버에서 처음부터 보내지 않아야 한다.**

---

## Response DTO를 사용해 필요한 값만 반환하기

API에 필요한 필드만 담는 Response DTO를 만들어 보자.

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

Controller에서는 Entity가 아니라 `UserResponse`를 반환한다.

```
@GetMapping("/users/{id}")
public UserResponse getUser(@PathVariable Long id) {
    User user = userService.getUser(id);
    return UserResponse.from(user);
}
```

이제 클라이언트에게는 다음 정보만 전달된다.

```
{
  "id": 1,
  "email": "junior@example.com",
  "name": "junior"
}
```

DTO에 정의한 필드만 응답에 포함되기 때문에 공개 범위를 명확하게 통제할 수 있다.

---

# Entity를 그대로 응답하면 위험한 이유

## 1. 민감한 내부 필드가 노출될 수 있다

Entity에는 데이터베이스 저장이나 서버 내부 처리를 위해 다양한 필드가 포함될 수 있다.

예를 들면 다음과 같다.

```
password
refreshToken
phoneNumber
address
residentNumber
internalMemo
deleted
createdBy
updatedBy
```

이 값들이 데이터베이스에 필요하다고 해서 클라이언트에게도 필요한 것은 아니다.

Entity를 그대로 JSON으로 직렬화하면 개발자가 의도하지 않은 필드까지 응답에 포함될 수 있다.

특히 다음과 같은 상황은 위험하다.

```
서버가 많은 데이터를 반환함
        ↓
프론트엔드는 필요한 일부 데이터만 화면에 표시함
        ↓
사용자는 화면에 보이는 데이터만 존재한다고 생각함
        ↓
공격자는 실제 API 응답을 확인해 숨겨진 필드를 발견함
```

중요한 점은 다음과 같다.

> 화면에 표시되지 않는 것과 API 응답에 존재하지 않는 것은 전혀 다르다.

클라이언트에게 전달된 데이터는 숨겨진 데이터가 아니다.

DTO를 사용하면 필요한 필드만 명시적으로 선택할 수 있어 이러한 실수를 줄일 수 있다.

---

## 2. Entity 구조가 API 응답 구조가 된다

Entity를 그대로 반환하면 Entity의 필드 구조가 JSON 응답 구조와 강하게 연결된다.

예를 들어 처음에는 다음과 같은 필드가 있었다고 해보자.

```
private String name;
```

이후 의미를 더 명확하게 표현하기 위해 필드명을 바꿨다.

```
private String username;
```

Entity를 그대로 반환하고 있다면 JSON 필드명도 함께 바뀔 수 있다.

변경 전 응답은 다음과 같다.

```
{
  "name": "junior"
}
```

변경 후에는 다음과 같이 바뀔 수 있다.

```
{
  "username": "junior"
}
```

서버 내부에서는 단순한 변수명 변경이라고 생각했지만, 클라이언트 입장에서는 API 응답 규격이 변경된 것이다.

기존에 `name`을 사용하던 웹이나 앱은 정상적으로 동작하지 않을 수 있다.

API 응답은 클라이언트와 서버 사이의 약속이다.

따라서 데이터베이스나 Entity의 내부 변경이 외부 API 변경으로 바로 이어지지 않도록 분리하는 것이 좋다.

DTO를 사용하면 Entity의 필드명이 변경되어도 API 응답은 유지할 수 있다.

```
public record UserResponse(
        String name
) {

    public static UserResponse from(User user) {
        return new UserResponse(user.getUsername());
    }
}
```

Entity에서는 `username`을 사용하지만, API 응답에서는 계속 `name`을 제공할 수 있다.

```
{
  "name": "junior"
}
```

DTO는 Entity와 API 사이에서 완충지대 역할을 한다.

---

## 3. 연관관계 때문에 응답이 예상보다 커질 수 있다

JPA Entity에는 다른 Entity와의 연관관계가 포함될 수 있다.

예를 들어 사용자와 주문이 양방향 연관관계를 맺고 있다고 해보자.

```
@Entity
public class User {

    @Id
    private Long id;

    private String name;

    @OneToMany(mappedBy = "user")
    private List<Order> orders = new ArrayList<>();
}
```

```
@Entity
public class Order {

    @Id
    private Long id;

    @ManyToOne
    private User user;
}
```

`User`는 여러 개의 `Order`를 가지고 있고, 각 `Order`는 다시 `User`를 참조한다.

이 Entity를 그대로 JSON으로 변환하면 다음과 같이 객체를 계속 따라갈 수 있다.

```
User
 └─ orders
     └─ Order
         └─ user
             └─ orders
                 └─ Order
                     └─ user
                         ...
```

이것을 **순환 참조**라고 한다.

설정과 구조에 따라 직렬화 오류가 발생하거나, 같은 객체를 계속 탐색하는 문제가 생길 수 있다.

순환 참조가 발생하지 않더라도 응답 크기가 예상보다 커질 수 있다.

클라이언트는 사용자 이름만 필요했는데 다음 데이터가 함께 조회되고 응답될 수 있다.

```
사용자
 └─ 주문 목록
     └─ 주문 상세
         └─ 상품
             └─ 판매자
```

DTO를 사용하면 API 목적에 필요한 깊이까지만 응답하도록 설계할 수 있다.

사용자 기본 정보만 필요하다면 다음처럼 만들 수 있다.

```
public record UserResponse(
        Long id,
        String name
) {
}
```

주문 전체 목록이 아니라 주문 개수만 필요하다면 다음처럼 표현할 수 있다.

```
public record UserDetailResponse(
        Long id,
        String name,
        int orderCount
) {
}
```

DTO를 사용하면 어떤 연관 데이터를 어디까지 공개할지 명확하게 결정할 수 있다.

---

## 4. JSON 변환 중 Lazy Loading이 발생할 수 있다

JPA 연관관계는 지연 로딩으로 설정할 수 있다.

```
@OneToMany(mappedBy = "user", fetch = FetchType.LAZY)
private List<Order> orders = new ArrayList<>();
```

지연 로딩은 연관된 데이터를 처음부터 조회하지 않고, 실제로 해당 데이터에 접근할 때 조회하는 방식이다.

예를 들어 사용자를 조회할 때 주문 목록을 바로 가져오지 않고, 나중에 `getOrders()`가 호출되는 순간 주문을 조회할 수 있다.

Entity를 그대로 응답하면 Jackson이 JSON을 만드는 과정에서 Entity의 getter를 호출할 수 있다.

```
Controller가 User Entity 반환
        ↓
Jackson이 JSON 변환 시작
        ↓
Jackson이 getOrders() 호출
        ↓
지연 로딩된 주문 데이터 조회
```

개발자가 Service에서 직접 주문을 사용하지 않았더라도 JSON 직렬화 과정에서 추가 조회가 발생할 수 있다.

이로 인해 다음과 같은 문제가 생길 수 있다.

- 트랜잭션이 끝난 뒤 Lazy Loading이 발생해 예외가 발생한다.
- 예상하지 못한 SQL이 실행된다.
- 사용자 여러 명의 연관 데이터를 각각 조회해 N+1 문제가 발생한다.
- 응답 생성 시점의 쿼리 수를 예측하기 어려워진다.

DTO를 사용하면 Service 또는 조회 단계에서 필요한 데이터를 명확하게 준비할 수 있다.

```
@Transactional(readOnly = true)
public UserResponse getUser(Long userId) {
    User user = userRepository.findById(userId)
            .orElseThrow();

    return UserResponse.from(user);
}
```

더 복잡한 조회라면 Fetch Join, DTO Projection, QueryDSL 등을 사용해 필요한 데이터만 직접 조회할 수도 있다.

핵심은 다음과 같다.

> JSON 직렬화 과정이 데이터 조회 범위를 결정하게 두지 말고, 애플리케이션 코드가 필요한 조회 범위를 명확하게 결정해야 한다.

---

## 5. API 목적에 맞는 응답을 만들기 어렵다

같은 `User` 데이터라도 API마다 필요한 정보가 다르다.

### 사용자 목록 조회

목록에서는 간단한 정보만 필요할 수 있다.

```
{
  "id": 1,
  "name": "junior"
}
```

### 사용자 상세 조회

상세 화면에서는 이메일과 가입 일자가 필요할 수 있다.

```
{
  "id": 1,
  "name": "junior",
  "email": "junior@example.com",
  "createdAt": "2026-08-04T09:00:00"
}
```

### 관리자 사용자 조회

관리자 화면에서는 권한과 삭제 여부가 필요할 수 있다.

```
{
  "id": 1,
  "name": "junior",
  "email": "junior@example.com",
  "role": "USER",
  "deleted": false
}
```

Entity 하나로 이 모든 응답 형태를 표현하려 하면 관리가 어려워진다.

특정 필드를 숨기거나 보여주기 위한 조건이 Entity 내부에 계속 추가될 수 있고, API마다 필요한 응답 범위를 구분하기도 어려워진다.

DTO를 API 목적별로 나누면 더 명확하다.

```
UserSummaryResponse
UserDetailResponse
AdminUserResponse
```

예를 들어 다음과 같이 작성할 수 있다.

```
public record UserSummaryResponse(
        Long id,
        String name
) {
}
```

```
public record UserDetailResponse(
        Long id,
        String name,
        String email,
        LocalDateTime createdAt
) {
}
```

```
public record AdminUserResponse(
        Long id,
        String name,
        String email,
        String role,
        boolean deleted
) {
}
```

각 DTO의 이름만 보더라도 어떤 API에서 어떤 목적으로 사용하는지 이해하기 쉬워진다.

---

## `@JsonIgnore`를 붙이면 해결되지 않을까?

특정 필드를 JSON 응답에서 제외하려면 `@JsonIgnore`를 사용할 수 있다.

```
@JsonIgnore
private String password;
```

이렇게 하면 Jackson이 해당 필드를 JSON으로 변환하지 않는다.

민감한 필드가 실수로 직렬화되는 것을 막는 데 도움이 될 수 있다.

하지만 `@JsonIgnore`만으로 Entity 직접 반환의 모든 문제를 해결하기는 어렵다.

가장 큰 이유는 API마다 필요한 필드가 다르기 때문이다.

예를 들어 일반 사용자 API와 관리자 API의 응답이 다음과 같다고 해보자.

### 일반 사용자 조회

```
{
  "id": 1,
  "name": "junior"
}
```

### 관리자 사용자 조회

```
{
  "id": 1,
  "name": "junior",
  "email": "junior@example.com",
  "role": "USER",
  "deleted": false
}
```

일반 사용자 API에서는 `role`과 `deleted`를 숨기고 싶지만, 관리자 API에서는 보여줘야 한다.

그런데 Entity 필드에 `@JsonIgnore`를 붙이면 모든 API에서 해당 필드가 제외된다.

반대로 `@JsonIgnore`를 붙이지 않으면 일반 사용자 API에서도 필드가 노출될 수 있다.

또한 Entity에 JSON 직렬화 정책이 계속 추가되면 Entity가 너무 많은 책임을 가지게 된다.

```
Entity의 원래 역할
- 데이터베이스 매핑
- 도메인 상태와 행위 표현

추가되는 역할
- 어떤 API 필드를 숨길지 결정
- JSON 직렬화 방식을 결정
- 연관관계 직렬화 방향을 결정
```

Entity가 데이터베이스 모델뿐 아니라 API 응답 정책까지 알아야 하는 구조가 되는 것이다.

`@JsonIgnore`는 특정 직렬화 문제를 방지하는 보조 수단으로는 사용할 수 있다.

하지만 외부 API 응답을 설계하는 기본 수단으로는 Response DTO가 더 명확하다.

---

## DTO를 사용하면 무조건 Lazy Loading과 N+1이 해결될까?

DTO를 사용한다고 해서 Lazy Loading이나 N+1 문제가 자동으로 해결되는 것은 아니다.

다음 코드처럼 DTO를 만드는 과정에서 지연 로딩된 연관관계를 반복해서 조회하면 여전히 N+1 문제가 발생할 수 있다.

```
public static UserDetailResponse from(User user) {
    return new UserDetailResponse(
            user.getId(),
            user.getName(),
            user.getOrders().size()
    );
}
```

사용자 목록을 조회한 뒤 각 사용자마다 `getOrders()`를 호출하면 사용자 수만큼 추가 쿼리가 발생할 수 있다.

따라서 DTO와 조회 전략은 함께 설계해야 한다.

필요에 따라 다음 방법을 사용할 수 있다.

- Fetch Join
- Entity Graph
- DTO Projection
- QueryDSL
- Batch Size 설정
- 별도의 집계 쿼리

DTO의 장점은 N+1을 자동으로 없애는 것이 아니다.

DTO를 통해 **API에 필요한 데이터의 범위를 먼저 명확히 정하고, 그 범위에 맞는 조회 쿼리를 설계할 수 있다는 것**이 핵심이다.

---

## 모든 상황에서 반드시 DTO를 사용해야 할까?

Entity를 직접 반환한다고 해서 모든 코드가 즉시 잘못되는 것은 아니다.

다음과 같은 환경에서는 Entity 직접 반환을 선택할 수도 있다.

- 학습용 예제
- 빠르게 검증하는 프로토타입
- 외부에 공개되지 않는 작은 내부 도구
- Entity와 응답 구조가 매우 단순한 일회성 기능

하지만 프로젝트가 커지거나 외부 클라이언트가 API를 사용하기 시작하면 다음 요구사항이 생긴다.

- 특정 필드만 공개해야 한다.
- API 버전을 안정적으로 유지해야 한다.
- 화면마다 다른 응답이 필요하다.
- 연관관계의 조회 범위를 통제해야 한다.
- 보안 검토가 필요하다.
- 프론트엔드와 응답 규격을 협의해야 한다.

이때 Entity와 API 응답이 직접 연결되어 있으면 변경 범위가 커진다.

따라서 외부 클라이언트가 사용하는 API라면 Entity를 직접 반환하지 않고 Response DTO를 사용하는 것을 기본 원칙으로 두는 편이 안전하다.

---

# 실무에서 기억할 점

## 1. 외부 API에는 Entity를 그대로 반환하지 않는다

Entity는 서버 내부 모델이고, API 응답은 외부에 공개되는 계약이다.

두 객체의 역할을 분리하는 것이 좋다.

```
Repository
    ↓
Entity 조회
    ↓
Service에서 필요한 데이터 구성
    ↓
Response DTO 변환
    ↓
Controller가 DTO 응답
```

---

## 2. DTO에도 필요한 필드만 넣는다

DTO를 만들었다고 해서 내부 필드를 모두 옮겨 담으면 의미가 줄어든다.

좋은 DTO는 해당 API에 필요한 값만 포함한다.

```
public record UserResponse(
        Long id,
        String name
) {
}
```

다음과 같이 필요할지 모르는 필드를 미리 전부 넣지 않는 것이 좋다.

```
public record UserResponse(
        Long id,
        String name,
        String email,
        String role,
        boolean deleted,
        String internalMemo,
        String createdBy,
        String updatedBy
) {
}
```

필요해졌을 때 API 목적과 공개 가능 여부를 검토한 뒤 추가해야 한다.

---

## 3. 민감한 정보는 서버에서 제외한다

프론트엔드에서 화면에 표시하지 않는 것은 보안 조치가 아니다.

API 응답에 포함된 데이터는 사용자가 확인할 수 있다.

따라서 다음과 같이 생각해야 한다.

```
잘못된 생각:
프론트엔드에서 안 보여주면 된다.

올바른 생각:
클라이언트에게 필요하지 않은 데이터는 서버가 보내지 않는다.
```

---

## 4. Entity 변경과 API 변경을 분리한다

데이터베이스 컬럼이나 Entity 필드가 변경됐다고 해서 외부 API까지 반드시 변경되어야 하는 것은 아니다.

DTO가 있으면 내부 구조를 변경하면서도 기존 API 응답을 유지할 수 있다.

```
Entity의 변경 이유
- 데이터베이스 구조 개선
- 도메인 용어 변경
- 내부 코드 리팩터링

API의 변경 이유
- 클라이언트 요구사항 변경
- 새로운 응답 정보 추가
- API 버전 변경
```

서로 변경 이유가 다르므로 객체도 분리하는 것이 자연스럽다.

---

## 5. 연관관계는 필요한 범위만 조회하고 응답한다

사용자 상세 API를 만든다고 해서 사용자의 모든 연관 데이터를 내려줄 필요는 없다.

먼저 API의 목적을 정해야 한다.

- 주문 전체 목록이 필요한가?
- 주문 개수만 필요한가?
- 최근 주문 3개만 필요한가?
- 주문 상태별 집계만 필요한가?

목적이 정해지면 그에 맞는 DTO와 조회 쿼리를 작성한다.

```
필요한 만큼만 조회한다.
필요한 만큼만 응답한다.
```

---

# 정리

Entity를 API 응답으로 그대로 반환하면 처음에는 편하다.

- DTO를 만들지 않아도 된다.
- 변환 코드가 필요 없다.
- Controller 코드가 짧아진다.

하지만 API가 커질수록 다음 문제가 발생할 가능성이 높아진다.

- 민감한 내부 필드가 노출될 수 있다.
- Entity 구조가 API 응답 구조가 된다.
- Entity 변경이 클라이언트 오류로 이어질 수 있다.
- 연관관계 때문에 응답이 지나치게 커질 수 있다.
- 양방향 연관관계에서 순환 참조가 발생할 수 있다.
- JSON 변환 중 Lazy Loading이 발생할 수 있다.
- 예상하지 못한 추가 쿼리와 N+1 문제가 생길 수 있다.
- API마다 다른 응답 형태를 만들기 어려워진다.

Entity와 Response DTO의 관계는 다음과 같이 이해할 수 있다.

> Entity는 서버 내부에서 사용하는 데이터 모델이고, Response DTO는 외부에 공개하기로 결정한 API 모델이다.

API를 설계할 때는 다음 질문을 해보는 것이 좋다.

1. 이 필드는 클라이언트에게 정말 필요한가?
2. 이 값이 외부에 노출되어도 안전한가?
3. Entity가 변경되어도 API 응답을 유지할 수 있는가?
4. 연관 데이터를 어디까지 조회해야 하는가?
5. 이 응답이 현재 API의 목적에 맞는가?

이 질문에 답하면서 응답 DTO를 설계하면, Entity를 그대로 반환했을 때 생길 수 있는 보안 문제와 유지보수 문제를 줄일 수 있다.
