---
title: "Spring Security에서 JWT 인증은 어떻게 동작하는가"
date: 2024-05-22
last_modified_at: 2026-10-01
categories: [Backend, Security]
tags: [spring-security, jwt, authentication, authorization]
description: 과거 JWT 학습 노트를 바탕으로 토큰 구조부터 Spring Security 인증 흐름까지 다시 정리합니다.
mermaid: true
---

> 과거 Tistory의 Spring Boot + JWT 학습 노트를 현재 기준으로 다시 정리했다.

처음 JWT를 공부했을 때는 Header, Payload, Signature 세 부분을 외우는 데 집중했다.

지금 다시 보면 더 중요한 질문은 이것이다.

"클라이언트가 JWT를 보내면 Spring Security는 어떻게 이 사용자를 인증된 사용자로 판단할까?"

## JWT 구조

JWT는 일반적으로 다음 세 부분으로 구성된다.

~~~text
Header.Payload.Signature
~~~

### Header

토큰의 타입과 서명 알고리즘 같은 메타데이터가 들어간다.

### Payload

사용자 식별자, 권한, 발급 시각, 만료 시각 같은 Claim이 들어갈 수 있다.

주의할 점은 Payload가 암호화된 영역이 아니라는 것이다.

Base64URL 인코딩이 되어 있을 뿐이므로 민감한 정보를 그대로 넣어서는 안 된다.

### Signature

Header와 Payload가 발급 이후 변조되지 않았는지 검증하는 데 사용된다.

## 인증과 인가는 다르다

JWT를 공부할 때 인증(Authentication)과 인가(Authorization)를 분리해서 보는 것이 중요하다.

- 인증: 이 요청을 보낸 사용자가 누구인가?
- 인가: 이 사용자가 이 자원에 접근할 권한이 있는가?

JWT는 사용자 정보를 전달하는 수단일 뿐이고, 실제 인증·인가 판단은 애플리케이션의 보안 구조가 담당한다.

## 요청이 들어왔을 때

일반적인 Bearer Token 방식은 다음과 같은 흐름을 가진다.

~~~mermaid
flowchart LR
    A[Client] -->|Authorization: Bearer JWT| B[Security Filter Chain]
    B --> C[JWT Parsing / Validation]
    C --> D[Authentication]
    D --> E[SecurityContext]
    E --> F[Controller]
~~~

Spring Security의 Servlet 기반 구조에서는 보안 Filter들이 요청을 DispatcherServlet보다 먼저 처리한다.

인증이 성공하면 Authentication 객체가 SecurityContext에 저장되고, 이후 Controller나 Service에서 현재 사용자 정보와 권한을 사용할 수 있다.

## Authentication 객체

Spring Security에서 인증된 사용자는 Authentication으로 표현된다.

Authentication에는 대략 다음 정보가 들어간다.

- principal: 사용자
- credentials: 인증에 사용된 자격 정보
- authorities: ROLE, Scope 같은 권한 정보
- authenticated: 인증 여부

JWT 자체가 곧 Authentication 객체인 것은 아니다.

JWT를 검증한 뒤 그 결과를 바탕으로 Authentication을 만들어 SecurityContext에 저장하는 과정이 필요하다.

## Spring Security Resource Server 방식

Spring Security의 Resource Server 기능을 사용하면 JWT 인증을 직접 Filter부터 전부 구현하지 않아도 된다.

개념적으로는 다음 흐름이다.

~~~text
Bearer Token
    ↓
BearerTokenAuthenticationToken
    ↓
AuthenticationManager
    ↓
JwtAuthenticationProvider
    ↓
JwtDecoder
    ↓
JwtAuthenticationToken
~~~

JwtDecoder는 서명, issuer, expiration 같은 토큰 유효성을 검증하고, JwtAuthenticationProvider는 그 결과를 Authentication으로 변환한다.

## 예시 설정

~~~java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/public/**").permitAll()
                .anyRequest().authenticated()
            )
            .oauth2ResourceServer(oauth2 -> oauth2.jwt());

        return http.build();
    }
}
~~~

실제 서비스에서는 JwtDecoder 설정이나 issuer-uri, 공개키 설정 등이 추가된다.

## 직접 JWT Filter를 만드는 방식

프로젝트에 따라 OncePerRequestFilter를 상속해 직접 JWT Filter를 구현하기도 한다.

~~~java
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    @Override
    protected void doFilterInternal(
            HttpServletRequest request,
            HttpServletResponse response,
            FilterChain filterChain
    ) throws ServletException, IOException {

        String token = resolveToken(request);

        if (token != null && validate(token)) {
            Authentication authentication = createAuthentication(token);

            SecurityContextHolder.getContext()
                    .setAuthentication(authentication);
        }

        filterChain.doFilter(request, response);
    }
}
~~~

학습에는 좋지만 직접 구현할수록 다음 책임도 직접 관리해야 한다.

- Token 추출
- 서명 검증
- 만료 검증
- Claim 검증
- Authentication 생성
- 예외 응답
- Filter 순서
- SecurityContext 관리

따라서 프레임워크가 제공하는 표준 지원으로 해결 가능한지 먼저 검토하는 것이 좋다.

## Access Token과 Refresh Token

Access Token은 일반적으로 API 요청에 사용하고 짧은 만료 시간을 둔다.

Refresh Token은 Access Token을 다시 발급하기 위한 수단으로 사용한다.

하지만 Refresh Token을 도입했다고 자동으로 안전해지는 것은 아니다.

다음 정책까지 같이 생각해야 한다.

- Refresh Token 저장 위치
- 탈취 시 폐기 방법
- Rotation 여부
- 로그아웃 처리
- 다중 기기 로그인 정책

## JWT의 장점

서버가 매 요청마다 세션 저장소에서 사용자 세션을 조회하지 않고도 토큰을 검증할 수 있는 구조를 만들 수 있다.

분산 환경에서도 인증 정보를 전달하기 편하다.

## JWT의 단점

발급된 토큰을 즉시 무효화하기 어렵다.

따라서 긴 만료 시간을 가진 Access Token을 무조건 사용하는 것은 위험하다.

또 JWT를 사용한다고 해서 인증 서버, 사용자 상태, 권한 변경, 로그아웃 문제까지 모두 Stateless해지는 것은 아니다.

## 예전에는 구조를 외웠고, 지금은 흐름을 본다

처음에는 Header, Payload, Signature를 알면 JWT를 이해했다고 생각했다.

지금은 다음 흐름을 더 중요하게 본다.

1. 토큰은 어디서 발급되는가?
2. 요청에서 누가 토큰을 읽는가?
3. 누가 서명과 Claim을 검증하는가?
4. 검증 결과가 어떻게 Authentication으로 변환되는가?
5. Authentication은 어디에 저장되는가?
6. 권한은 어디서 검사되는가?
7. 토큰을 폐기해야 할 때 어떻게 처리할 것인가?

JWT는 문자열 포맷보다 인증 시스템 전체 안에서 어떤 역할을 맡는지가 더 중요하다.

## 참고

- Spring Security Authentication Architecture: https://docs.spring.io/spring-security/reference/servlet/authentication/architecture.html
- Spring Security JWT Resource Server: https://docs.spring.io/spring-security/reference/servlet/oauth2/resource-server/jwt.html
