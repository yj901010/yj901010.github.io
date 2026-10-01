---
title: "좋은 메서드 이름은 어떻게 지을까?"
date: 2026-08-13 09:54:51 +0900
last_modified_at: 2026-10-01
categories: [Engineering, Code Quality]
tags: [naming, method, clean-code]
source_url: "https://velog.io/@joker901010/%EC%A2%8B%EC%9D%80-%EB%A9%94%EC%84%9C%EB%93%9C-%EC%9D%B4%EB%A6%84%EC%9D%80-%EC%96%B4%EB%96%BB%EA%B2%8C-%EC%A7%80%EC%9D%84%EA%B9%8C"
---
코드를 작성하다 보면 생각보다 자주 멈추게 되는 순간이 있다.

바로 **메서드 이름을 지을 때**다.

```
process()
handle()
execute()
get()
```

처음에는 이런 이름도 크게 이상해 보이지 않는다.

어차피 내가 작성한 코드이고, 메서드 안에 무엇이 들어 있는지 알고 있기 때문이다.

하지만 며칠 뒤 다시 코드를 보거나 다른 개발자가 코드를 읽는 순간 이야기가 달라진다.

```
processOrder();
```

이 메서드는 정확히 무엇을 할까?

- 주문을 검증하는 걸까?
- 주문을 저장하는 걸까?
- 재고를 차감하는 걸까?
- 결제를 요청하는 걸까?
- 이 모든 작업을 한 번에 하는 걸까?

결국 메서드 내부를 열어봐야 알 수 있다.

반대로 이런 코드는 어떨까?

```
validateOrder();
decreaseStock();
requestPayment();
sendOrderCompletedNotification();
```

메서드 내부를 열어보지 않아도 대략 어떤 작업이 이루어지는지 알 수 있다.

좋은 메서드 이름의 핵심은 여기에서 시작한다.

> **메서드 이름만 보고도 무엇을 하는 코드인지 어느 정도 예상할 수 있어야 한다.**

---

## Java 메서드 이름의 기본 규칙

먼저 가장 기본적인 규칙부터 살펴보자.

Google Java Style Guide에서는 Java 메서드 이름을 `lowerCamelCase`로 작성하고, 일반적으로 **동사 또는 동사구**를 사용한다고 설명한다.

예를 들면 다음과 같다.

```
sendMessage();
stop();
createOrder();
changePassword();
```

즉 기본적인 형태는 어렵지 않다.

```
lowerCamelCase를 사용한다.
동사 또는 동사구를 사용한다.
```

하지만 실무에서 어려운 것은 문법이 아니다.

다음 이름은 모두 Java 문법상 문제가 없다.

```
getUser();
findUser();
loadUser();
createUser();
registerUser();
```

그렇다면 이 중 어떤 이름을 사용해야 할까?

여기부터는 단순한 naming convention이 아니라 **메서드가 가진 의미**를 생각해야 한다.

---

# 1. 짧은 이름보다 의미가 있는 이름이 중요하다

처음에는 이름이 짧으면 코드도 깔끔해 보인다.

```
public User get(Long id) {
    return userRepository.findById(id).orElseThrow();
}
```

코드 자체에는 문제가 없다.

하지만 호출하는 쪽에서 보면 조금 애매하다.

```
User user = userService.get(userId);
```

`get`이 무엇을 가져오는 것인지 클래스 문맥을 확인해야 한다.

이번에는 다음 코드를 보자.

```
public User getUser(Long userId) {
    return userRepository.findById(userId).orElseThrow();
}
```

조금 길어졌지만 훨씬 명확하다.

사용자를 가져오는 메서드라는 사실이 바로 보인다.

상황에 따라 실패 방식까지 표현할 수도 있다.

```
public User findUserOrThrow(Long userId) {
    return userRepository.findById(userId)
            .orElseThrow(() -> new UserNotFoundException(userId));
}
```

이름만 보더라도

> 사용자를 찾고, 없으면 예외가 발생하겠구나.

라고 예상할 수 있다.

메서드 이름은 짧다고 좋은 것이 아니다.

**짧으면서 의미까지 명확하다면 가장 좋지만, 의미를 없애면서까지 줄일 필요는 없다.**

---

# 2. `process`, `handle` 같은 넓은 단어를 조심하자

다음과 같은 이름은 개발하면서 정말 쉽게 등장한다.

```
process();
handle();
execute();
manage();
doWork();
```

문제는 범위가 너무 넓다는 것이다.

예를 들어 다음 코드가 있다고 해보자.

```
public void processOrder(OrderRequest request) {
    // 주문 검증
    // 재고 차감
    // 쿠폰 사용
    // 결제 요청
    // 알림 발송
}
```

`processOrder`라는 이름 자체가 틀린 것은 아니다.

하지만 정확히 무엇을 하는지는 알기 어렵다.

오히려 메서드를 나눠보면 의도가 훨씬 잘 보인다.

```
validateOrderRequest(request);
decreaseStock(orderItems);
useCoupon(couponId);
requestPayment(paymentRequest);
sendOrderCompletedNotification(order);
```

이제 코드를 위에서 아래로 읽기만 해도 주문이 처리되는 흐름을 이해할 수 있다.

물론 메서드를 무조건 작게 쪼개라는 의미는 아니다.

지나치게 잘게 나누면 오히려 코드의 흐름을 따라가기 어려워질 수 있다.

다만 메서드 이름을 지으려고 할 때 계속

```
process
handle
execute
```

같은 이름밖에 떠오르지 않는다면 한 번쯤 생각해볼 필요가 있다.

> **이 메서드가 너무 많은 일을 하고 있는 것은 아닐까?**

이름을 짓기 어렵다는 것이 메서드의 책임이 너무 크다는 신호일 수도 있다.

---

# 3. 메서드 이름으로 결과를 어느 정도 예상할 수 있어야 한다

다음 메서드를 보자.

```
checkEmail(email);
```

무엇을 확인한다는 것일까?

가능한 경우가 너무 많다.

- 이메일 형식을 확인한다.
- 이메일 중복을 확인한다.
- 존재하는 이메일인지 확인한다.
- 문제가 있으면 예외를 던진다.
- `boolean`을 반환한다.

조금 더 구체적으로 작성하면 의미가 달라진다.

```
boolean isEmailDuplicated(String email);
```

이름과 반환 타입만 보면 이메일 중복 여부를 반환한다는 사실을 예상할 수 있다.

```
void validateEmailFormat(String email);
```

이번에는 이메일 형식을 검증하고, 문제가 있다면 정상적으로 메서드가 끝나지 않을 가능성을 예상할 수 있다.

중복되지 않았음을 검증하는 메서드라면 다음과 같이 표현할 수도 있다.

```
void validateEmailNotDuplicated(String email);
```

좋은 이름은 단순히 "무언가를 한다"에서 끝나지 않는다.

가능하다면 호출자가 **결과와 동작 방식까지 어느 정도 예상할 수 있게 해준다.**

---

# 4. Boolean 메서드는 질문처럼 읽히게 만들자

`boolean`을 반환하는 메서드는 특히 이름이 중요하다.

다음 코드를 보자.

```
user.active();
order.cancel();
```

이름만 보면 헷갈린다.

`active()`는 사용자를 활성화하는 것일까?

현재 활성 상태인지 반환하는 것일까?

`cancel()` 역시 주문을 취소하는 동작인지, 취소 가능한지를 반환하는 것인지 알기 어렵다.

Boolean 메서드는 보통 다음과 같은 단어를 활용하면 읽기 쉬워진다.

```
is
has
can
should
```

예를 들어 다음과 같다.

```
user.isActive();
user.hasRole(Role.ADMIN);
order.canCancel();
coupon.isExpired();
notification.shouldSend();
```

각각의 느낌도 조금씩 다르다.

### `is`

현재 상태를 묻는다.

```
user.isActive();
coupon.isExpired();
```

### `has`

어떤 값을 가지고 있는지 묻는다.

```
user.hasRole(Role.ADMIN);
```

### `can`

특정 행동이 가능한지를 묻는다.

```
order.canCancel();
```

### `should`

특정 행동을 수행해야 하는지를 판단한다.

```
notification.shouldSend();
```

이렇게 이름을 정하면 실제 코드도 자연스럽게 읽힌다.

```
if (order.canCancel()) {
    order.cancel();
}
```

코드를 문장처럼 읽어보자.

> 주문을 취소할 수 있다면 주문을 취소한다.

의미가 바로 전달된다.

좋은 메서드 이름은 이렇게 **코드 자체가 하나의 문장처럼 읽히게 만들어준다.**

---

# 5. 구현 방법보다 의도를 표현하자

메서드 이름에는 현재 구현 방법을 그대로 넣고 싶은 경우가 있다.

```
selectUserFromDatabase(userId);
```

실제로 DB에서 데이터를 조회하고 있으니 틀린 이름은 아니다.

하지만 Service 계층에서는 조금 다르게 생각할 수 있다.

```
getUserProfile(userId);
```

두 메서드가 내부적으로 동일하게 DB를 조회하더라도 두 번째 이름이 비즈니스적인 목적을 더 잘 보여준다.

다음 이름도 마찬가지다.

```
queryActiveUserAndMapToDto();
```

구현 과정이 이름에 그대로 들어가 있다.

이를 다음과 같이 표현할 수 있다.

```
getActiveUserProfile();
```

호출하는 입장에서 중요한 것은

> DB에서 SELECT를 하고 DTO로 변환한다.

가 아니라

> 활성 사용자의 프로필을 가져온다.

일 가능성이 높다.

내부 구현은 나중에 바뀔 수도 있다.

MySQL을 사용하다가 다른 저장소를 사용할 수도 있고, 조회 결과를 매핑하는 방식도 바뀔 수 있다.

비즈니스 의도를 중심으로 이름을 지으면 구현이 변경되더라도 메서드 이름은 유지될 가능성이 높다.

---

# 6. 도메인에서 사용하는 언어를 사용하자

쇼핑몰에서 주문 상태를 변경한다고 생각해보자.

다음 코드도 기능적으로는 틀리지 않는다.

```
updateStatus(orderId, "CANCELLED");
```

하지만 주문을 취소하는 것은 단순히 DB의 상태값 하나를 바꾸는 작업이 아닐 수도 있다.

실제로는 다음과 같은 작업이 필요할 수 있다.

```
취소 가능한 주문인지 확인
재고 복구
사용한 쿠폰 복구
결제 취소
취소 알림 발송
```

그렇다면 다음 이름이 더 자연스럽다.

```
cancelOrder(orderId);
```

`updateStatus`는 구현 관점의 표현이다.

`cancelOrder`는 비즈니스 행동을 표현한다.

Service 계층에서는 특히 이런 차이가 중요하다.

예를 들어

```
insertOrder();
updatePassword();
updateReviewStatus();
```

보다 상황에 따라 다음과 같은 이름이 더 의미 있을 수 있다.

```
createOrder();
changePassword();
approveReview();
rejectReview();
```

그 밖에도 비즈니스에서는 다음과 같은 단어를 자주 사용할 수 있다.

```
registerUser();
approveApplication();
rejectApplication();
cancelOrder();
publishPost();
archiveProject();
```

단순한 CRUD보다 실제 업무에서 부르는 행동을 코드에 그대로 표현하는 것이다.

---

# 7. `get`, `find`, `load`는 어떻게 구분할까?

이 부분은 프로젝트마다 기준이 다를 수 있다.

따라서 다음 구분을 절대적인 규칙으로 볼 필요는 없다.

나는 보통 이름을 생각할 때 다음 정도의 느낌으로 구분한다.

### `get`

반드시 존재해야 하거나 이미 존재한다고 기대하는 값을 가져오는 느낌이다.

```
getUser();
getOrder();
```

### `find`

대상을 찾는다는 느낌이 강하다.

```
findUser();
findByEmail();
```

특히 Repository에서는 Spring Data JPA의 다음과 같은 형태가 익숙하다.

```
Optional<User> findById(Long id);
Optional<User> findByEmail(String email);
```

### `load`

DB나 외부 저장소 등에서 데이터를 읽어오는 느낌을 표현하고 싶을 때 사용할 수 있다.

```
loadUser();
loadConfiguration();
```

하지만 여기서 중요한 것은

> "`get`은 반드시 이것이고 `find`는 반드시 이것이다."

라는 규칙을 만드는 것이 아니다.

프로젝트에서 이미 정해진 규칙이 있다면 그 규칙을 따르는 것이 더 중요하다.

예를 들어 Service에서는 API의 "회원 정보 조회"라는 의미로 다음과 같이 사용할 수 있다.

```
public UserResponse getUser(Long userId) {
    User user = userRepository.findById(userId)
            .orElseThrow(() -> new UserNotFoundException(userId));

    return UserResponse.from(user);
}
```

반면 내부 helper에서는 실패 동작을 더 구체적으로 표현할 수도 있다.

```
private User findUserOrThrow(Long userId) {
    return userRepository.findById(userId)
            .orElseThrow(() -> new UserNotFoundException(userId));
}
```

결국 가장 중요한 것은 단어 하나의 정답을 찾는 것이 아니다.

> **프로젝트 안에서 같은 단어를 같은 의미로 사용하는 것이 중요하다.**

---

# 8. Service와 Repository는 이름을 바라보는 관점이 다르다

계층에 따라 메서드 이름에서 중요하게 보는 정보도 조금 다를 수 있다.

## Repository

Repository는 데이터 접근 계층이므로 **어떤 조건으로 데이터를 찾는지**가 중요하다.

Spring Data JPA에서는 메서드 이름 자체로 조회 조건을 표현할 수도 있다.

```
Optional<User> findByEmail(String email);

boolean existsByEmail(String email);

List<Order> findByUserIdAndStatus(
        Long userId,
        OrderStatus status
);
```

이름만 보더라도 어느 조건으로 데이터를 조회하는지 알 수 있다.

```
findByEmail
→ email로 사용자를 찾는다.

existsByEmail
→ 해당 email이 존재하는지 확인한다.

findByUserIdAndStatus
→ userId와 status를 기준으로 주문을 찾는다.
```

다만 조건이 많아지면 문제가 생긴다.

```
findByUserIdAndStatusAndCreatedAtBetweenAndDeletedFalseOrderByCreatedAtDesc(...)
```

이 정도로 이름이 길어진다면 무조건 메서드 이름 하나에 모든 조건을 표현하기보다 별도의 Repository 메서드나 JPQL, QueryDSL 등을 활용하는 방법도 고민할 수 있다.

---

## Service

Service에서는 데이터 조회 방식보다는 **어떤 비즈니스 유스케이스를 수행하는지**가 더 중요하다.

예를 들어 다음 이름보다

```
selectUser(userId);
insertOrder(request);
updatePassword(userId, password);
```

다음과 같은 이름이 서비스의 역할을 더 잘 보여줄 수 있다.

```
getUserProfile(userId);
createOrder(request);
changePassword(userId, request);
```

Repository가

> 어떤 데이터를 어떤 조건으로 가져올까?

에 가깝다면,

Service는

> 사용자가 어떤 행동을 수행하려는가?

에 조금 더 가깝다고 생각하면 이해하기 쉽다.

---

# 9. 이름과 실제 동작이 달라서는 안 된다

이름 자체는 좋아 보이지만 실제 동작과 맞지 않는 경우도 있다.

다음 코드를 보자.

```
public User getUser(Long userId) {
    User user = findUser(userId);

    user.updateLastAccessedAt();

    return user;
}
```

메서드 이름은 단순 조회처럼 보인다.

하지만 실제로는 사용자의 마지막 접근 시간까지 변경한다.

호출하는 사람은

```
getUser(userId);
```

만 보고 상태가 변경될 것이라고 예상하기 어렵다.

이런 예상하지 못한 side effect는 코드를 이해하기 어렵게 만든다.

상태 변경 자체가 중요한 동작이라면 이름에 드러낼 수도 있다.

```
getUserAndUpdateLastAccessedAt();
```

혹은 더 나은 구조라면 조회와 상태 변경을 각각의 책임으로 분리하는 것도 생각해볼 수 있다.

중요한 것은 하나다.

> **메서드의 이름과 실제 행동이 일치해야 한다.**

---

# 10. 팀에서 사용하는 용어를 우선하자

혼자 개발하는 프로젝트가 아니라면 내가 생각하기에 가장 예쁜 이름보다 **팀 전체가 같은 말을 사용하는 것**이 더 중요하다.

프로젝트 전체에서 회원을 `Member`라고 부르고 있다고 해보자.

```
MemberService
MemberRepository
MemberResponse
```

그런데 특정 코드에서만 갑자기

```
UserService
findUser()
```

라고 작성하면 같은 개념을 서로 다른 이름으로 표현하게 된다.

코드를 읽는 사람은 고민하게 된다.

> Member와 User가 다른 개념인가?

실제로 같은 개념인데 이름만 다르다면 불필요한 혼란이 생긴다.

따라서 프로젝트에서 이미 사용하고 있는 도메인 용어가 있다면 우선 그 용어를 따르는 편이 좋다.

DB 컬럼명 역시 마찬가지다.

DB에 이런 컬럼이 있다고 해서

```
stat_cd
```

Service 코드까지 다음처럼 작성할 필요는 없다.

```
updateStatCd();
```

업무에서 사용하는 의미에 따라 다음과 같이 표현하는 것이 더 자연스러울 수 있다.

```
changeOrderStatus();
cancelOrder();
approveOrder();
```

---

# 11. 이름이 조금 길어지는 것을 너무 두려워하지 말자

개발을 처음 할 때는 메서드 이름이 길어지면 나쁜 코드처럼 느껴질 수 있다.

그래서 다음과 같이 줄이고 싶어진다.

```
check();
validate();
process();
```

하지만 조금 길더라도 의미가 명확하다면 오히려 읽기 쉬울 수 있다.

```
validateUserCanCancelOrder();
```

분명 짧은 이름은 아니다.

하지만 어떤 검증을 하는지는 명확하다.

물론 이름이 끝없이 길어져도 괜찮다는 의미는 아니다.

이름이 지나치게 길어진다면 이런 질문을 해볼 수 있다.

> 이 메서드가 너무 많은 조건과 책임을 알고 있는 것은 아닐까?

메서드 이름이 길다는 사실 자체보다 **왜 이름이 길어졌는지**를 보는 것이 중요하다.

---

# 12. 이름과 반환 타입도 자연스럽게 맞아야 한다

다음 메서드를 보자.

```
boolean validateEmail(String email);
```

조금 어색하게 느껴질 수 있다.

`validate`라는 이름은 보통 검증을 수행하고 문제가 있을 경우 정상적으로 진행되지 않는 동작을 떠올리게 할 수 있다.

Boolean 값으로 유효 여부를 반환하고 싶다면 다음 이름이 더 자연스럽다.

```
boolean isValidEmail(String email);
```

반대로 검증에 실패했을 때 예외가 발생하는 방식이라면 다음 형태를 사용할 수 있다.

```
void validateEmail(String email);
```

이름만 보는 것이 아니라

```
메서드 이름
파라미터
반환 타입
실패 방식
```

이 함께 자연스러운지도 살펴보면 좋다.

---

# 피하고 싶은 메서드 이름 정리

지금까지 내용을 몇 가지 패턴으로 정리해보자.

## 너무 넓은 이름

```
process();
handle();
manage();
execute();
doWork();
```

특별히 범용적인 작업을 표현하는 것이 아니라면 조금 더 구체적으로 표현할 수 있는지 생각해보자.

---

## 클래스 이름만 반복하는 이름

```
UserService.user();
OrderService.order();
```

이미 클래스가 `UserService`, `OrderService`라는 문맥을 제공하고 있다.

메서드에서는 그 안에서 **어떤 행동을 하는지**를 표현하는 편이 좋다.

```
UserService.getProfile();
OrderService.cancelOrder();
```

---

## 지나친 축약어

```
calcAmt();
updUsr();
regPmt();
```

작성할 때는 빠를 수 있지만 읽는 사람은 해석하는 과정이 필요하다.

조금 길더라도 다음 이름이 훨씬 이해하기 쉽다.

```
calculateAmount();
updateUser();
registerPayment();
```

---

## 실제 동작과 다른 이름

```
getUser();
```

라고 해놓고 내부에서

```
조회
수정
저장
알림 발송
```

까지 일어나면 호출자는 예상하지 못한 동작을 만나게 된다.

이름과 실제 책임이 일치하는지 확인하자.

---

# 그래서 좋은 메서드 이름은 무엇일까?

좋은 메서드 이름에 절대적인 정답이 있는 것은 아니다.

프로젝트의 성격도 다르고, 팀마다 사용하는 용어와 규칙도 다르다.

그래도 메서드 이름을 지을 때 확인할 수 있는 기준은 있다.

### 기본 규칙

```
lowerCamelCase를 사용하는가?
동사 또는 동사구로 작성했는가?
불필요한 축약어를 사용하지 않았는가?
프로젝트에서 사용하는 용어와 일치하는가?
```

그리고 실무에서는 다음 질문이 더 중요하다.

```
이름만 보고 무엇을 하는지 알 수 있는가?

호출 결과를 어느 정도 예상할 수 있는가?

비즈니스에서 사용하는 의미가 드러나는가?

이름과 실제 동작이 일치하는가?

Boolean 메서드는 질문처럼 읽히는가?

호출하는 코드가 자연스러운 문장처럼 읽히는가?
```

예를 들어 다음 코드보다

```
if (check(order)) {
    process(order);
}
```

다음 코드가 훨씬 이해하기 쉽다.

```
if (order.canCancel()) {
    cancelOrder(order);
}
```

코드를 처음 보는 사람도 대략적인 의도를 바로 이해할 수 있다.

---

# 마지막으로

메서드 이름은 단순히 컴파일을 통과하기 위해 붙이는 식별자가 아니다.

메서드 내부를 직접 열어보기 전에

> "이 코드에서는 이런 일을 합니다."

라고 알려주는 가장 작은 설명서에 가깝다.

그래서 메서드 이름을 짓기 어렵다면 무작정 `process`, `handle`, `execute`를 붙이기 전에 한 번만 더 생각해보면 좋다.

```
이 메서드는 정확히 무엇을 하는가?

어떤 결과가 나오는가?

상태를 변경하는가, 단순히 조회하는가?

어떤 비즈니스 행동을 표현하는가?

팀에서 이미 사용하는 용어가 있는가?
```

이 질문에 답하다 보면

```
process();
```

보다

```
createOrder();
cancelPayment();
findActiveUser();
validateEmailNotDuplicated();
```

같은 이름이 자연스럽게 보이기 시작한다.

그리고 이런 작은 이름 하나하나가 쌓이면, 코드는 주석을 계속 읽지 않아도 흐름을 이해할 수 있는 코드에 조금 더 가까워진다.

> **좋은 메서드 이름은 구현을 열어보기 전에 의도를 알려주는 가장 작은 문서다.**
