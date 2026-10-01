---
title: "REST API를 다시 정리하며: URL보다 중요한 리소스와 HTTP Method"
date: 2023-09-13
last_modified_at: 2026-10-01
categories: [Backend, HTTP]
tags: [rest, http, api, spring]
description: 과거 Spring REST 학습 노트를 바탕으로 URL 중심 설계에서 리소스와 HTTP Method 중심 설계로 관점을 다시 정리합니다.
---

> 과거 Tistory의 Spring - 0913 REST 전송방식 노트를 현재 관점에서 다시 정리했다.

처음 REST를 배울 때는 기존 URL을 어떻게 바꾸는지에 집중했다.

예를 들어 게시판 기능이 다음처럼 존재한다고 생각했다.

- boardList.do
- boardInsert.do
- boardUpdate.do
- boardDelete.do
- boardCount.do

당시에는 이것을 GET, POST, PUT, DELETE와 연결해 URL을 단순화한다고 이해했다.

방향은 맞았지만, 지금 다시 보면 REST에서 더 중요한 것은 동사 이름을 없애는 것 자체가 아니다.

핵심은 URI가 리소스를 표현하고, HTTP Method가 그 리소스에 수행할 행위를 표현하도록 설계하는 것이다.

## URL에 행위를 넣는 방식

~~~text
GET  /boardList.do
POST /boardInsert.do
POST /boardUpdate.do
POST /boardDelete.do
~~~

이 방식에서는 URL 자체가 명령을 설명한다.

기능이 늘어날수록 URL 이름도 계속 늘어난다.

## 리소스 중심으로 바꾸기

게시글이라는 리소스를 posts라고 표현하면 다음처럼 정리할 수 있다.

| 작업 | HTTP Method | URI |
|---|---|---|
| 게시글 목록 조회 | GET | /posts |
| 게시글 한 건 조회 | GET | /posts/{id} |
| 게시글 생성 | POST | /posts |
| 게시글 전체 교체 | PUT | /posts/{id} |
| 게시글 일부 수정 | PATCH | /posts/{id} |
| 게시글 삭제 | DELETE | /posts/{id} |

URI에서는 posts라는 리소스가 유지되고 작업의 의미는 HTTP Method가 담당한다.

## 예전 노트에서 수정해야 할 부분

과거 메모에는 다음과 같이 적혀 있었다.

GET, POST, PUT, DELETE, UPDATE

여기서 UPDATE는 HTTP Method가 아니다.

일반적으로 수정에는 PUT 또는 PATCH를 사용한다.

### PUT

대상 리소스를 요청 표현으로 교체하는 의미에 가깝다.

~~~http
PUT /posts/10
Content-Type: application/json

{
  "title": "새 제목",
  "content": "새 내용"
}
~~~

### PATCH

리소스의 일부를 수정하는 의미로 사용한다.

~~~http
PATCH /posts/10
Content-Type: application/json

{
  "title": "제목만 변경"
}
~~~

실제 API에서는 팀의 규칙과 서버 구현에 따라 세부 의미가 달라질 수 있지만, PUT과 PATCH의 의미 차이를 알고 선택하는 것이 좋다.

## 조회수 증가 API는 어떻게 볼까

예전에는 boardCount.do 같은 URL을 사용했다.

이 기능을 REST 스타일로 바꾸려 할 때 무조건 다음처럼 만들 필요는 없다.

~~~text
POST /posts/{id}/count
~~~

조회수가 단순히 게시글 조회에 따른 부수 효과라면 GET /posts/{id} 처리 과정에서 증가시킬 수도 있다.

하지만 이 경우 GET이 서버 상태를 변경하므로 HTTP의 safe method 의미와 충돌한다.

그래서 조회수 정책이 중요하다면 별도 이벤트나 비동기 카운팅, 중복 조회 방지 정책 등을 함께 고려해야 한다.

URL 모양만 REST처럼 만드는 것보다 의미가 일관적인지가 더 중요하다.

## Spring에서는

Spring MVC에서는 애노테이션으로 Method와 URI를 함께 표현한다.

~~~java
@RestController
@RequestMapping("/posts")
public class PostController {

    @GetMapping
    public List<PostResponse> findAll() {
        // ...
    }

    @PostMapping
    public PostResponse create(@RequestBody CreatePostRequest request) {
        // ...
    }

    @PatchMapping("/{id}")
    public PostResponse update(
            @PathVariable Long id,
            @RequestBody UpdatePostRequest request
    ) {
        // ...
    }

    @DeleteMapping("/{id}")
    public void delete(@PathVariable Long id) {
        // ...
    }
}
~~~

## 지금 REST API를 설계할 때 보는 것

예전에는 URL 이름을 먼저 생각했다.

지금은 다음 순서가 더 자연스럽다고 본다.

1. 어떤 리소스를 다루는가?
2. 리소스를 식별하는 URI는 무엇인가?
3. 어떤 HTTP Method가 의미에 맞는가?
4. 요청과 응답 상태 코드는 무엇인가?
5. 멱등성이 필요한가?
6. 실패 응답은 어떻게 일관되게 표현할 것인가?

REST는 단순히 .do URL을 없애는 규칙이 아니라 HTTP의 의미를 최대한 활용해 API의 인터페이스를 일관되게 만드는 설계 방식으로 이해하는 편이 좋다.
