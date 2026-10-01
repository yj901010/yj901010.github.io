---
title: "Front Controller 패턴에서 DispatcherServlet까지"
date: 2026-10-01 11:20:00 +0900
categories: [Backend, Spring]
tags: [spring-mvc, servlet, dispatcher-servlet, design-pattern]
description: 과거 FrontController 학습 노트를 바탕으로 Spring MVC의 DispatcherServlet이 왜 필요한지 요청 흐름 중심으로 정리합니다.
mermaid: true
---

> 과거 Tistory의 FrontController와 Spring MVC 학습 내용을 하나로 통합해 다시 작성했다.

Spring MVC를 처음 배우면 Controller에 @GetMapping을 붙이고 메서드를 작성하는 것부터 시작한다.

하지만 그 전에 "요청이 어떻게 이 Controller까지 도착하는가"를 이해하면 Spring MVC 구조가 훨씬 잘 보인다.

## Front Controller가 없을 때

Servlet을 URL마다 하나씩 만든다고 가정해보자.

~~~text
/member/list  -> MemberListServlet
/member/save  -> MemberSaveServlet
/member/edit  -> MemberEditServlet
/order/list   -> OrderListServlet
~~~

각 Servlet은 다음과 같은 공통 작업을 반복할 수 있다.

- Character Encoding 설정
- 인증 확인
- 공통 예외 처리
- 요청 파라미터 처리
- View 이동
- Logging

기능이 많아질수록 중복도 늘어난다.

## Front Controller 패턴

Front Controller 패턴은 모든 요청을 하나의 중앙 진입점에서 먼저 받는다.

~~~mermaid
flowchart LR
    A[Client] --> B[Front Controller]
    B --> C[Controller A]
    B --> D[Controller B]
    B --> E[Controller C]
~~~

공통 처리는 Front Controller가 담당하고, 실제 비즈니스 기능은 각 Controller에게 위임한다.

## 아주 단순한 구현

~~~java
@WebServlet("/")
public class FrontController extends HttpServlet {

    @Override
    protected void service(
            HttpServletRequest request,
            HttpServletResponse response
    ) throws ServletException, IOException {

        String uri = request.getRequestURI();

        if (uri.equals("/members")) {
            // MemberController 호출
        } else if (uri.equals("/orders")) {
            // OrderController 호출
        }
    }
}
~~~

물론 실제로 이렇게 if문을 계속 늘리면 금방 관리하기 어려워진다.

그래서 다음 단계로 요청 URL과 Controller를 매핑하는 구조가 필요해진다.

~~~java
Map<String, Controller> handlerMapping = new HashMap<>();

handlerMapping.put("/members", new MemberController());
handlerMapping.put("/orders", new OrderController());
~~~

여기서 이미 Spring MVC의 핵심 구조와 비슷한 모습이 보이기 시작한다.

## Spring MVC의 DispatcherServlet

Spring MVC는 Front Controller 패턴을 기반으로 한다.

중앙의 DispatcherServlet이 요청을 받고, 실제 처리는 여러 구성 요소에 위임한다.

~~~mermaid
flowchart LR
    A[Client] --> B[DispatcherServlet]
    B --> C[HandlerMapping]
    C --> D[HandlerAdapter]
    D --> E[Controller]
    E --> D
    D --> B
    B --> F[ViewResolver / Response]
    F --> A
~~~

Spring 공식 문서도 DispatcherServlet을 Front Controller라고 설명한다.

## HandlerMapping

DispatcherServlet은 요청을 받았다고 바로 특정 Controller를 직접 호출하지 않는다.

먼저 HandlerMapping을 통해 이 요청을 처리할 Handler를 찾는다.

예를 들어 다음 Controller가 있다.

~~~java
@RestController
@RequestMapping("/members")
public class MemberController {

    @GetMapping("/{id}")
    public MemberResponse find(@PathVariable Long id) {
        // ...
    }
}
~~~

GET /members/10 요청이 들어오면 HandlerMapping이 해당 메서드를 찾는 역할을 한다.

## HandlerAdapter

Handler를 찾았다고 해서 DispatcherServlet이 모든 종류의 Handler를 직접 실행하는 것도 아니다.

HandlerAdapter가 실제 Handler 호출 방법을 추상화한다.

이 구조 덕분에 DispatcherServlet은 특정 Controller 구현 방식에 강하게 의존하지 않는다.

## Controller

Controller는 웹 요청을 비즈니스 로직으로 연결하는 역할을 한다.

~~~java
@GetMapping("/{id}")
public MemberResponse find(@PathVariable Long id) {
    return memberService.find(id);
}
~~~

실제 업무 로직은 Service 계층으로 위임하는 것이 일반적이다.

## ViewResolver와 REST API

전통적인 Spring MVC에서는 Controller가 ModelAndView나 View 이름을 반환하고 ViewResolver가 실제 View를 찾았다.

REST API에서는 @ResponseBody 또는 @RestController를 사용하면서 View 렌더링 대신 HttpMessageConverter가 객체를 JSON 같은 응답 본문으로 변환하는 경우가 많다.

그래도 요청의 중앙 진입점이 DispatcherServlet이라는 사실은 같다.

## Filter와 DispatcherServlet

여기서 한 단계 더 보면 Servlet Filter는 DispatcherServlet보다 앞에서 동작한다.

~~~mermaid
flowchart LR
    A[Client] --> B[Filter Chain]
    B --> C[DispatcherServlet]
    C --> D[Controller]
~~~

Spring Security 역시 Servlet Filter 기반으로 동작한다.

그래서 인증과 인가 처리가 Controller보다 먼저 수행된다.

이 흐름을 이해하면 Spring Security를 볼 때도 구조가 연결된다.

## Interceptor는 어디에 있을까

HandlerInterceptor는 Spring MVC 영역에서 Controller 호출 전후에 동작한다.

Filter와 비슷해 보이지만 위치가 다르다.

~~~text
Client
  ↓
Servlet Filter
  ↓
DispatcherServlet
  ↓
HandlerInterceptor
  ↓
Controller
~~~

Filter는 Servlet Container 수준,
Interceptor는 Spring MVC 수준이라고 생각하면 구조를 이해하기 쉽다.

## Spring Boot에서는 왜 DispatcherServlet 설정을 거의 안 할까

Spring Boot는 Spring MVC 자동 설정을 제공하기 때문에 일반적인 애플리케이션에서는 DispatcherServlet을 직접 등록하는 코드를 거의 작성하지 않는다.

하지만 보이지 않는다고 없는 것은 아니다.

우리가 작성한 @RestController 뒤에는 여전히 다음 구조가 있다.

~~~text
Request
  ↓
DispatcherServlet
  ↓
HandlerMapping
  ↓
HandlerAdapter
  ↓
Controller
~~~

## Front Controller를 직접 구현해본 경험이 의미 있는 이유

예전에는 FrontController 패턴을 단순 디자인 패턴 하나로 배웠다.

지금 다시 보면 이 패턴을 직접 구현해보는 과정은 Spring MVC가 왜 DispatcherServlet, HandlerMapping, HandlerAdapter 같은 구조를 갖는지 이해하는 데 도움이 된다.

프레임워크를 사용할 때는 한 줄로 끝난다.

~~~java
@GetMapping("/members")
~~~

하지만 이 한 줄이 동작하려면 내부에는 요청 탐색, Handler 선택, 파라미터 바인딩, 반환값 처리, 예외 처리 등 많은 과정이 필요하다.

## 정리

Front Controller 패턴의 목적은 단순하다.

공통 요청 처리를 중앙화하고 실제 기능으로 요청을 위임한다.

Spring MVC는 이 아이디어를 DispatcherServlet을 중심으로 확장한다.

그래서 Spring MVC를 이해할 때 Controller 애노테이션만 외우기보다 DispatcherServlet부터 요청 흐름을 따라가면 전체 구조가 훨씬 명확해진다.

## 참고

- Spring Framework DispatcherServlet: https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-servlet.html
