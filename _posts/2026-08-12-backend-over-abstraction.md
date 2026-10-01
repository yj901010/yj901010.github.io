---
title: "백엔드에서 공통화가 오히려 독이 되는 순간"
date: 2026-08-12 09:58:48 +0900
last_modified_at: 2026-10-01
categories: [Engineering, Design]
tags: [abstraction, refactoring, design]
source_url: "https://velog.io/@joker901010/%EB%B0%B1%EC%97%94%EB%93%9C%EC%97%90%EC%84%9C-%EA%B3%B5%ED%86%B5%ED%99%94%EA%B0%80-%EC%98%A4%ED%9E%88%EB%A0%A4-%EB%8F%85%EC%9D%B4-%EB%90%98%EB%8A%94-%EC%88%9C%EA%B0%84"
---
**중복 코드는 줄이는 것이 좋다. 하지만 모양이 같다는 이유만으로 합치면, 나중에 변경하기 더 어려운 코드가 될 수 있다.**

중요한 것은 코드가 똑같이 생겼는지가 아니다.

> **앞으로도 같은 이유로 함께 바뀔 코드인지**가 더 중요하다.

---

## 중복 코드를 발견하면 바로 합쳐야 할까?

개발을 배우다 보면 `DRY(Don't Repeat Yourself)`라는 말을 자주 듣는다.

쉽게 말하면 이런 의미다.

```
같은 코드를 반복해서 작성하지 말자.
```

그래서 처음에는 이런 생각을 하기 쉽다.

> "같은 코드가 두 번 나오면 메서드로 빼야 하는 거 아닌가?"

실제로 중복 코드는 대표적인 `code smell` 중 하나다.

같은 코드가 여러 곳에 있으면 수정할 때 문제가 생길 수 있다.

예를 들어 같은 검증 코드가 5곳에 있다고 해보자.

```
if (email == null || email.isBlank()) {
    throw new InvalidRequestException("이메일은 필수입니다.");
}
```

검증 방법을 바꾸려면 5곳을 모두 수정해야 한다.

한 군데를 놓치면 기능마다 동작이 달라질 수도 있다.

그래서 이런 코드는 공통화하고 싶어진다.

```
public void validateEmail(String email) {
    if (email == null || email.isBlank()) {
        throw new InvalidRequestException("이메일은 필수입니다.");
    }
}
```

이제 검증 규칙이 바뀌어도 한 곳만 수정하면 된다.

여기까지만 보면 중복은 무조건 없애는 것이 좋아 보인다.

하지만 문제가 하나 있다.

**지금 같아 보이는 두 코드가 앞으로도 같은 규칙을 사용할지는 알 수 없다는 것이다.**

---

## 똑같이 생긴 코드가 항상 같은 코드는 아니다

회원가입과 관리자 초대 기능이 있다고 해보자.

두 기능 모두 이메일을 검사한다.

처음에는 규칙도 똑같다.

```
회원가입
→ 이메일이 비어 있으면 안 된다.

관리자 초대
→ 이메일이 비어 있으면 안 된다.
```

그래서 하나의 메서드로 합쳤다.

```
validateEmail(email);
```

그런데 몇 달 뒤 요구사항이 바뀌었다.

```
회원가입
→ 모든 정상적인 이메일 허용

관리자 초대
→ 회사 이메일만 허용
```

이제 두 기능의 규칙이 달라졌다.

공통 메서드를 유지하려면 이런 식의 코드가 생길 수 있다.

```
validateEmail(email, UserType.ADMIN);
validateEmail(email, UserType.NORMAL);
```

그리고 내부에서는 타입에 따라 분기한다.

```
public void validateEmail(String email, UserType userType) {

    if (email == null || email.isBlank()) {
        throw new InvalidRequestException();
    }

    if (userType == UserType.ADMIN
            && !email.endsWith("@company.com")) {
        throw new InvalidRequestException();
    }
}
```

처음에는 중복을 없애기 위해 만든 메서드였다.

그런데 기능별 요구사항이 들어오기 시작하면서 공통 메서드가 점점 복잡해진다.

이럴 때 중요한 질문이 나온다.

> **이 두 이메일 검증은 정말 같은 책임이었을까?**

처음에는 코드 모양만 같았을 뿐, 실제로는 서로 다른 정책이었을 수도 있다.

---

# 코드가 같은 것과 정책이 같은 것은 다르다

이 부분이 중복 제거에서 가장 중요하다고 생각한다.

예를 들어 다음 두 코드를 보자.

```
if (request.name() == null || request.name().isBlank()) {
    throw new InvalidRequestException("이름은 필수입니다.");
}
```

```
if (request.title() == null || request.title().isBlank()) {
    throw new InvalidRequestException("제목은 필수입니다.");
}
```

코드 모양은 거의 같다.

둘 다 결국 이것을 검사한다.

```
문자열이 비어 있는가?
```

그래서 아래와 같이 아주 작은 기술적 기능으로 공통화하는 것은 자연스럽다.

```
public class ValidationUtils {

    public static void requireNotBlank(
            String value,
            String message
    ) {
        if (value == null || value.isBlank()) {
            throw new InvalidRequestException(message);
        }
    }
}
```

사용하는 쪽도 어렵지 않다.

```
ValidationUtils.requireNotBlank(
        request.name(),
        "이름은 필수입니다."
);

ValidationUtils.requireNotBlank(
        request.title(),
        "제목은 필수입니다."
);
```

이 메서드의 역할은 명확하다.

```
값이 비어 있으면 예외를 던진다.
```

회원, 게시글, 주문 같은 특정 도메인을 알 필요도 없다.

이런 공통화는 비교적 안전하다.

---

## 비즈니스 규칙은 조금 다르다

이번에는 사용자 상태를 검사하는 코드를 생각해보자.

처음에는 모든 기능에서 다음 조건을 사용한다고 해보자.

```
if (!user.isActive()) {
    throw new InvalidUserStatusException();
}
```

사용되는 곳은 세 군데다.

```
게시글 작성
댓글 작성
주문 생성
```

그러면 자연스럽게 이런 메서드를 만들고 싶어진다.

```
validateUserStatus(user);
```

그런데 시간이 지나면서 요구사항이 달라졌다.

```
게시글 작성
→ ACTIVE 사용자만 가능

댓글 작성
→ ACTIVE, LIMITED 사용자 가능

주문 생성
→ ACTIVE 사용자만 가능
   단, 휴면 전환 예정자는 주문 불가능
```

이걸 계속 하나의 공통 메서드에서 처리하면 어떻게 될까?

```
public void validateUserStatus(
        User user,
        ActionType actionType
) {

    if (actionType == ActionType.POST
            && !user.isActive()) {
        throw new InvalidUserStatusException();
    }

    if (actionType == ActionType.COMMENT
            && user.isBlocked()) {
        throw new InvalidUserStatusException();
    }

    if (actionType == ActionType.ORDER
            && (!user.isActive() || user.isDormantSoon())) {
        throw new InvalidUserStatusException();
    }
}
```

중복은 사라졌다.

하지만 코드가 좋아졌다고 하기는 어렵다.

새로운 기능이 추가될 때마다 `ActionType`이 늘어나고 조건문도 계속 증가할 가능성이 높다.

이럴 때는 오히려 각각의 정책을 분리하는 것이 더 이해하기 쉽다.

```
postPolicy.validateWritable(user);
commentPolicy.validateCommentable(user);
orderPolicy.validateOrderable(user);
```

조금 더 많은 코드가 생기더라도 각 코드가 무엇을 의미하는지는 훨씬 명확하다.

---

# 공통화의 목적은 코드 줄 수를 줄이는 것이 아니다

초보 개발자일 때는 리팩터링을 하면서 코드가 줄어들면 왠지 잘한 것처럼 느껴질 때가 있다.

예를 들어 30줄이던 코드가 15줄이 되면 깔끔해진 것처럼 보인다.

하지만 코드의 품질을 단순히 줄 수로 판단하기는 어렵다.

이런 메서드를 생각해보자.

```
public void process(
        String type,
        Long id,
        boolean includeDeleted,
        boolean checkPermission,
        boolean sendNotification
) {
    // 여러 기능의 공통 처리
}
```

호출하는 곳에서는 이렇게 사용한다.

```
process(
    "USER",
    1L,
    false,
    true,
    false
);
```

코드는 짧다.

하지만 처음 보는 사람은 이것만 보고 의미를 이해하기 어렵다.

```
false가 무엇을 의미하지?

첫 번째 true는 뭐지?

USER일 때와 ORDER일 때 동작이 다른가?
```

중복 코드는 줄었지만 **읽는 사람이 이해해야 하는 비용은 오히려 커졌다.**

공통화가 항상 좋은 리팩터링은 아닌 이유다.

---

# 좋은 공통화는 이름만 봐도 역할이 보인다

좋은 공통 코드는 대체로 역할이 작고 명확하다.

예를 들어 다음과 같은 코드다.

```
private boolean isBlank(String value) {
    return value == null || value.isBlank();
}
```

이름만 봐도 무엇을 하는지 알 수 있다.

또는 이런 코드도 있다.

```
private User findUser(Long userId) {
    return userRepository.findById(userId)
            .orElseThrow(
                () -> new UserNotFoundException(userId)
            );
}
```

호출하는 쪽에서는 이렇게 읽힌다.

```
User user = findUser(userId);
```

의도가 명확하다.

반대로 공통화하면서 다음과 같은 이름이 계속 등장한다면 한번 고민해볼 필요가 있다.

```
CommonService

CommonUtils

BaseManager

CommonProcessor

CommonValidator
```

`Common`이라는 이름 자체가 나쁜 것은 아니다.

하지만 "여러 곳에서 쓰니까 일단 여기 넣자"라는 방식으로 코드가 모이다 보면 하나의 클래스가 너무 많은 책임을 가지기 쉽다.

가능하다면 역할을 조금 더 구체적으로 표현하는 것이 좋다.

```
EmailValidator

PageRequestMapper

ErrorResponseFactory

OrderCancelPolicy
```

이름만 보고도 무엇을 담당하는 코드인지 어느 정도 예상할 수 있다.

---

# 백엔드에서 자주 볼 수 있는 과도한 공통화

## 1. 모든 검증을 CommonValidator에 넣기

처음에는 편하다.

```
public class CommonValidator {

    public void validate(
            Object request,
            String type
    ) {
        // ...
    }
}
```

그런데 서비스가 커지면서 검증이 늘어난다.

```
회원가입 검증

게시글 검증

주문 검증

결제 검증

관리자 검증
```

결국 하나의 Validator가 여러 도메인의 규칙을 모두 알아야 한다.

차라리 목적에 맞게 분리하는 것이 이해하기 쉬울 수 있다.

```
SignupValidator

OrderValidator

PaymentValidator
```

혹은 검증이 중요한 도메인 규칙이라면 별도의 Policy나 도메인 객체가 담당할 수도 있다.

---

## 2. 모든 Service의 부모 클래스 만들기

여러 Service에 비슷한 코드가 보이면 이런 클래스를 만들고 싶을 수 있다.

```
public abstract class BaseService {

    protected void validateUser(Long userId) {
        // ...
    }

    protected void checkPermission(Long userId) {
        // ...
    }

    protected void publishEvent(Object event) {
        // ...
    }
}
```

그리고 여러 서비스가 이를 상속한다.

```
public class OrderService extends BaseService {
}
```

처음에는 중복을 쉽게 제거할 수 있다.

하지만 시간이 지나면 문제가 생길 수 있다.

`OrderService`가 어떤 기능을 부모에게서 가져오는지 코드를 바로 보고 알기 어렵고, 서로 관련 없는 Service들이 하나의 부모 클래스에 묶이게 될 수도 있다.

이런 경우에는 상속하기 전에 정말 `BaseService`라는 공통 개념이 존재하는지 생각해볼 필요가 있다.

필요하다면 별도의 Component로 분리해서 명시적으로 의존하게 만드는 방법도 있다.

```
public class OrderService {

    private final PermissionChecker permissionChecker;
    private final EventPublisher eventPublisher;
}
```

의존 관계가 코드에 직접 드러난다는 장점이 있다.

---

## 3. Mapper를 하나로 합치기

DTO 변환 코드가 반복되면 하나로 만들고 싶어진다.

```
public class CommonMapper {

    public Object toResponse(Object entity) {
        // 타입에 따라 변환
    }
}
```

하지만 API마다 원하는 응답이 다를 수 있다.

```
UserSummaryResponse

UserDetailResponse

AdminUserResponse
```

처음에는 모두 `User`를 변환한다는 이유로 합칠 수 있다.

하지만 각각의 API가 보여줘야 하는 정보가 다르다면 변환 목적도 다르다.

이럴 때는 코드가 조금 반복되더라도 목적별 Mapper나 변환 메서드를 두는 것이 더 자연스러울 수 있다.

---

## 4. 공통 응답 객체에 모든 것을 넣기

예를 들어 모든 API를 하나의 응답 객체로 만들었다고 해보자.

```
public record ApiResponse<T>(
    boolean success,
    String code,
    String message,
    T data,
    Object error,
    Object meta,
    String traceId
) {
}
```

처음에는 모든 응답 형식을 통일할 수 있어 편리하다.

그런데 요구사항이 생길 때마다 필드가 추가되면 문제가 된다.

```
이 API에서는 error를 쓰나?

meta는 언제 들어가지?

성공 응답에도 code가 필요한가?

traceId는 모든 응답에 존재하나?
```

공통화의 범위가 지나치게 커지면 오히려 API의 의미를 이해하기 어려워질 수 있다.

공통 응답 형식을 사용하는 것 자체가 문제라는 뜻은 아니다.

중요한 것은 **공통으로 묶은 요소가 정말 모든 API에서 같은 의미와 정책을 가지는지**다.

---

# 그렇다면 언제 공통화하는 것이 좋을까?

나는 다음과 같은 경우라면 공통화를 적극적으로 고려할 수 있다고 생각한다.

## 1. 아주 명확한 기술적 반복

예를 들어 다음과 같은 것들이다.

```
문자열 blank 검사

날짜 형식 변환

공통 페이지 요청 변환

에러 응답 생성

로그 형식 생성
```

특정 도메인에 강하게 묶여 있지 않고 동작도 단순하다면 공통화하기 좋다.

---

## 2. 공통된 도메인 개념이 존재할 때

예를 들어 주문 취소 가능 여부를 여러 곳에서 검사한다고 해보자.

```
if (order.getStatus() == OrderStatus.CREATED
        || order.getStatus() == OrderStatus.PAID) {
    // 주문 취소
}
```

이 조건이 여러 곳에서 반복된다면 단순한 코드 중복이 아니라 하나의 도메인 개념을 발견한 것일 수 있다.

```
public boolean canCancel() {
    return status == OrderStatus.CREATED
            || status == OrderStatus.PAID;
}
```

사용하는 곳도 훨씬 자연스럽다.

```
if (order.canCancel()) {
    order.cancel();
}
```

이것은 단순히 코드를 줄인 것이 아니다.

```
CREATED 또는 PAID 상태인지 검사한다.
```

라는 구현을

```
주문을 취소할 수 있는가?
```

라는 의미로 바꾼 것이다.

이런 리팩터링은 코드의 의도를 더 잘 보여준다.

---

## 3. 여러 곳이 정말 같은 정책을 따라야 할 때

예를 들어 서비스 전체에서 에러 코드 정책을 동일하게 관리한다고 해보자.

```
public record ErrorResponse(
    String code,
    String message
) {
}
```

에러 응답 정책이 바뀌면 모든 API도 같이 바뀌어야 한다.

이런 경우는 **변경 이유 자체가 동일하기 때문에** 공통화할 이유가 충분하다.

---

# 반대로 언제 그냥 두는 것이 좋을까?

## 1. 아직 두 번밖에 나오지 않았을 때

중복을 발견했다고 바로 공통화를 시작할 필요는 없다.

두 코드가 앞으로 어떻게 바뀔지 아직 알 수 없다면 잠시 기다리는 것도 방법이다.

조금 더 반복되는 모습을 보면서

```
정말 같은 개념인지

우연히 지금만 같은 코드인지
```

판단할 수 있기 때문이다.

너무 일찍 공통화하면 잘못된 추상화를 만들 수 있다.

---

## 2. 서로 다른 도메인의 코드일 때

예를 들어 다음 코드는 현재 우연히 조건이 같을 수 있다.

```
회원 상태 검증

주문 상태 검증

결제 상태 검증
```

하지만 각각 바뀌는 이유가 다르다.

회원 정책이 바뀐다고 주문 정책까지 같이 바뀌는 것은 아니다.

코드의 모양만 같다는 이유로 하나로 합치면 도메인의 경계가 흐려질 수 있다.

---

## 3. 공통화하면서 옵션이 계속 늘어날 때

이런 메서드가 만들어지고 있다면 한번 의심해볼 만하다.

```
validate(
    value,
    type,
    required,
    min,
    max,
    pattern,
    message
);
```

공통 메서드가 거의 작은 프레임워크처럼 변하고 있다.

특히 boolean 파라미터가 여러 개 붙기 시작하면 호출하는 코드를 이해하기 어려워질 수 있다.

---

## 4. 타입별 분기가 계속 추가될 때

예를 들어 이런 코드다.

```
if (type == Type.USER) {
    // ...
} else if (type == Type.ORDER) {
    // ...
} else if (type == Type.PAYMENT) {
    // ...
}
```

새로운 기능이 생길 때마다 분기가 추가된다면 실제로는 하나의 공통 기능이 아니라 여러 기능을 억지로 한곳에 모아둔 것일 수 있다.

---

# 중복을 발견했을 때 내가 확인하는 것

중복 코드를 발견하면 바로 메서드를 추출하기보다 몇 가지를 먼저 생각해본다.

```
1. 이 두 코드는 정말 같은 일을 하는가?

2. 앞으로도 같은 이유로 함께 바뀔까?

3. 공통된 이름을 자연스럽게 붙일 수 있을까?

4. 공통화하면 호출하는 코드가 더 읽기 쉬워질까?

5. 공통화하면서 type이나 boolean 파라미터가 늘어나지는 않을까?

6. 공통 코드가 여러 도메인의 규칙을 알아야 하지는 않을까?
```

그중에서도 가장 중요하게 보는 질문은 두 가지다.

> **이 코드는 정말 같은 책임인가?**

그리고

> **앞으로도 같은 이유로 바뀔 것인가?**

둘 다 그렇다면 공통화할 가능성이 높다.

반대로 코드 모양만 같고 변경 이유가 다르다면 조금 중복되어 있더라도 분리해 두는 편이 나을 수 있다.

---

# 중복 제거의 목표는 '짧은 코드'가 아니다

중복을 제거하면 여러 장점이 있다.

```
수정해야 하는 위치가 줄어든다.

같은 정책을 일관되게 유지할 수 있다.

실수로 한 곳만 수정하는 일을 줄일 수 있다.

공통된 개념에 이름을 붙일 수 있다.
```

하지만 잘못된 공통화는 반대의 문제를 만든다.

```
분기문이 계속 늘어난다.

파라미터가 많아진다.

여러 도메인의 규칙이 한곳에 섞인다.

하나의 공통 코드를 수정했는데 여러 기능이 영향을 받는다.

코드는 짧아졌지만 이해하기 어려워진다.
```

결국 중요한 것은 **중복 코드의 개수 자체가 아니다.**

코드를 수정해야 할 때 어디를 봐야 하는지 쉽게 알 수 있고, 변경의 영향 범위를 예측할 수 있는 구조가 더 중요하다.

---

# 마무리

중복 코드는 분명 살펴볼 필요가 있는 `code smell`이다.

하지만 `code smell`은

```
"무조건 제거해야 한다."
```

라는 뜻이라기보다

```
"여기에 개선할 부분이 있는지 한번 확인해보자."
```

라는 신호에 가깝다.

처음 개발을 배울 때는 이런 생각을 하기 쉽다.

```
중복이 있다
→ 나쁜 코드다
→ 공통 메서드로 만든다
```

하지만 실제로는 한 단계가 더 필요하다.

```
중복을 발견한다
        ↓
왜 같은 코드가 생겼는지 확인한다
        ↓
정말 같은 개념인지 확인한다
        ↓
앞으로도 같은 이유로 바뀔지 생각한다
        ↓
그다음 공통화할지 결정한다
```

그래서 나는 중복 제거를 다음과 같이 이해하고 있다.

> **중복을 없애는 목적은 코드를 짧게 만드는 것이 아니라, 같은 이유로 바뀌는 코드를 한곳에서 관리하고 코드의 의도를 더 명확하게 만드는 것이다.**

백엔드 코드를 작성하다 중복을 발견했다면 바로 `CommonUtils`부터 만들기보다 한번 질문해보자.

```
이 코드는 정말 같은 책임인가?

지금 우연히 모양만 같은 것은 아닐까?

공통화하면 코드의 의미가 더 잘 보일까?

아니면 나중에 조건문과 옵션만 늘어나게 될까?
```

중복을 없애는 것보다 더 중요한 것은 **변경하기 쉬운 코드를 만드는 것**이다.

그리고 때로는 조금의 중복을 남겨두는 것이, 잘못된 공통화를 만드는 것보다 훨씬 나은 선택일 수 있다.

---

### 참고할 만한 자료

중복 코드와 리팩터링을 더 공부하고 싶다면 다음 키워드부터 찾아보면 좋다.

- Martin Fowler — Refactoring
- Refactoring.Guru — Duplicate Code
- DRY(Don't Repeat Yourself)
- Code Smell
- Extract Method
- Single Responsibility Principle
