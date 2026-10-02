---
title: Guide
icon: fas fa-compass
order: 1
---

처음 방문했다면 아래 순서로 읽는 것을 추천합니다. 백엔드 요청 흐름부터 API 설계, 아키텍처, 데이터베이스와 운영 경험까지 이어지도록 정리했습니다.

## 1. Spring 백엔드의 전체 흐름

- [낯선 백엔드 프로젝트를 처음 열었을 때 무엇부터 봐야 할까?](/posts/reading-unfamiliar-backend-project/)
- [Controller, Service, Repository는 왜 나눌까?](/posts/controller-service-repository/)
- [Spring 백엔드에서 API 요청 하나는 어떻게 처리될까?](/posts/spring-api-request-flow/)
- [Front Controller 패턴에서 DispatcherServlet까지](/posts/front-controller-dispatcher-servlet/)

## 2. API와 DTO 설계

- [DTO를 쓰는 이유](/posts/why-use-dto/)
- [엔터티를 API 응답으로 반환하면 안되는 이유](/posts/entity-api-response/)
- [Request DTO와 Response DTO는 왜 따로 만들까?](/posts/request-response-dto/)
- [Spring Validation은 어디서 해야 할까?](/posts/spring-validation-boundaries/)
- [Spring Boot 예외 처리, 왜 한 곳에서 관리할까?](/posts/spring-global-exception-handler/)
- [Spring API 응답, 꼭 공통 형식으로 감싸야 할까?](/posts/spring-api-response-wrapper/)
- [REST API를 다시 정리하며: URL보다 중요한 리소스와 HTTP Method](/posts/rest-api-http-methods/)

## 3. 아키텍처와 책임 분리

- [Spring Boot로 이해하는 레이어드 아키텍처](/posts/spring-layered-architecture/)
- [Spring Boot 패키지 구조, 어떻게 나눠야 할까? 레이어형과 도메인형 비교](/posts/spring-package-structure/)
- [의존성 방향은 왜 중요할까?](/posts/dependency-direction/)
- [비즈니스 로직은 Service에만 넣으면 될까?](/posts/business-logic-service-domain/)
- [백엔드에서 공통화가 오히려 독이 되는 순간](/posts/backend-over-abstraction/)

## 4. Database와 성능

- [JDBC는 DB와 어떻게 통신하는가: Connection부터 PreparedStatement까지](/posts/jdbc-flow/)
- [회사 DB 한 테이블 전체를 Update 해버렸다?](/posts/database-full-update-retrospective/)
- [800GB 테이블 조회 12초에서 0.4초가 된 이유?](/posts/800gb-index-performance/)

## 5. 보안과 환경 설정

- [Spring Security에서 JWT 인증은 어떻게 동작하는가](/posts/spring-security-jwt/)
- [환경변수는 왜 사용하는 걸까?](/posts/why-environment-variables/)
- [비밀번호를 코드에서 분리해야 하는 이유](/posts/separate-secrets-from-code/)

## 6. 코드 품질과 Java

- [좋은 메서드 이름은 어떻게 지을까?](/posts/method-naming/)
- [주석보다 코드가 먼저다: 초보자를 위한 좋은 주석 작성법](/posts/good-comments/)
- [Java 객체지향을 다시 정리하며: 오버로딩, 오버라이딩, 다형성](/posts/java-oop-revisited/)

프로젝트 단위의 기술 선택과 구현 경험은 [Projects](/projects/)에서 확인할 수 있습니다.
