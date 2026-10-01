---
title: "주석보다 코드가 먼저다: 초보자를 위한 좋은 주석 작성법"
date: 2026-08-16 20:52:02 +0900
last_modified_at: 2026-10-01
categories: [Engineering, Code Quality]
tags: [comment, clean-code]
source_url: "https://velog.io/@joker901010/%EC%A3%BC%EC%84%9D%EB%B3%B4%EB%8B%A4-%EC%BD%94%EB%93%9C%EA%B0%80-%EB%A8%BC%EC%A0%80%EB%8B%A4-%EC%B4%88%EB%B3%B4%EC%9E%90%EB%A5%BC-%EC%9C%84%ED%95%9C-%EC%A2%8B%EC%9D%80-%EC%A3%BC%EC%84%9D-%EC%9E%91%EC%84%B1%EB%B2%95"
---
주석을 쓰지 말라는 뜻이 아니다. 코드가 설명할 수 있는 **‘무엇을 하는지’**는 코드로 드러내고, 코드만으로 알기 어려운 **‘왜 그렇게 했는지’**를 주석으로 보완하자는 뜻이다.

## 주석이 많으면 더 친절한 코드일까?

처음 코드를 배울 때는 주석을 많이 달수록 친절한 코드라고 생각하기 쉽다.

```
// 사용자 ID로 사용자를 조회한다.
User user = userRepository.findById(userId)
        .orElseThrow(() -> new UserNotFoundException(userId));

// 사용자의 이름을 가져온다.
String name = user.getName();

// 사용자 응답을 생성한다.
return new UserResponse(user.getId(), name);
```

주석이 세 줄이나 있으니 이해하기 쉬워 보인다. 하지만 주석을 하나씩 지우고 코드를 다시 읽어보자.

- `findById`는 ID로 사용자를 찾는다는 뜻이다.
- `getName`은 이름을 가져온다는 뜻이다.
- `new UserResponse`는 사용자 응답을 만든다는 뜻이다.

주석이 코드에 이미 적힌 내용을 그대로 반복하고 있다. 이런 주석은 새로운 정보를 주지 않는다.

더 큰 문제는 코드와 주석이 서로 다르게 바뀔 수 있다는 점이다.

```
// 사용자 ID로 사용자를 조회한다.
User user = userRepository.findByEmail(email)
        .orElseThrow(() -> new UserNotFoundException(email));
```

코드는 이메일로 사용자를 찾는데 주석은 여전히 ID로 찾는다고 말한다. 이제 주석은 이해를 돕는 설명이 아니라 잘못된 안내가 되었다.

**텍스트**

## “주석보다 코드가 먼저”라는 말의 뜻

이 말은 **주석을 모두 지우자**는 뜻이 아니다.

주석이 필요하다고 느꼈을 때 곧바로 설명을 덧붙이기 전에, 변수명이나 메서드명, 조건식과 코드 구조를 조금 더 명확하게 만들 수 있는지 먼저 살펴보자는 뜻이다.

예를 들어 다음 코드를 보자.

```
// 주문을 처리한다.
process(orderId);
```

`process`만으로는 주문을 생성하는지, 취소하는지, 결제하는지 알기 어렵다. 그래서 주석이 필요해졌다.

메서드 이름을 구체적으로 바꾸면 주석 없이도 의도가 보인다.

```
cancelOrderAndRestoreStock(orderId);
```

주석을 없애기 위해 억지로 코드를 바꾸는 것이 아니다. 코드가 자신의 역할을 더 정확하게 말하도록 고치는 것이다.

코드는 잘 정리된 건물과 비슷하다. 문마다 `주문 취소`, `결제 승인`, `비밀번호 변경`처럼 정확한 이름이 붙어 있다면 안내판이 많이 필요하지 않다. 반대로 모든 문에 `처리`, `작업`, `실행`이라고만 적혀 있다면 곳곳에 설명을 붙여야 한다.

## 공식 문서에서는 어떻게 설명할까?

Oracle의 *Java Code Conventions*는 주석이 코드의 개요와 코드만으로 쉽게 알 수 없는 추가 정보를 제공해야 한다고 설명한다. 또한 코드에 이미 명확하게 나타난 정보를 주석으로 되풀이하지 말고, 주석이 자주 필요하다면 코드를 더 명확하게 다시 작성할 수 있는지 생각해보라고 권한다.

다만 이 문서는 1999년에 마지막으로 개정된 보관 문서이며 현재 적극적으로 유지되는 표준은 아니다. 그 점을 감안해 읽어야 하지만, **코드와 중복되는 주석은 낡기 쉽다**는 원칙은 지금도 충분히 참고할 만하다.

현재 Javadoc 명세에서는 문서화 주석이 모듈, 패키지, 클래스, 인터페이스, 생성자, 메서드, 필드 등의 선언 바로 앞에 놓일 때 인식된다고 설명한다. Javadoc은 단순한 구현 설명보다, 다른 개발자가 알아야 할 API의 의미와 사용 계약을 문서화할 때 유용하다.

참고 문서:

- [Oracle Java Code Conventions - Comments](https://www.oracle.com/java/technologies/javase/codeconventions-comments.html)
- [Javadoc Documentation Comment Specification](https://docs.oracle.com/en/java/javase/25/docs/specs/javadoc/doc-comment-spec.html)

## 코드가 ‘무엇’을 말하고, 주석이 ‘왜’를 말한다

좋은 기준 하나만 기억한다면 이것으로 충분하다.

> 코드에는 무엇을 하는지가 드러나야 하고, 주석은 왜 그렇게 해야 하는지를 보완해야 한다.

다음 주석은 코드와 같은 말을 반복한다.

```
// 주문을 취소한다.
order.cancel();
```

반면 다음 주석은 코드만으로 알기 어려운 운영 배경을 알려준다.

```
// 결제사가 같은 콜백을 다시 보낼 수 있으므로 이미 처리된 요청도 성공으로 응답한다.
if (payment.isAlreadyCompleted()) {
    return PaymentCallbackResponse.success();
}
```

코드를 보면 이미 완료된 결제에 성공 응답을 보낸다는 사실은 알 수 있다. 하지만 **왜 실패가 아니라 성공으로 응답하는지**는 알기 어렵다. 주석이 그 빈칸을 채워준다.

물론 ‘무엇’과 ‘왜’를 기계적으로 나눌 수 있는 것은 아니다. 중요한 것은 주석이 코드를 읽는 사람에게 새로운 정보를 주는지 확인하는 것이다.

## 작은 리팩터링으로 주석 줄이기

복잡한 기술을 사용하지 않아도 이름과 구조를 조금만 고치면 많은 주석이 필요 없어질 수 있다.

### 1. 변수 이름에 의미 담기

```
// 탈퇴하지 않은 활성 사용자 수
int count = userRepository.countByDeletedFalseAndActiveTrue();
```

`count`가 무엇의 개수인지 이름만으로는 알 수 없다.

```
int activeUserCount = userRepository.countByDeletedFalseAndActiveTrue();
```

이제 변수를 사용하는 곳에서도 의미를 바로 알 수 있다.

### 2. 메서드 이름을 구체적으로 짓기

```
// 비밀번호를 암호화하고 사용자를 저장한다.
save(request);
```

메서드의 책임을 이름에 드러낼 수 있다.

```
createUserWithEncodedPassword(request);
```

이름이 지나치게 길어진다면 메서드가 여러 책임을 한꺼번에 맡고 있다는 신호일 수도 있다. 그럴 때는 책임을 나누는 방법도 생각해볼 수 있다.

```
String encodedPassword = passwordEncoder.encode(request.password());
User user = User.create(request.email(), encodedPassword);

userRepository.save(user);
```

### 3. 복잡한 조건에 이름 붙이기

```
if (user.getRole() == Role.ADMIN && user.isActive() && !user.isLocked()) {
    showAdminMenu();
}
```

조건을 하나씩 해석해야 전체 의미를 알 수 있다. 의미 있는 메서드로 추출하면 문장처럼 읽힌다.

```
if (user.canAccessAdminFeature()) {
    showAdminMenu();
}
```

```
public boolean canAccessAdminFeature() {
    return role == Role.ADMIN && active && !locked;
}
```

### 4. 계산 결과에 이름 붙이기

```
if (order.getTotalPrice() - order.getDiscountPrice() >= 50_000) {
    applyFreeShipping();
}
```

계산식 자체보다 그 결과가 무엇을 의미하는지 알려주는 편이 읽기 쉽다.

```
int finalPrice = order.getTotalPrice() - order.getDiscountPrice();

if (finalPrice >= 50_000) {
    applyFreeShipping();
}
```

무료 배송 기준이 여러 곳에서 사용되는 도메인 규칙이라면 메서드로 옮길 수도 있다.

```
if (order.isEligibleForFreeShipping()) {
    applyFreeShipping();
}
```

모든 계산식을 반드시 메서드로 만들 필요는 없다. 한 번만 사용되는 간단한 계산이라면 의미 있는 중간 변수만으로도 충분하다.

### 5. 긴 흐름을 작은 단계로 나누기

한 메서드가 너무 많은 일을 하면 주석으로 구역을 나누고 싶어진다.

```
public void createOrder(OrderRequest request) {
    // 주문 검증
    // 재고 차감
    // 쿠폰 사용
    // 결제 요청
    // 알림 발송
}
```

각 단계의 이름이 보이도록 나누면 전체 흐름을 빠르게 파악할 수 있다.

```
public void createOrder(OrderRequest request) {
    validateOrder(request);
    decreaseStock(request.items());
    applyCoupon(request.couponId());
    requestPayment(request.payment());
    sendOrderCreatedNotification(request.userId());
}
```

다만 줄 수를 줄이기 위해 무조건 메서드를 쪼개는 것이 목적은 아니다. 이름을 붙였을 때 의미 있는 단계가 되는지를 기준으로 판단하는 편이 좋다.

## 도메인 메서드로 의도 드러내기

다음 코드는 주문 상태를 직접 확인하고 변경한다.

```
// 취소할 수 있는 주문인지 확인한다.
if (order.getStatus() == OrderStatus.PAID
        || order.getStatus() == OrderStatus.CREATED) {
    // 주문 상태를 취소로 변경한다.
    order.setStatus(OrderStatus.CANCELLED);

    // 재고를 복구한다.
    stock.increase(order.getQuantity());
}
```

주석 덕분에 동작은 이해할 수 있지만, 주문 취소 규칙과 상태 변경 방법이 외부에 그대로 드러나 있다. 주문 객체가 자신의 취소 규칙을 책임지게 만들면 호출하는 코드는 더 단순해진다.

```
order.cancel();
stock.restore(order.getQuantity());
```

`Order` 내부에서는 취소할 수 있는 상태인지 검사하고 상태를 변경한다.

```
public void cancel() {
    if (!isCancellableStatus()) {
        throw new OrderCannotCancelException(id);
    }

    this.status = OrderStatus.CANCELLED;
}

private boolean isCancellableStatus() {
    return status == OrderStatus.PAID
            || status == OrderStatus.CREATED;
}
```

이제 호출하는 쪽은 다음과 같이 읽힌다.

> 주문을 취소하고 재고를 복구한다.

여기에도 정책의 배경이 중요하다면 주석을 남길 수 있다.

```
private boolean isCancellableStatus() {
    // 고객센터 정책상 배송 준비가 시작되기 전까지만 즉시 취소할 수 있다.
    return status == OrderStatus.PAID
            || status == OrderStatus.CREATED;
}
```

코드는 **어떤 상태에서 취소할 수 있는지** 보여주고, 주석은 **왜 그 상태까지만 허용하는지** 알려준다.

## 좋은 주석이 필요한 순간

주석 없이 모든 것을 설명하려고 하면 오히려 코드가 부자연스러워질 수 있다. 다음과 같은 정보는 코드만으로 충분히 드러나지 않는 경우가 많다.

### 1. 비즈니스 정책의 배경

```
// 무료 체험 사용자는 결제 수단을 등록한 뒤에만 유료 기능을 사용할 수 있다.
if (user.isTrial() && !user.hasPaymentMethod()) {
    throw new PaymentMethodRequiredException();
}
```

### 2. 외부 시스템의 제약

```
// 배송사 API가 초 단위 값만 받으므로 밀리초가 아닌 epoch second를 전송한다.
long requestedAt = Instant.now().getEpochSecond();
```

### 3. 레거시 호환성

```
// v1 앱이 null 목록을 처리하지 못하므로 데이터가 없어도 빈 배열을 반환한다.
return List.of();
```

### 4. 성능을 위한 의도적인 선택

```
// 이 화면에는 개수만 필요하므로 주문 Entity 전체를 조회하지 않는다.
int orderCount = orderRepository.countByUserId(userId);
```

### 5. 이상해 보이지만 필요한 코드

```
// 중복 콜백에 실패를 반환하면 결제사가 계속 재시도하므로 성공으로 응답한다.
if (callbackHistory.exists(callbackId)) {
    return CallbackResponse.success();
}
```

이런 주석의 공통점은 코드의 동작을 번역하는 데 그치지 않고 정책, 제약, 호환성, 성능과 같은 배경을 알려준다는 것이다.

가능하다면 임시 대응 주석에는 **언제 제거할 수 있는지**도 함께 적어두는 것이 좋다.

```
// v1 앱 지원 종료 후 빈 문자열 호환 처리를 제거한다. TODO #123
if (nickname != null && nickname.isBlank()) {
    nickname = null;
}
```

## Javadoc은 일반 주석과 무엇이 다를까?

일반 구현 주석은 코드 내부의 특정 선택이나 배경을 설명한다.

```
// 외부 결제사가 같은 요청을 여러 번 보낼 수 있어 중복 요청을 허용한다.
```

Javadoc은 클래스나 메서드를 사용하는 개발자에게 API의 사용 방법과 계약을 알려주는 문서다.

```
/**
 * 주문을 취소하고 취소된 상품의 재고를 복구한다.
 *
 * @param orderId 취소할 주문 ID
 * @throws OrderNotFoundException 주문을 찾을 수 없는 경우
 * @throws OrderCannotCancelException 주문이 취소 가능한 상태가 아닌 경우
 */
public void cancelOrder(long orderId) {
    // ...
}
```

Javadoc은 다음과 같은 경우 특히 도움이 된다.

- 다른 모듈이나 팀에서 사용하는 공개 API
- 여러 프로젝트에서 함께 쓰는 공통 라이브러리
- 예외 발생 조건이나 부수 효과를 알아야 하는 메서드
- 단위, 범위, `null` 허용 여부처럼 타입과 이름만으로 부족한 계약
- 복잡한 도메인 정책을 제공하는 클래스나 메서드

반대로 모든 `private` 메서드에 기계적으로 Javadoc을 붙일 필요는 없다.

```
/**
 * 사용자를 반환한다.
 *
 * @return 사용자
 */
private User getUser() {
    return user;
}
```

이 Javadoc은 코드와 같은 내용을 반복한다. 문서가 추가로 알려주는 계약이나 주의점도 없다.

## 실무에서 주석을 남길 때 조심할 점

### 주석도 코드와 함께 수정한다

```
// 최신 게시글을 최대 10개 조회한다.
List<Post> posts = postRepository.findTop20ByOrderByCreatedAtDesc();
```

코드는 20개를 조회하지만 주석은 10개라고 말한다. 코드 변경으로 설명이 달라졌다면 주석도 같은 작업에서 수정해야 한다.

### 나쁜 이름을 주석으로 덮지 않는다

```
// 활성 사용자만 찾는다.
findUser();
```

이 경우에는 주석보다 메서드 이름을 먼저 고치는 편이 낫다.

```
findActiveUsers();
```

반환값이 한 명인지 여러 명인지에 따라 `findActiveUser` 또는 `findActiveUsers`처럼 이름을 맞추면 더 명확하다.

### TODO에는 이유와 완료 조건을 남긴다

```
// TODO 나중에 수정
```

무엇을 왜 수정해야 하는지 알 수 없으므로 시간이 지나면 의미를 잃는다.

```
// TODO #123 결제 API v2 전환이 끝나면 legacyPaymentClient를 제거한다.
```

이슈 번호와 제거 조건이 있으면 나중에 추적하기 쉽다.

### 민감한 정보는 주석에도 적지 않는다

비밀번호, API 키, 접근 토큰, 개인정보와 같은 값은 주석에도 남기면 안 된다. 주석 역시 코드와 함께 저장소, 코드 리뷰 기록, 백업 등에 남을 수 있다.

## 주석을 달기 전 확인할 다섯 가지

주석을 쓰고 싶어질 때 다음 질문을 차례로 확인해보자.

1. 이 주석은 코드와 같은 말을 반복하고 있지 않은가?
2. 변수명이나 메서드명을 바꾸면 주석 없이도 이해할 수 있는가?
3. 복잡한 조건이나 계산에 의미 있는 이름을 붙일 수 있는가?
4. 이 주석은 동작이 아니라 이유와 배경을 알려주는가?
5. 코드가 바뀔 때 함께 관리할 수 있고, 외부에 노출되어도 안전한 내용인가?

다섯 질문에 모두 완벽하게 답해야만 주석을 쓸 수 있다는 뜻은 아니다. 주석이 정말 필요한 정보인지 잠시 생각해보는 습관을 만드는 것이 목적이다.

## 마무리

주석은 많다고 좋은 것도 아니고, 적다고 좋은 것도 아니다.

코드가 스스로 설명할 수 있는 부분은 이름과 구조로 표현하고, 코드만으로 알기 어려운 정책과 제약, 의도는 주석으로 보완해야 한다.

정리하면 역할은 다음과 같다.

```
코드: 지금 무엇을 하는지 보여준다.
주석: 왜 이런 선택을 했는지 설명한다.
Javadoc: 이 API를 어떻게 사용해야 하는지 알려준다.
```

그래서 나는 “주석보다 코드가 먼저”라는 말을 이렇게 이해한다.

> 주석을 줄이는 것이 목적이 아니다. 코드와 주석이 서로 다른 역할을 제대로 하게 만드는 것이 목적이다.
