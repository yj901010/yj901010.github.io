---
title: "Spring API 응답, 꼭 공통 형식으로 감싸야 할까?"
date: 2026-08-11 09:57:41 +0900
last_modified_at: 2026-10-01
categories: [Backend, Spring]
tags: [api, response, spring]
source_url: "https://velog.io/@joker901010/Spring-API-%EC%9D%91%EB%8B%B5-%EA%BC%AD-%EA%B3%B5%ED%86%B5-%ED%98%95%EC%8B%9D%EC%9C%BC%EB%A1%9C-%EA%B0%90%EC%8B%B8%EC%95%BC-%ED%95%A0%EA%B9%8C"
---
Spring으로 API를 만들다 보면 `ApiResponse<T>` 같은 **공통 응답 형식**을 자주 보게 된다.

공통 응답 형식은 여러 API의 응답 구조를 일정하게 만들어주는 장점이 있지만, 모든 API에 무조건 적용해야 하는 것은 아니다.

핵심은 이것이다.

> 공통 응답 형식은 API를 일관되게 만들기 위한 도구이지, 모든 응답을 반드시 감싸야 하는 규칙은 아니다.

---

## API 응답에는 무엇이 들어갈까?

Spring에서 API를 처음 만들다 보면 JSON 응답만 생각하기 쉽다.

예를 들어 회원 조회 API가 있다고 해보자.

```
{
  "id": 1,
  "name": "junior"
}
```

하지만 실제 HTTP 응답은 JSON만 있는 것이 아니다.

크게 보면 다음과 같이 구성된다.

```
HTTP Status
Headers
Body
```

예를 들어 정상적으로 회원을 조회했다면 다음과 같은 응답이 만들어질 수 있다.

```
HTTP/1.1 200 OK
Content-Type: application/json
```

```
{
  "id": 1,
  "name": "junior"
}
```

Spring에서는 `ResponseEntity`를 사용하면 Body뿐 아니라 HTTP Status와 Header까지 함께 제어할 수 있다.

```
@GetMapping("/users/{id}")
public ResponseEntity<UserResponse> getUser(@PathVariable Long id) {
    UserResponse response = userService.getUser(id);

    return ResponseEntity.ok(response);
}
```

여기서 중요한 것은 하나다.

**API 응답의 의미를 JSON Body만으로 표현할 필요는 없다는 것이다.**

HTTP 자체에서도 이미 `200`, `404`, `500` 같은 상태 코드를 통해 요청의 결과를 표현하고 있다.

---

# 그런데 공통 응답 형식은 왜 사용할까?

Spring 프로젝트를 보다 보면 이런 형태를 한 번쯤 보게 된다.

```
{
  "success": true,
  "data": {
    "id": 1,
    "name": "junior"
  },
  "message": "성공"
}
```

실패하면 이런 식이다.

```
{
  "success": false,
  "data": null,
  "message": "사용자를 찾을 수 없습니다."
}
```

이처럼 여러 API가 동일한 구조로 응답하도록 만드는 것을 보통 **공통 응답 형식** 또는 **Response Wrapper**라고 부른다.

Spring에서는 다음과 같이 만들 수 있다.

```
public record ApiResponse<T>(
    String code,
    String message,
    T data
) {

    public static <T> ApiResponse<T> success(T data) {
        return new ApiResponse<>(
            "SUCCESS",
            "요청이 성공했습니다.",
            data
        );
    }
}
```

Controller에서는 다음과 같이 사용한다.

```
@GetMapping("/users/{id}")
public ApiResponse<UserResponse> getUser(@PathVariable Long id) {
    UserResponse response = userService.getUser(id);

    return ApiResponse.success(response);
}
```

그러면 응답은 항상 비슷한 형태가 된다.

```
{
  "code": "SUCCESS",
  "message": "요청이 성공했습니다.",
  "data": {
    "id": 1,
    "name": "junior"
  }
}
```

이것이 공통 응답 형식을 사용하는 가장 기본적인 이유다.

**API마다 제각각인 응답을 일정한 규칙으로 맞추는 것이다.**

---

# 회사 문서 양식으로 생각해보자

공통 응답 형식이 아직 어렵다면 회사의 문서 양식을 생각하면 이해하기 쉽다.

회사에서 보고서를 작성할 때 다음과 같은 양식이 있다고 해보자.

```
제목
작성자
요약
본문
첨부
```

모든 보고서가 같은 양식을 사용하면 읽는 사람 입장에서는 편하다.

어디에 제목이 있고, 어디에 요약이 있으며, 어디부터 본문인지 쉽게 알 수 있기 때문이다.

API 응답도 비슷하다.

```
{
  "code": "SUCCESS",
  "message": "요청이 성공했습니다.",
  "data": {
    "id": 1,
    "name": "junior"
  }
}
```

클라이언트는 항상 이렇게 생각할 수 있다.

```
code    → 요청 결과 코드
message → 사용자에게 보여줄 메시지
data    → 실제 데이터
```

응답 구조를 예측하기 쉬워진다.

하지만 문제가 하나 있다.

회사에서 사용하는 모든 문서가 보고서는 아니다.

```
보고서
계약서
영수증
회의록
공지사항
```

각각 목적이 다르기 때문에 모든 문서를 하나의 양식으로 만드는 것은 오히려 불편할 수 있다.

API도 마찬가지다.

```
일반 JSON 응답
목록 조회
페이징
파일 다운로드
204 No Content
Streaming
Error Response
```

응답의 목적이 서로 다르기 때문에 하나의 공통 형식을 무조건 강제하면 오히려 어색해질 수 있다.

---

# 공통 응답 형식이 좋은 점

그렇다면 공통 응답 형식은 언제 도움이 될까?

## 1. 프론트엔드에서 응답을 일정하게 처리할 수 있다

API마다 응답 구조가 다르면 클라이언트 코드도 달라질 수 있다.

예를 들어 어떤 API는 이렇게 내려오고,

```
{
  "user": {
    "id": 1
  }
}
```

다른 API는 이렇게 내려오고,

```
{
  "result": {
    "title": "게시글"
  }
}
```

또 다른 API는 이렇게 내려온다면,

```
{
  "data": []
}
```

클라이언트는 API마다 다른 처리를 해야 한다.

반대로 공통 형식을 사용하면 다음처럼 규칙을 맞출 수 있다.

```
{
  "code": "SUCCESS",
  "data": {
    "id": 1
  }
}
```

```
{
  "code": "SUCCESS",
  "data": {
    "title": "게시글"
  }
}
```

클라이언트는 항상 `data`에서 실제 데이터를 찾으면 된다.

```
const data = response.data;
```

프로젝트 규모가 커질수록 이런 일관성이 협업에 도움이 될 수 있다.

---

## 2. 서비스에서 사용하는 에러 코드를 만들 수 있다

HTTP Status만으로 모든 에러 상황을 자세하게 표현하기는 어렵다.

예를 들어 다음 두 상황을 생각해보자.

```
회원이 존재하지 않음
게시글이 존재하지 않음
```

둘 다 HTTP Status는 `404 Not Found`가 될 수 있다.

하지만 서비스 내부에서는 구분하고 싶을 수 있다.

```
{
  "code": "USER_NOT_FOUND",
  "message": "사용자를 찾을 수 없습니다."
}
```

또는

```
{
  "code": "POST_NOT_FOUND",
  "message": "게시글을 찾을 수 없습니다."
}
```

즉 다음 두 가지가 서로 다른 역할을 갖는다.

```
HTTP Status
404 Not Found

Application Error Code
USER_NOT_FOUND
```

HTTP Status는 **HTTP 관점에서 요청이 어떻게 처리되었는지** 알려주고,

서비스 에러 코드는 **우리 서비스에서 정확히 어떤 문제가 발생했는지** 알려준다.

---

## 3. 페이징이나 추가 정보를 함께 내려주기 좋다

목록 API에서는 실제 데이터 외에도 여러 정보가 필요할 수 있다.

예를 들어 다음과 같다.

```
현재 페이지
페이지 크기
전체 데이터 개수
다음 페이지 존재 여부
```

이런 경우 공통 구조를 사용하면 `data`와 `meta`를 나눌 수 있다.

```
{
  "code": "SUCCESS",
  "data": [
    {
      "id": 1,
      "title": "첫 번째 글"
    }
  ],
  "meta": {
    "page": 1,
    "size": 20,
    "totalCount": 53
  }
}
```

`data`에는 실제 데이터를 넣고,

`meta`에는 데이터 자체가 아닌 부가 정보를 넣는 식이다.

목록 조회가 많은 서비스에서는 이런 구조가 유용할 수 있다.

---

## 4. 여러 클라이언트가 같은 API를 사용할 때 편하다

하나의 백엔드 API를 여러 클라이언트가 사용할 수도 있다.

```
Web
iOS
Android
Admin
Partner API
```

이때 API마다 응답 규칙이 다르면 각각의 클라이언트에서 처리해야 할 경우의 수가 늘어난다.

반대로 공통된 응답 규칙을 정해두면 여러 클라이언트가 같은 규칙을 공유할 수 있다.

---

# 그렇다면 모든 API를 공통 형식으로 감싸면 될까?

여기서 가장 중요한 부분이다.

**그렇지는 않다.**

공통 응답 형식은 편리하지만 모든 API에 억지로 적용하면 오히려 문제가 생길 수 있다.

---

# 1. HTTP Status의 의미가 흐려질 수 있다

가장 흔하게 볼 수 있는 문제다.

다음 응답을 보자.

```
HTTP/1.1 200 OK
```

```
{
  "success": false,
  "code": "USER_NOT_FOUND",
  "message": "사용자를 찾을 수 없습니다."
}
```

조금 이상하다.

HTTP에서는 성공했다고 말하고 있다.

```
200 OK
```

그런데 Body에서는 실패했다고 말한다.

```
success: false
```

클라이언트는 어떤 값을 기준으로 판단해야 할까?

이런 정책을 사용하는 서비스도 있지만, 특별한 이유가 없다면 HTTP Status를 의미 있게 사용하는 편이 이해하기 쉽다.

예를 들어 사용자를 찾지 못했다면 다음과 같이 표현할 수 있다.

```
HTTP/1.1 404 Not Found
```

```
{
  "code": "USER_NOT_FOUND",
  "message": "사용자를 찾을 수 없습니다."
}
```

이제 역할이 명확하다.

```
404
→ 요청한 리소스를 찾지 못했다.

USER_NOT_FOUND
→ 그 리소스가 사용자였다.
```

---

# 2. `success: true/false`가 정말 필요할까?

공통 응답 객체에서 자주 보이는 필드가 있다.

```
{
  "success": true
}
```

하지만 생각해볼 필요가 있다.

이미 HTTP Status가 있기 때문이다.

```
2xx → 성공
4xx → 클라이언트 요청 문제
5xx → 서버 문제
```

그렇다면 다음 두 정보가 중복될 수 있다.

```
HTTP 200 OK
success: true
```

물론 팀 정책상 `success` 필드를 사용하는 것은 문제가 아니다.

다만 단순히

> "공통 응답에는 원래 success가 들어가니까"

라는 이유로 넣을 필요는 없다.

각 필드가 어떤 역할을 하는지 먼저 생각하는 것이 좋다.

---

# 3. `204 No Content`와 잘 맞지 않을 수 있다

예를 들어 회원 삭제 API가 있다고 해보자.

삭제가 성공했고 클라이언트에게 특별히 돌려줄 데이터가 없다면 HTTP에서는 다음과 같은 응답을 사용할 수 있다.

```
HTTP/1.1 204 No Content
```

말 그대로 Body가 없는 응답이다.

그런데 모든 응답에 공통 Wrapper를 강제한다면 다음과 같이 만들어야 할 수도 있다.

```
{
  "code": "SUCCESS",
  "message": "삭제되었습니다.",
  "data": null
}
```

이것이 반드시 잘못된 것은 아니다.

다만 이렇게 결정했다면 더 이상 `204 No Content`를 사용하는 것은 자연스럽지 않다.

즉 공통 응답 형식을 도입할 때는

**HTTP의 표현을 사용할 것인지, Body의 표현을 사용할 것인지**

팀에서 기준을 정하는 것이 중요하다.

---

# 4. 파일 다운로드에는 적용하기 어렵다

다음과 같은 PDF 다운로드 API가 있다고 해보자.

```
HTTP/1.1 200 OK
Content-Type: application/pdf
Content-Disposition: attachment; filename="report.pdf"
```

이 API의 Body에는 JSON이 아니라 실제 PDF 파일 데이터가 들어간다.

따라서 다음과 같은 공통 JSON 구조로 감싸기가 어렵다.

```
{
  "code": "SUCCESS",
  "message": "성공",
  "data": ???
}
```

파일뿐만 아니라 이미지, 영상, Streaming 응답 등도 비슷하다.

그래서 실무에서는 보통

```
일반 JSON API → 공통 응답 정책 적용

파일/Streaming API → 별도 응답 정책 적용
```

처럼 예외를 두기도 한다.

---

# 에러 응답은 조금 더 중요하게 생각해야 한다

성공 응답 Wrapper는 프로젝트마다 선택이 다를 수 있다.

하지만 **에러 응답의 일관성은 특히 중요하다.**

예를 들어 어떤 API에서는 이런 에러가 내려오고,

```
{
  "message": "잘못된 요청입니다."
}
```

다른 API에서는 이렇게 내려오고,

```
{
  "error": "INVALID_REQUEST"
}
```

또 다른 API에서는 이렇게 내려온다면,

```
{
  "code": 1003,
  "reason": "Invalid request"
}
```

클라이언트 입장에서는 에러를 처리하기 어려워진다.

따라서 프로젝트에서 에러 구조는 가능한 한 일정하게 가져가는 것이 좋다.

예를 들어 다음과 같은 형태다.

```
{
  "code": "VALIDATION_ERROR",
  "message": "입력값이 올바르지 않습니다.",
  "errors": [
    {
      "field": "email",
      "message": "이메일 형식이 올바르지 않습니다."
    }
  ]
}
```

클라이언트는 항상 다음 규칙을 사용할 수 있다.

```
code
→ 어떤 종류의 에러인가?

message
→ 사용자에게 어떤 내용을 보여줄 것인가?

errors
→ 어떤 필드에 문제가 있는가?
```

---

# RFC 9457 Problem Details라는 표준도 있다

HTTP API의 에러 응답에는 직접 만든 형식만 사용할 수 있는 것은 아니다.

RFC 9457에서는 HTTP API의 에러 정보를 표현하기 위한 **Problem Details** 형식을 정의하고 있다.

예를 들면 다음과 같다.

```
{
  "type": "https://example.com/problems/user-not-found",
  "title": "User not found",
  "status": 404,
  "detail": "사용자를 찾을 수 없습니다."
}
```

여기서도 중요한 점은 HTTP Status를 버리지 않는다는 것이다.

```
HTTP/1.1 404 Not Found
```

Body에서는 HTTP Status만으로 설명하기 부족한 상세 정보를 추가한다.

```
type
title
status
detail
```

따라서 API를 설계할 때는 항상 다음 두 가지를 함께 생각하는 것이 좋다.

```
HTTP가 제공하는 정보
+
Body가 제공해야 하는 정보
```

---

# 성공과 실패가 반드시 같은 모양일 필요는 없다

공통 응답 형식을 처음 접하면 이런 생각을 하기 쉽다.

> 모든 응답은 무조건 같은 구조여야 하지 않을까?

하지만 반드시 그럴 필요는 없다.

예를 들어 성공 응답은 리소스를 바로 내려줄 수 있다.

```
{
  "id": 1,
  "name": "junior"
}
```

그리고 실패 응답만 공통된 형태로 만들 수도 있다.

```
{
  "code": "USER_NOT_FOUND",
  "message": "사용자를 찾을 수 없습니다."
}
```

또는 성공 응답에는 Wrapper를 사용하고,

```
{
  "code": "SUCCESS",
  "data": {
    "id": 1,
    "name": "junior"
  }
}
```

실패 응답은 별도의 Error Response를 사용할 수도 있다.

```
{
  "code": "USER_NOT_FOUND",
  "message": "사용자를 찾을 수 없습니다."
}
```

중요한 것은

**모든 응답이 정확히 같은 JSON 모양을 갖는 것**

그 자체가 아니다.

더 중요한 것은

**클라이언트가 응답 규칙을 예측할 수 있는가**

이다.

---

# 그렇다면 실무에서는 어떻게 정하면 좋을까?

정답은 프로젝트마다 다르다.

하지만 처음 API 응답 정책을 만든다면 다음 정도의 기준으로 시작해볼 수 있다.

### 일반적인 성공 응답

단순한 API라면 리소스를 그대로 반환한다.

```
{
  "id": 1,
  "name": "junior"
}
```

프로젝트에서 Wrapper의 장점이 충분하다면 다음처럼 사용한다.

```
{
  "code": "SUCCESS",
  "data": {
    "id": 1,
    "name": "junior"
  }
}
```

### 생성

리소스를 생성했다면 HTTP Status도 의미 있게 사용한다.

```
201 Created
```

### 삭제 후 반환할 내용이 없음

```
204 No Content
```

### 리소스를 찾지 못함

```
404 Not Found
```

```
{
  "code": "USER_NOT_FOUND",
  "message": "사용자를 찾을 수 없습니다."
}
```

### 잘못된 요청

```
400 Bad Request
```

```
{
  "code": "INVALID_REQUEST",
  "message": "요청값이 올바르지 않습니다."
}
```

### 중복이나 상태 충돌

```
409 Conflict
```

```
{
  "code": "EMAIL_ALREADY_EXISTS",
  "message": "이미 사용 중인 이메일입니다."
}
```

이처럼 HTTP Status와 서비스 코드를 각각 역할에 맞게 사용하는 방법이 이해하기 쉽다.

---

# 공통 응답 객체가 너무 커지는 것도 조심하자

프로젝트를 진행하다 보면 공통 응답 객체에 하나씩 필드가 추가되기 시작한다.

처음에는 단순했다.

```
public record ApiResponse<T>(
    String code,
    T data
) {
}
```

그러다가 요구사항이 생길 때마다 하나씩 추가된다.

```
public record ApiResponse<T>(
    boolean success,
    String code,
    String message,
    T data,
    Object error,
    Object meta,
    String traceId,
    LocalDateTime timestamp
) {
}
```

이렇게 되면 다시 문제가 생긴다.

```
error는 언제 사용하지?
message와 error의 차이는 뭐지?
meta는 어떤 API에서 사용하지?
data가 없으면 null인가?
success와 HTTP Status는 어떤 관계지?
```

공통으로 사용하기 위해 만든 객체가 오히려 사용법을 어렵게 만들 수 있다.

따라서 공통 응답 객체에는

**모든 상황에서 사용할지도 모르는 필드**

가 아니라

**실제로 공통으로 필요한 필드**

만 넣는 것이 좋다.

---

# API 문서에도 같은 규칙이 보여야 한다

코드에서 공통 응답 형식을 사용한다면 API 문서에도 실제 응답 형태가 보여야 한다.

예를 들어 실제 응답이 다음과 같은데,

```
{
  "code": "SUCCESS",
  "message": "요청이 성공했습니다.",
  "data": {
    "id": 1,
    "name": "junior"
  }
}
```

Swagger나 API 문서에서는 다음처럼 `data` 내부만 보여준다면,

```
{
  "id": 1,
  "name": "junior"
}
```

프론트엔드 개발자는 실제 응답 구조를 잘못 이해할 수 있다.

따라서 공통 응답 정책을 만들었다면

```
Controller
DTO
Error Response
Swagger / API 문서
Frontend 처리 방식
```

까지 같은 규칙을 바라보도록 만드는 것이 중요하다.

---

# 공통 응답 형식이 필요한지 판단하는 기준

공통 응답 형식을 만들기 전에 다음 질문을 해보면 좋다.

```
1. 클라이언트가 Wrapper를 통해 실제로 얻는 이점이 있는가?

2. HTTP Status와 중복되는 필드를 만들고 있지는 않은가?

3. 서비스 전용 code가 필요한가?

4. 성공 응답까지 반드시 같은 형태여야 하는가?

5. 페이징이나 meta 정보가 자주 필요한가?

6. 파일 다운로드나 Streaming API는 어떻게 처리할 것인가?

7. 204 No Content 같은 응답은 어떻게 처리할 것인가?

8. 에러 응답은 일관된 규칙을 가지고 있는가?

9. Swagger와 API 문서에도 같은 구조를 표현할 수 있는가?
```

이 질문에 답하다 보면

**공통 응답을 사용할지**

뿐만 아니라

**어디까지 공통으로 만들지**

도 자연스럽게 결정할 수 있다.

---

# 정리

공통 응답 형식을 사용하면 다음과 같은 장점이 있다.

```
API 응답 구조를 일정하게 만들 수 있다.

클라이언트에서 응답을 공통 처리하기 쉽다.

서비스 전용 에러 코드를 관리하기 좋다.

페이징이나 meta 정보를 함께 전달하기 좋다.

여러 클라이언트가 같은 응답 규칙을 공유할 수 있다.
```

반면 단점도 있다.

```
단순한 응답도 불필요하게 감싸질 수 있다.

HTTP Status와 역할이 중복될 수 있다.

success 같은 필드가 불필요한 중복이 될 수 있다.

204 No Content와 자연스럽게 맞지 않을 수 있다.

파일 다운로드나 Streaming 응답에는 적용하기 어렵다.

Wrapper가 너무 많은 역할을 가지면 오히려 복잡해진다.
```

결국 중요한 것은

> **공통 Response 클래스를 만드는 것 자체가 아니라, API가 일관되고 예측 가능한 규칙을 가지도록 만드는 것이다.**

공통 응답 형식을 사용해도 HTTP Status를 제대로 활용할 수 있고,

공통 응답 형식을 사용하지 않아도 충분히 일관된 API를 만들 수 있다.

따라서

```
"다른 프로젝트에서도 ApiResponse를 쓰니까 우리도 만들자."
```

보다는

```
"우리 서비스에서는 왜 ApiResponse가 필요한가?"
```

를 먼저 생각해보는 것이 좋다.

공통 응답 형식은 그 질문에 대한 답이 있을 때 가장 유용하다.
