---
title: "브라우저 비동기 요청을 다시 정리하며: Ajax, fetch, Same-Origin Policy, CORS"
date: 2023-07-17
last_modified_at: 2026-10-01
categories: [Backend, HTTP]
tags: [ajax, fetch, cors, http, browser]
description: 과거 Ajax 학습 노트를 바탕으로 브라우저가 서버와 비동기로 통신하는 방식과 Same-Origin Policy, CORS의 관계를 다시 정리합니다.
---

> 과거 Tistory의 Ajax 학습 노트를 현재 기준으로 다시 정리했다.

원문 카테고리: https://it-r-b.tistory.com/category/Java/JavaScript

처음 Ajax를 배울 때는 "페이지 새로고침 없이 서버와 통신하는 방식"이라고 외웠다.

틀린 설명은 아니지만, 지금 다시 보면 Ajax라는 단어 자체보다 브라우저가 HTTP 요청을 보내고 응답을 받아 화면 일부만 갱신하는 흐름을 이해하는 것이 더 중요하다.

## Ajax는 특정 라이브러리 이름이 아니다

Ajax는 Asynchronous JavaScript and XML의 약자다.

이름에는 XML이 들어가지만 현대 웹 애플리케이션에서는 JSON을 훨씬 자주 사용한다.

핵심은 다음과 같다.

1. 사용자가 페이지에서 동작한다.
2. JavaScript가 서버에 HTTP 요청을 보낸다.
3. 전체 페이지를 다시 로드하지 않고 응답을 받는다.
4. 필요한 부분만 화면에 반영한다.

예전에는 XMLHttpRequest를 많이 사용했고, 현재는 fetch API나 Axios 같은 라이브러리를 자주 사용한다.

## fetch로 요청 보내기

~~~javascript
async function loadMember(id) {
  const response = await fetch(`/api/members/${id}`);

  if (!response.ok) {
    throw new Error(`HTTP error: ${response.status}`);
  }

  return response.json();
}
~~~

여기서 중요한 점은 fetch가 HTTP 상태 코드 404나 500이라고 해서 항상 Promise를 reject하는 것은 아니라는 것이다.

네트워크 자체가 실패한 경우와 HTTP 응답이 에러 상태인 경우를 구분해야 한다.

그래서 response.ok나 response.status를 확인하는 코드가 필요하다.

## 비동기 요청의 의미

비동기 요청이라는 말을 "서버가 동시에 여러 작업을 처리한다"는 뜻으로 오해하면 안 된다.

브라우저 입장에서 중요한 점은 요청을 보낸 뒤 JavaScript 실행 흐름 전체를 멈추고 응답만 기다리지 않는다는 것이다.

~~~javascript
console.log("before");

fetch("/api/data")
  .then(response => response.json())
  .then(data => console.log(data));

console.log("after");
~~~

출력은 일반적으로 다음 순서가 된다.

~~~text
before
after
data
~~~

HTTP 응답은 나중에 도착하고, 완료된 뒤 callback이나 Promise 후속 로직이 실행된다.

## Same-Origin Policy

브라우저는 보안 때문에 다른 Origin의 리소스 접근을 제한한다.

Origin은 일반적으로 다음 세 요소의 조합이다.

- Scheme
- Host
- Port

예를 들어 다음 두 주소는 Origin이 다르다.

~~~text
http://localhost:3000
http://localhost:8080
~~~

Port가 다르기 때문이다.

React 개발 서버와 Spring Boot 서버를 각각 실행하면 흔히 이 상황이 발생한다.

## "다른 도메인과 통신할 수 없다"는 표현은 정확하지 않다

예전 Ajax 노트에는 Same-Origin Policy 때문에 다른 도메인과 통신이 불가능하다고 정리했다.

지금 기준으로는 너무 단순한 설명이다.

브라우저는 Cross-Origin 요청 자체를 전부 금지하는 것이 아니라, 서버가 CORS 정책을 통해 해당 Origin의 접근을 허용할 수 있다.

예를 들어 Spring Boot에서 다음과 같이 특정 Origin을 허용할 수 있다.

~~~java
@Configuration
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")
                .allowedOrigins("http://localhost:3000")
                .allowedMethods("GET", "POST", "PATCH", "DELETE");
    }
}
~~~

핵심은 "브라우저 제한을 끈다"가 아니다.

서버가 어떤 Origin의 요청을 허용할지 응답 헤더를 통해 명시하는 것이다.

## CORS는 브라우저 보안 정책과 관련 있다

CORS를 공부할 때 서버와 브라우저 역할을 구분하면 이해하기 쉽다.

~~~text
Browser
  ↓
Cross-Origin HTTP Request
  ↓
Server
  ↓
Access-Control-Allow-Origin 응답
  ↓
Browser가 응답 사용 가능 여부 판단
~~~

Postman이나 서버 간 HTTP 요청에서는 브라우저의 Same-Origin Policy가 적용되지 않는다.

그래서 "Postman에서는 되는데 React에서는 안 된다"는 상황이 자주 발생한다.

이 경우 API 자체가 죽은 것이 아니라 CORS 설정 문제일 가능성이 있다.

## Preflight 요청

모든 Cross-Origin 요청이 바로 본 요청으로 전송되는 것은 아니다.

브라우저는 특정 조건에서 먼저 OPTIONS 요청을 보내 서버가 실제 요청을 허용하는지 확인한다.

이를 Preflight Request라고 한다.

예를 들어 다음과 같은 요청은 Preflight가 발생할 수 있다.

~~~http
PATCH /api/members/10
Origin: https://frontend.example.com
Content-Type: application/json
~~~

서버는 OPTIONS 요청에 대해 허용 Origin, Method, Header 등을 응답해야 한다.

Spring Security를 사용한다면 CORS 처리가 Security Filter Chain과 충돌하지 않도록 설정해야 한다.

## Cookie를 함께 보내는 경우

Cross-Origin 환경에서 Cookie나 인증 정보를 보내려면 추가로 신경 쓸 부분이 있다.

브라우저 요청에서 credentials 설정이 필요할 수 있다.

~~~javascript
fetch("https://api.example.com/me", {
  credentials: "include"
});
~~~

서버도 credentials를 허용해야 한다.

~~~java
registry.addMapping("/api/**")
        .allowedOrigins("https://frontend.example.com")
        .allowCredentials(true);
~~~

이때 allowedOrigins("*")와 credentials 허용을 무작정 같이 사용하는 방식은 피해야 한다.

인증 정보가 오가는 서비스라면 허용 Origin을 명시적으로 관리하는 편이 안전하다.

## Ajax보다 더 중요한 것

처음에는 Ajax 자체를 하나의 기술로 외웠다.

현재는 다음 흐름을 더 중요하게 본다.

~~~text
JavaScript
  ↓
HTTP Request
  ↓
Browser Security Policy
  ↓
CORS
  ↓
Backend API
  ↓
HTTP Response
  ↓
Promise / async-await
  ↓
UI Update
~~~

이 흐름을 이해하면 React, Vue, Vanilla JavaScript 중 무엇을 사용하더라도 API 호출 문제를 비슷한 방식으로 추적할 수 있다.

## 디버깅할 때 확인하는 순서

브라우저에서 API 요청이 실패한다면 다음 순서로 보는 편이 좋다.

1. 개발자도구 Network 탭에서 요청이 실제로 전송됐는가?
2. 요청 URL과 HTTP Method가 맞는가?
3. 상태 코드는 무엇인가?
4. OPTIONS 요청이 실패하지 않았는가?
5. Response Header에 CORS 관련 헤더가 있는가?
6. Cookie나 Authorization Header가 필요한가?
7. 백엔드 로그에는 요청이 도착했는가?

"CORS 에러"라는 메시지만 보고 무조건 백엔드 설정을 바꾸는 것보다 실제 요청 흐름을 먼저 확인해야 한다.

## 정리

Ajax는 페이지 일부를 비동기로 갱신하는 웹 개발 방식에서 출발한 용어다.

현대 웹에서는 fetch, Axios, React Query 같은 도구가 더 자주 보이지만 기반은 같다.

브라우저가 HTTP 요청을 보내고, Promise 기반으로 응답을 처리하며, Cross-Origin 요청에서는 Same-Origin Policy와 CORS가 개입한다.

결국 프론트엔드와 백엔드가 연결되는 지점에서는 JavaScript 문법보다 HTTP와 브라우저 보안 모델을 이해하는 것이 더 중요하다.

## 참고

- MDN Fetch API: https://developer.mozilla.org/docs/Web/API/Fetch_API
- MDN Same-Origin Policy: https://developer.mozilla.org/docs/Web/Security/Same-origin_policy
- MDN CORS: https://developer.mozilla.org/docs/Web/HTTP/CORS
