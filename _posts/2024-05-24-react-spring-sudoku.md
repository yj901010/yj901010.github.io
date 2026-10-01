---
title: "React + Spring Boot로 Sudoku Solver 만들기: CORS와 백트래킹"
date: 2024-05-24 15:58:00 +0900
last_modified_at: 2026-10-01 11:10:00 +0900
categories: [Project, Backend]
tags: [spring-boot, react, cors, backtracking, sudoku]
description: React와 Spring Boot로 스도쿠 검증·풀이 기능을 만들면서 CORS 문제와 백트래킹 알고리즘을 다뤘던 개인 프로젝트를 다시 정리합니다.
pin: true
mermaid: true
---

> 2024년에 작성했던 Tistory 글을 2026년 기준으로 다시 정리했다. 당시 구현 방식은 최대한 남기되, 지금 다시 보면 보완할 부분도 함께 기록한다.

원문: https://it-r-b.tistory.com/122

## 왜 만들었나

스도쿠를 직접 풀다가 한 문제에 한 시간 가까이 쓰면서, 입력한 문제를 검증하고 정답까지 보여주는 간단한 웹 페이지를 만들어보기로 했다.

구성은 단순했다.

- Frontend: React
- Backend: Spring Boot
- 기능 1: 사용자가 입력한 9x9 스도쿠가 유효한지 검증
- 기능 2: 입력한 문제의 정답 계산
- 이후 목표: AWS EC2 배포

## 전체 구조

~~~mermaid
flowchart LR
    U[User] --> R[React]
    R -->|POST /api/validate| S[Spring Boot]
    S --> V[Sudoku Validator]
    V --> S
    S --> R
    R --> B[Backtracking Solver]
~~~

당시에는 검증 로직을 Spring Boot에 두고, 정답을 계산하는 백트래킹 로직은 React 쪽에 두었다.

작은 개인 프로젝트라 동작 자체에는 문제가 없었지만, 지금 다시 설계한다면 핵심 도메인 로직을 어디에 둘지부터 다시 생각할 것 같다.

## 첫 번째 문제: React와 Spring Boot 사이의 CORS

개발 환경에서 React는 localhost:3000, Spring Boot는 localhost:8080을 사용했다.

브라우저 기준으로 두 주소는 Origin이 다르기 때문에 React에서 Spring Boot API를 호출하면 CORS 정책의 영향을 받는다.

당시에는 WebMvcConfigurer를 이용해 /api/** 요청에 대해 React 개발 서버 Origin을 허용했다.

~~~java
@Configuration
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")
                .allowedOrigins("http://localhost:3000")
                .allowedMethods("GET", "POST", "PUT", "DELETE")
                .allowCredentials(true);
    }
}
~~~

여기서 한 가지는 지금 기준으로 분명히 구분해야 한다.

CORS와 CSRF는 다른 문제다.

당시 코드에서는 Spring Security 설정에서 CSRF를 비활성화했지만, CSRF를 끄는 것이 CORS 해결 방법인 것은 아니다. 세션·쿠키 기반 인증을 사용하는 서비스라면 CSRF 보호를 단순히 비활성화하면 안 된다.

Spring Security를 함께 사용한다면 CORS 설정을 SecurityFilterChain과 일관되게 연결하는 것이 좋다.

## 서버에서 스도쿠 검증하기

유효한 완성 스도쿠라면 다음 조건을 모두 만족해야 한다.

1. 모든 칸이 채워져 있다.
2. 각 행에 1~9가 중복 없이 존재한다.
3. 각 열에 1~9가 중복 없이 존재한다.
4. 각 3x3 박스에 1~9가 중복 없이 존재한다.

행 검증은 Set을 이용해 간단히 구현할 수 있다.

~~~java
private boolean isValidRow(int[][] board, int row) {
    Set<Integer> values = new HashSet<>();

    for (int col = 0; col < 9; col++) {
        int value = board[row][col];

        if (value < 1 || value > 9) {
            return false;
        }

        if (!values.add(value)) {
            return false;
        }
    }

    return true;
}
~~~

열과 3x3 박스도 같은 원리로 검사하면 된다.

당시 코드에서는 값이 0인지와 중복 여부를 확인했는데, 지금 다시 작성한다면 1~9 범위 검증도 함께 넣는 편이 명확하다.

## 정답 계산: 백트래킹

스도쿠 풀이는 전형적인 백트래킹 문제다.

핵심은 다음과 같다.

1. 빈 칸을 찾는다.
2. 1부터 9까지 넣어본다.
3. 현재 위치에 넣어도 유효한 숫자라면 다음 칸으로 이동한다.
4. 이후 진행이 불가능하면 값을 다시 0으로 되돌리고 다른 숫자를 시도한다.

~~~typescript
const solve = (row: number, col: number): boolean => {
  if (row === 9) {
    return true;
  }

  const nextRow = col === 8 ? row + 1 : row;
  const nextCol = col === 8 ? 0 : col + 1;

  if (board[row][col] !== 0) {
    return solve(nextRow, nextCol);
  }

  for (let num = 1; num <= 9; num++) {
    if (isValid(row, col, num)) {
      board[row][col] = num;

      if (solve(nextRow, nextCol)) {
        return true;
      }

      board[row][col] = 0;
    }
  }

  return false;
};
~~~

이 구조는 가능한 경우를 선택하고, 실패하면 이전 상태로 되돌아가는 백트래킹의 기본 형태다.

## 지금 다시 만든다면

### 1. 배열 복사를 더 명확하게 한다

당시에는 다음과 같은 코드가 있었다.

~~~typescript
const solvedGrid = [...grid];
~~~

하지만 이 방식은 바깥 배열만 복사하는 얕은 복사다. 내부 배열은 기존 grid와 같은 참조를 공유한다.

따라서 현재는 다음처럼 복사하는 편이 안전하다.

~~~typescript
const solvedGrid = grid.map(row => [...row]);
~~~

### 2. 검증과 풀이의 책임을 정한다

검증은 서버, 풀이는 클라이언트에 두면 동일한 스도쿠 규칙을 양쪽에서 각각 구현하게 된다.

지금 다시 만든다면 두 가지 중 하나를 선택할 것 같다.

- 학습 목적이면 클라이언트와 서버를 명확히 분리해 각각의 역할을 실험한다.
- 서비스 목적이면 검증과 풀이를 하나의 도메인 로직으로 모아 중복을 줄인다.

### 3. CORS 설정은 환경별로 분리한다

localhost Origin을 코드에 직접 넣기보다는 개발·운영 환경별 설정으로 분리하고, 필요한 Origin만 허용한다.

## 이 프로젝트에서 얻은 것

규모가 작은 프로젝트였지만 세 가지를 한 번에 경험했다.

- React와 Spring Boot를 연결하면서 브라우저의 Same-Origin Policy와 CORS를 만났다.
- 서버에서 요청 데이터를 검증하는 로직을 작성했다.
- 백트래킹으로 실제 문제를 해결했다.

처음에는 스도쿠를 빨리 풀기 위해 만든 도구였지만, 결과적으로 프론트엔드와 백엔드를 연결하는 과정에서 생기는 문제를 직접 확인해본 작은 실험이 됐다.

## 참고

- Spring Framework CORS: https://docs.spring.io/spring-framework/reference/web/webmvc-cors.html
- Spring Security CORS: https://docs.spring.io/spring-security/reference/servlet/integrations/cors.html
