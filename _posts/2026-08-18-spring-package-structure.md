---
title: "Spring Boot 패키지 구조, 어떻게 나눠야 할까? 레이어형과 도메인형 비교"
date: 2026-08-18 09:58:06 +0900
last_modified_at: 2026-10-01
categories: [Architecture, Spring]
tags: [package, layered, domain]
source_url: "https://velog.io/@joker901010/Spring-Boot-%ED%8C%A8%ED%82%A4%EC%A7%80-%EA%B5%AC%EC%A1%B0-%EC%96%B4%EB%96%BB%EA%B2%8C-%EB%82%98%EB%88%A0%EC%95%BC-%ED%95%A0%EA%B9%8C-%EB%A0%88%EC%9D%B4%EC%96%B4%ED%98%95%EA%B3%BC-%EB%8F%84%EB%A9%94%EC%9D%B8%ED%98%95-%EB%B9%84%EA%B5%90"
---
Spring Boot의 패키지 구조에는 하나의 정답이 없다.

작은 프로젝트에서는 `controller`, `service`, `repository`처럼 **레이어별로 나누는 구조**가 단순하고, 기능이 많아질수록 `user`, `order`, `payment`처럼 **도메인별로 묶는 구조**가 코드를 찾기 편한 경우가 많다.

무엇을 선택하든 가장 중요한 것은 **프로젝트 안에서 같은 기준을 계속 유지하는 것**이다.

---

## 패키지는 왜 나눌까?

Spring Boot 프로젝트를 처음 만들면 보통 이런 고민을 하게 된다.

```
UserController는 어디에 둬야 하지?
UserService는 service 패키지에 넣어야 하나?
user라는 패키지를 만들어서 전부 같이 넣어야 하나?
DTO는 dto에 모아야 하나?
```

처음에는 패키지마다 정해진 정답이 있을 것 같지만 그렇지는 않다.

패키지를 나누는 가장 큰 이유는 결국 하나다.

> 필요한 코드를 쉽게 찾고, 관련된 코드를 쉽게 이해하기 위해서다.

파일이 몇 개 없는 프로젝트에서는 어디에 두어도 크게 문제가 없다.

하지만 기능이 점점 많아지면 이야기가 달라진다.

```
회원
주문
결제
상품
쿠폰
리뷰
배송
알림
```

이때 아무 기준 없이 파일을 추가하면 원하는 코드를 찾는 것부터 어려워진다.

그래서 프로젝트가 커질수록 **어떤 기준으로 코드를 분류할 것인지**가 중요해진다.

---

## Spring Boot에서는 어떤 구조를 권장할까?

Spring Boot가 동작하기 위해 반드시 따라야 하는 하나의 패키지 구조는 없다.

다만 Spring Boot 공식 문서에서는 main application class를 프로젝트의 **root package**에 두는 방식을 권장한다.

예를 들어 다음과 같은 구조다.

```
com.example.myapplication
├── MyApplication.java
├── customer
│   ├── Customer.java
│   ├── CustomerController.java
│   ├── CustomerService.java
│   └── CustomerRepository.java
└── order
    ├── Order.java
    ├── OrderController.java
    ├── OrderService.java
    └── OrderRepository.java
```

`@SpringBootApplication`에는 component scan과 관련된 설정이 포함되어 있기 때문에 보통 main application class 아래에 애플리케이션 코드를 배치한다.

Java 프로젝트의 디렉터리도 일반적으로 Maven과 Gradle의 convention을 따른다.

```
src
├── main
│   ├── java
│   └── resources
└── test
    ├── java
    └── resources
```

따라서 가장 기본적인 방향은 다음과 같이 생각할 수 있다.

```
빌드 도구의 기본 디렉터리 구조를 따른다.

Spring Boot main class는 상위 root package에 둔다.

그 아래 패키지는 프로젝트의 성격에 맞게 구성한다.
```

그렇다면 진짜 고민은 그다음이다.

> root package 아래를 어떤 기준으로 나눌 것인가?

![Spring Boot 패키지 구조, 어떻게 나눠야 할까? 레이어형과 도메인형 비교 설명 이미지 1](/assets/images/posts/2026-08-18-spring-package-structure/01.png)

여기서 가장 많이 비교되는 것이 **레이어 기준 구조**와 **도메인 기준 구조**다.

---

# 1. 레이어 기준 패키지 구조

Spring을 처음 배울 때 가장 자주 접하는 구조다.

```
com.example.app
├── controller
├── service
├── repository
├── dto
└── entity
```

예를 들어 회원 기능이 있다면 다음과 같이 배치할 수 있다.

```
com.example.app
├── controller
│   └── UserController.java
├── service
│   └── UserService.java
├── repository
│   └── UserRepository.java
├── dto
│   ├── UserCreateRequest.java
│   └── UserResponse.java
└── entity
    └── User.java
```

기준이 매우 단순하다.

```
Controller → controller
Service    → service
Repository → repository
DTO        → dto
Entity     → entity
```

### 장점

초보자 입장에서는 각 클래스의 역할을 이해하기 쉽다.

`UserService`를 찾고 싶다면 `service` 패키지를 보면 된다.

구조도 단순해서 작은 CRUD 프로젝트에서는 충분히 사용할 수 있다.

### 단점

기능이 많아지면 한 패키지에 파일이 계속 쌓인다.

```
controller
├── UserController.java
├── OrderController.java
├── PaymentController.java
├── ProductController.java
├── CouponController.java
├── ReviewController.java
├── DeliveryController.java
└── ...
```

여기서 회원 기능 전체를 확인하고 싶다고 해보자.

```
controller/UserController.java
service/UserService.java
repository/UserRepository.java
dto/UserCreateRequest.java
dto/UserResponse.java
entity/User.java
```

관련 코드가 여러 패키지에 흩어져 있기 때문에 패키지를 계속 이동해야 한다.

즉, **역할별로 코드를 찾기는 쉽지만 기능 전체를 따라가기에는 불편해질 수 있다.**

![Spring Boot 패키지 구조, 어떻게 나눠야 할까? 레이어형과 도메인형 비교 설명 이미지 2](/assets/images/posts/2026-08-18-spring-package-structure/02.png)

---

# 2. 도메인 기준 패키지 구조

이번에는 클래스의 역할이 아니라 **기능이나 도메인**을 기준으로 코드를 묶는다.

```
com.example.app
├── user
├── order
├── payment
└── product
```

회원 기능을 예로 들면 다음과 같다.

```
com.example.app
└── user
    ├── UserController.java
    ├── UserService.java
    ├── UserRepository.java
    ├── User.java
    └── dto
        ├── UserCreateRequest.java
        └── UserResponse.java
```

주문 기능도 마찬가지다.

```
order
├── OrderController.java
├── OrderService.java
├── OrderRepository.java
├── Order.java
└── dto
    ├── OrderCreateRequest.java
    └── OrderResponse.java
```

회원 기능을 수정해야 한다면 우선 `user` 패키지를 보면 된다.

```
user
├── 요청을 받는 코드
├── 비즈니스 로직
├── 데이터 접근 코드
├── Entity
└── DTO
```

관련 코드가 가까이 모여 있기 때문에 기능의 흐름을 따라가기 편하다.

### 장점

도메인 하나를 중심으로 코드를 찾기 쉽다.

```
user
order
payment
product
```

프로젝트가 커져도 어떤 기능의 코드인지 비교적 쉽게 알 수 있다.

나중에 특정 기능을 별도 모듈이나 서비스로 분리해야 할 때도 경계를 파악하기 편한 경우가 많다.

### 단점

도메인 안의 파일이 많아지면 그 안에서도 다시 정리가 필요하다.

처음에는 이 정도였던 패키지가

```
user
├── UserController.java
├── UserService.java
├── UserRepository.java
└── User.java
```

기능이 많아지면 이렇게 될 수 있다.

```
user
├── controller
├── service
├── repository
├── dto
├── entity
├── exception
└── validator
```

즉, **도메인 기준으로 나눈다고 해서 더 이상 패키지 구조를 고민하지 않아도 되는 것은 아니다.**

![Spring Boot 패키지 구조, 어떻게 나눠야 할까? 레이어형과 도메인형 비교 설명 이미지 3](/assets/images/posts/2026-08-18-spring-package-structure/03.png)

---

# 결국 둘 중 무엇을 써야 할까?

여기서 초보자가 가장 궁금한 질문이 나온다.

> 그래서 레이어 기준과 도메인 기준 중 무엇이 더 좋은가?

프로젝트 상황에 따라 다르다.

아주 작은 CRUD 프로젝트라면 이런 구조도 충분하다.

```
controller
service
repository
dto
entity
```

구조가 단순하고 Spring의 각 레이어를 공부하기에도 편하다.

반대로 기능이 점점 많아지는 프로젝트라면 다음과 같이 도메인을 먼저 나누는 방식이 편할 수 있다.

```
user
order
payment
product
```

그리고 각 도메인이 커졌을 때 내부를 다시 나눈다.

```
user
├── controller
├── service
├── repository
├── dto
└── entity
```

즉 이런 형태다.

```
com.example.app
├── user
│   ├── controller
│   ├── service
│   ├── repository
│   ├── dto
│   └── entity
├── order
│   ├── controller
│   ├── service
│   ├── repository
│   ├── dto
│   └── entity
└── payment
    ├── controller
    ├── service
    ├── repository
    └── dto
```

실제로는 이렇게 **도메인을 먼저 나누고 그 안에서 역할을 다시 나누는 방식**도 많이 사용한다.

중요한 것은 이름이 아니라 기준이다.

---

# 패키지 구조가 어려워지는 진짜 순간

문제는 특정 구조를 선택했을 때보다 **기준이 섞이기 시작했을 때** 많이 발생한다.

예를 들어 프로젝트가 처음에는 이렇게 시작했다고 해보자.

```
controller
service
repository
dto
entity
```

그런데 개발하면서 다음과 같이 변했다.

```
controller
service
repository
dto
entity

user
order
payment
```

이제 새로운 코드를 어디에 넣어야 할지 애매해진다.

```
UserController는 controller인가 user인가?

UserResponse는 dto인가 user/dto인가?

PaymentService는 service인가 payment인가?
```

개발자마다 판단이 달라질 수도 있다.

한 사람은 이렇게 만든다.

```
controller/UserController.java
```

다른 사람은 이렇게 만든다.

```
order/OrderController.java
```

또 다른 사람은 이렇게 만들 수도 있다.

```
payment/controller/PaymentController.java
```

모두 동작은 한다.

하지만 프로젝트 전체를 보면 규칙이 보이지 않는다.

새로운 파일을 만들 때마다

> "이건 어디에 넣어야 하지?"

라는 고민을 반복하게 된다.

그래서 패키지 구조에서 가장 중요한 것은 **완벽한 구조보다 예측 가능한 구조**다.

![Spring Boot 패키지 구조, 어떻게 나눠야 할까? 레이어형과 도메인형 비교 설명 이미지 4](/assets/images/posts/2026-08-18-spring-package-structure/04.png)

---

# 패키지 구조는 마트 진열과 비슷하다

마트에 물건을 정리한다고 생각해보자.

상품 종류별로 진열할 수도 있다.

```
과자
음료
생활용품
냉동식품
```

또 다른 기준으로 정리할 수도 있다.

```
신상품
할인상품
인기상품
행사상품
```

어떤 기준을 선택하든 사람들이 그 규칙을 알고 있다면 원하는 물건을 찾을 수 있다.

문제는 기준이 계속 바뀌는 경우다.

```
1번 진열대 → 상품 종류별
2번 진열대 → 가격별
3번 진열대 → 제조사별
4번 진열대 → 색깔별
```

처음 온 사람은 원하는 상품이 어디에 있는지 예측하기 어렵다.

패키지도 비슷하다.

```
회원은 도메인별로 관리하고

주문은 레이어별로 관리하고

결제 관련 코드는 common에 있고

쿠폰 로직은 util에 있다.
```

이런 구조에서는 파일을 찾는 일이 점점 어려워진다.

좋은 패키지 구조는 결국 **코드의 위치를 어느 정도 예상할 수 있는 구조**다.

---

# common에는 무엇을 넣어야 할까?

프로젝트를 만들다 보면 이런 패키지가 자연스럽게 생긴다.

```
common
util
config
infra
global
shared
```

이런 패키지를 사용하는 것 자체가 잘못된 것은 아니다.

문제는 이름이 넓다 보니 무엇이든 넣기 쉬워진다는 것이다.

## common

`common`에는 여러 기능에서 실제로 함께 사용하는 코드를 두는 것이 자연스럽다.

예를 들면 다음과 같다.

```
common
├── error
├── response
└── pagination
```

또는

```
공통 에러 응답
공통 예외 처리
페이지 응답 객체
여러 도메인에서 사용하는 공통 타입
```

반면 이런 코드는 한 번 생각해볼 필요가 있다.

```
common/UserValidator.java
common/OrderStatusHelper.java
common/PaymentPolicy.java
```

이름에는 `common`이 붙었지만 실제로는 각각 특정 도메인의 규칙일 가능성이 높다.

```
UserValidator → user

OrderStatusHelper → order

PaymentPolicy → payment
```

특정 도메인에서만 사용하는 코드를 단순히 여러 곳에서 호출할 수 있다는 이유로 `common`에 옮기면 도메인 경계가 흐려질 수 있다.

---

# util에는 무엇을 넣어야 할까?

`util`도 자주 커지는 패키지다.

예를 들어 이런 클래스들은 비교적 이해하기 쉽다.

```
DateUtils
StringUtils
MaskingUtils
```

특정 비즈니스에 크게 의존하지 않는 작은 도구들이다.

하지만 이런 코드가 있다고 해보자.

```
OrderUtils.canCancel(order);
```

주문 취소 가능 여부가

```
결제 상태
배송 상태
주문 시간
상품 상태
```

같은 비즈니스 규칙에 따라 결정된다면 단순한 유틸이라고 보기 어렵다.

이런 로직은 경우에 따라

```
Order
OrderPolicy
OrderService
```

등 주문 도메인에 위치하는 편이 의도를 더 잘 드러낼 수 있다.

`util`을 사용할 때는 이런 질문을 해볼 수 있다.

> 이것은 정말 범용적인 도구인가, 아니면 특정 도메인의 규칙인가?

---

# config는 설정 코드에 사용하자

`config`는 비교적 역할이 명확하다.

```
config
├── SecurityConfig.java
├── JacksonConfig.java
├── JpaConfig.java
└── RedisConfig.java
```

Spring이나 외부 라이브러리 설정을 모아두기 좋다.

반대로 비즈니스 규칙을 `config`에 넣는 것은 피하는 것이 좋다.

```
PaymentPolicyConfig
OrderCancelConfig
```

이름만 설정처럼 보일 뿐 실제로 비즈니스 로직이라면 해당 도메인에 두는 편이 더 이해하기 쉽다.

---

# infra는 외부 시스템과의 연결에 자주 사용한다

프로젝트가 조금 커지면 `infra` 또는 `infrastructure`라는 이름도 볼 수 있다.

예를 들면 다음과 같다.

```
infra
├── mail
├── storage
├── payment
└── messaging
```

여기에는 보통 애플리케이션 외부와 연결되는 코드가 들어간다.

```
외부 결제 API
메일 서버
파일 저장소
S3
Kafka
Redis
외부 REST API
```

다만 `infra`의 정확한 역할과 의존 방향은 프로젝트의 아키텍처에 따라 달라질 수 있다.

특히 헥사고날 아키텍처나 클린 아키텍처를 적용하면 단순히 외부 API 코드를 모아두는 것보다 더 명확한 역할을 갖게 된다.

처음 Spring을 공부하는 단계라면 우선

> 외부 시스템과 통신하는 기술적인 코드가 들어갈 수 있는 곳

정도로 이해해도 충분하다.

---

# 처음부터 복잡한 구조를 만들 필요는 없다

인터넷에서 Spring 프로젝트 구조를 찾아보면 이런 구조도 많이 볼 수 있다.

```
domain
application
presentation
infrastructure
```

또는

```
adapter
├── in
└── out

application
├── port
└── service

domain
```

좋은 아키텍처에서 사용되는 구조들이다.

하지만 그렇다고 모든 프로젝트가 처음부터 이런 구조를 가져야 하는 것은 아니다.

게시판 하나를 만드는 작은 프로젝트인데

```
domain
application
adapter
port
infrastructure
presentation
```

까지 모두 만들면 오히려 초보자에게는

> 이 DTO는 application인가 presentation인가?

> Repository 구현체는 adapter인가 infrastructure인가?

같은 새로운 고민이 생긴다.

구조는 복잡할수록 좋은 것이 아니다.

현재 프로젝트가 해결해야 하는 문제에 비해 구조가 지나치게 복잡하면 오히려 개발 비용이 커진다.

---

# 프로젝트 규모에 따라 생각해보기

아주 단순하게 기준을 잡아보면 다음처럼 생각할 수 있다.

### 작은 학습용 CRUD 프로젝트

```
controller
service
repository
dto
entity
```

충분하다.

Spring MVC, JPA, Controller, Service 같은 개념을 익히기에도 좋다.

### 기능이 조금씩 많아지는 프로젝트

```
user
order
payment
product
```

도메인을 기준으로 묶는 방식을 고려할 수 있다.

### 하나의 도메인이 커진 프로젝트

```
user
├── controller
├── service
├── repository
├── dto
└── entity
```

도메인 내부를 다시 역할별로 나누는 것도 가능하다.

### 도메인 규칙과 외부 시스템이 복잡한 프로젝트

이때부터는 필요에 따라

```
domain
application
infrastructure
presentation
```

같은 더 명확한 아키텍처를 검토할 수 있다.

핵심은 **처음부터 가장 복잡한 구조를 선택하는 것이 아니라 프로젝트의 복잡도에 맞게 구조도 함께 발전시키는 것**이다.

![Spring Boot 패키지 구조, 어떻게 나눠야 할까? 레이어형과 도메인형 비교 설명 이미지 5](/assets/images/posts/2026-08-18-spring-package-structure/05.png)

---

# 기존 프로젝트에서는 기존 규칙을 먼저 확인하자

실무에서는 내가 처음부터 패키지 구조를 설계하는 경우보다 이미 만들어진 프로젝트에 들어가는 경우가 더 많다.

예를 들어 기존 코드가 이런 구조라고 해보자.

```
controller
service
repository
dto
entity
```

그런데 내가 도메인 기준 구조가 더 좋다고 생각해서 혼자 다음과 같이 추가한다.

```
controller
service
repository
dto
entity

payment
├── PaymentController
├── PaymentService
└── PaymentRepository
```

내 코드만 보면 깔끔해 보일 수 있다.

하지만 프로젝트 전체의 규칙은 깨졌다.

새로운 구조가 정말 필요하다면 기존 코드까지 어떻게 변경할지 팀과 함께 결정하는 것이 좋다.

그렇지 않다면 우선 기존 프로젝트의 convention을 따르는 편이 안전하다.

실무에서 좋은 구조는 개인이 가장 좋아하는 구조가 아니라 **팀이 함께 이해하고 유지할 수 있는 구조**다.

---

# 테스트 코드도 같은 기준으로 배치하자

패키지 구조는 테스트 코드에도 적용된다.

보통 `src/test/java` 아래에서도 실제 코드와 비슷한 패키지 구조를 사용한다.

예를 들어 실제 코드가

```
src/main/java/com/example/app/user/UserService.java
```

에 있다면 테스트는

```
src/test/java/com/example/app/user/UserServiceTest.java
```

처럼 두는 것이다.

도메인 구조가 다음과 같다면

```
main
└── user
    ├── UserController.java
    ├── UserService.java
    └── UserRepository.java
```

테스트도 비슷하게 구성할 수 있다.

```
test
└── user
    ├── UserControllerTest.java
    ├── UserServiceTest.java
    └── UserRepositoryTest.java
```

이렇게 하면 실제 코드와 테스트 코드의 위치를 서로 쉽게 예상할 수 있다.

---

# 패키지를 옮길 때 조심할 점

Java에서 패키지는 단순한 폴더 이름만은 아니다.

클래스를 다른 패키지로 옮기면 다음과 같은 부분이 영향을 받을 수 있다.

```
package 선언
import
접근 제한자
component scan
JPA Entity scan
설정 클래스의 scan 범위
테스트 패키지
```

특히 Spring Boot main application class보다 상위나 전혀 다른 패키지로 Bean을 옮기면 component scan 대상에서 빠질 수 있다.

예를 들어 main class가

```
com.example.app.Application
```

에 있다면 일반적으로

```
com.example.app.user
com.example.app.order
com.example.app.payment
```

는 하위 패키지이므로 component scan 대상이 된다.

하지만 전혀 다른 곳에 클래스를 두면 별도 설정이 필요할 수 있다.

따라서 대규모 패키지 이동은 단순한 폴더 정리라고 생각하지 않는 것이 좋다.

---

# 팀의 패키지 규칙을 간단하게 적어두자

패키지 구조를 정했다면 README나 개발 문서에 간단한 규칙을 남겨두는 것도 좋다.

예를 들어 다음 정도면 충분하다.

```
패키지 규칙

- 기능 코드는 도메인 기준으로 배치한다.
- DTO는 각 도메인의 dto 패키지에 둔다.
- 도메인 전용 예외는 해당 도메인에 둔다.
- 공통 에러 응답은 common/error에 둔다.
- 외부 API 연동 코드는 infra에 둔다.
- Spring 설정 클래스는 config에 둔다.
- 테스트 코드는 main 코드와 동일한 패키지 구조를 따른다.
```

몇 줄 안 되는 규칙이지만 새로운 기능을 추가할 때

> 이 클래스는 어디에 두지?

라는 고민을 크게 줄여준다.

코드 리뷰에서도 패키지 위치를 가지고 반복해서 논의할 필요가 줄어든다.

---

# 패키지 구조를 정할 때 확인할 질문

패키지 구조가 고민된다면 다음 질문을 해보자.

```
프로젝트 규모가 얼마나 큰가?

단순 CRUD 중심인가?

도메인별 기능이 많이 나뉘는가?

관련 코드를 한곳에서 보는 것이 중요한가?

새로운 파일의 위치를 쉽게 결정할 수 있는가?

공통 코드와 도메인 코드를 구분할 수 있는가?

기존 프로젝트가 사용하고 있는 규칙은 무엇인가?

나중에 모듈을 분리할 가능성이 있는가?

테스트 코드도 쉽게 찾을 수 있는가?
```

그리고 개인적으로 가장 이해하기 쉬운 질문은 이것이라고 생각한다.

> 처음 프로젝트에 들어온 개발자가 회원 기능을 수정하려면 어디부터 보면 될까?

이 질문에 쉽게 답할 수 있다면 패키지 구조가 어느 정도 역할을 하고 있는 것이다.

---

# 정리

Spring Boot의 패키지 구조에는 하나의 정답이 없다.

레이어를 기준으로 나눌 수도 있다.

```
controller
service
repository
dto
entity
```

기능이나 도메인을 기준으로 나눌 수도 있다.

```
user
order
payment
product
```

또는 도메인을 먼저 나눈 뒤 그 안에서 다시 역할을 나눌 수도 있다.

```
user
├── controller
├── service
├── repository
├── dto
└── entity
```

작은 프로젝트라면 단순한 레이어 구조만으로도 충분하다.

기능이 많아지면 관련 코드를 가까이 모을 수 있는 도메인 기준 구조가 더 편해질 수 있다.

그리고 프로젝트의 복잡도가 더 높아진다면 그때 애플리케이션, 도메인, 인프라 등의 경계를 더 명확하게 나누는 아키텍처를 고민하면 된다.

무엇보다 피해야 할 것은 기준 없이 구조가 섞이는 것이다.

```
회원은 도메인 기준
주문은 레이어 기준
결제는 common
쿠폰은 util
```

이렇게 되면 새로운 코드를 추가할 때마다 위치를 다시 고민해야 한다.

반대로 기준이 명확하다면 새로운 개발자가 들어와도 어느 정도 코드의 위치를 예상할 수 있다.

결국 좋은 패키지 구조는 화려한 이름을 가진 구조가 아니다.

> **필요한 코드를 어디에서 찾아야 할지 예측할 수 있는 구조다.**

패키지를 나눌 때는 "어떤 구조가 정답인가?"보다 다음 질문을 먼저 해보면 좋다.

```
이 코드는 왜 이 패키지에 있는가?

비슷한 역할의 코드도 같은 기준으로 배치되어 있는가?

새로운 기능을 추가할 때 위치를 쉽게 결정할 수 있는가?

처음 온 개발자도 구조를 예상할 수 있는가?
```

이 질문에 일관되게 답할 수 있다면, 그 프로젝트에는 이미 꽤 괜찮은 패키지 구조가 만들어져 있다고 볼 수 있다.
