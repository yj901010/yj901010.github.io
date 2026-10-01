---
title: "낯선 백엔드 프로젝트를 처음 열었을 때 무엇부터 봐야 할까?"
date: 2026-08-02 18:47:51 +0900
last_modified_at: 2026-10-01
categories: [Backend, Engineering]
tags: [code-reading, spring, project]
source_url: "https://velog.io/@joker901010/%EB%82%AF%EC%84%A0-%EB%B0%B1%EC%97%94%EB%93%9C-%ED%94%84%EB%A1%9C%EC%A0%9D%ED%8A%B8%EB%A5%BC-%EC%B2%98%EC%9D%8C-%EC%97%B4%EC%97%88%EC%9D%84-%EB%95%8C-%EB%AC%B4%EC%97%87%EB%B6%80%ED%84%B0-%EB%B4%90%EC%95%BC-%ED%95%A0%EA%B9%8C"
---
새로운 회사에 입사하거나 다른 사람이 만든 프로젝트를 처음 맡으면 가장 먼저 이런 생각이 든다.

> “파일이 너무 많은데 어디부터 봐야 하지?”

프로젝트를 열어보면 `controller`, `service`, `repository`, `domain`, `config`, `common`, `infra` 같은 폴더가 한꺼번에 보인다.

이때 눈에 띄는 파일부터 무작정 열기 시작하면 금방 길을 잃기 쉽다.

낯선 백엔드 프로젝트를 이해할 때 중요한 것은 모든 파일을 읽는 것이 아니다. 먼저 프로젝트가 어떻게 실행되고, 요청이 어디로 들어오며, 내부에서 어떤 순서로 처리되는지 전체 지도를 만드는 것이다.

내가 처음 프로젝트를 볼 때 확인하는 순서는 다음과 같다.

1. 빌드 파일
2. 프로젝트 모듈 구성
3. 실행 진입점
4. 설정 파일
5. 패키지 구조
6. 요청 진입점
7. 핵심 비즈니스 흐름
8. 테스트 코드

이 순서대로 살펴보면 처음 보는 프로젝트도 훨씬 빠르게 이해할 수 있다.

---

## 프로젝트 구조를 본다는 것은 지도를 그리는 일이다

낯선 백엔드 프로젝트는 처음 가보는 건물과 비슷하다.

건물에 들어가자마자 아무 방이나 열어보면 전체 구조를 알기 어렵다. 먼저 안내도를 보고 입구와 층별 구조를 파악해야 한다.

백엔드 프로젝트도 마찬가지다.

- 어떻게 빌드하는가?
- 어디에서 애플리케이션이 시작되는가?
- 설정은 어디에 있는가?
- 외부 요청은 어디로 들어오는가?
- 비즈니스 로직은 어디에서 처리되는가?
- 데이터베이스에는 어떻게 접근하는가?
- 테스트는 어디에 있는가?

이 질문에 답할 수 있다면 아직 모든 코드를 읽지 않았더라도 프로젝트의 큰 구조는 파악한 것이다.

---

## 1. 가장 먼저 빌드 파일을 확인한다

Java 백엔드 프로젝트는 주로 Maven이나 Gradle을 사용한다.

Maven 프로젝트라면 다음 파일을 확인한다.

```
pom.xml
```

Gradle 프로젝트라면 다음 파일을 확인한다.

```
build.gradle
settings.gradle
```

Kotlin DSL을 사용한다면 파일 이름이 다음과 같을 수 있다.

```
build.gradle.kts
settings.gradle.kts
```

빌드 파일에서는 다음 내용을 확인한다.

- Maven과 Gradle 중 무엇을 사용하는가?
- Java 버전은 무엇인가?
- Spring Boot 버전은 무엇인가?
- 어떤 라이브러리를 사용하는가?
- 데이터베이스는 무엇인가?
- 테스트 도구는 무엇인가?
- 여러 모듈로 나뉜 프로젝트인가?

예를 들어 `build.gradle`에 다음과 같은 내용이 있다고 해보자.

```
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    runtimeOnly 'org.postgresql:postgresql'
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
}
```

이 몇 줄만 보더라도 프로젝트의 성격을 어느 정도 추측할 수 있다.

- `spring-boot-starter-web`: 웹 API 서버
- `spring-boot-starter-data-jpa`: JPA를 사용한 데이터 접근
- `postgresql`: PostgreSQL 데이터베이스 사용
- `spring-boot-starter-test`: Spring Boot 테스트 환경 사용

즉, 세부 코드를 보기 전에도 이 프로젝트가 어떤 기술 위에서 동작하는지 알 수 있다.

---

## 2. 멀티 모듈 프로젝트인지 확인한다

Gradle 프로젝트라면 `settings.gradle`도 함께 확인한다.

예를 들어 다음과 같은 내용이 있을 수 있다.

```
include 'api'
include 'domain'
include 'batch'
```

이 경우 하나의 애플리케이션만 있는 것이 아니라 여러 모듈이 역할별로 나뉘어 있다는 뜻이다.

예를 들면 다음처럼 구성되어 있을 수 있다.

```
project
├── api
├── domain
├── batch
└── build.gradle
```

- `api`: 외부 요청을 받는 API 서버
- `domain`: 핵심 비즈니스 규칙과 도메인 모델
- `batch`: 정기 작업이나 대량 데이터 처리

멀티 모듈 프로젝트에서는 각 모듈의 역할과 의존 관계를 먼저 확인해야 한다.

예를 들어 `api` 모듈이 `domain` 모듈을 참조하고 있다면, 실제 핵심 로직은 `api`가 아니라 `domain`에 있을 가능성이 크다.

모듈 구성을 모른 채 `Controller`만 따라가면 중요한 코드를 놓칠 수 있다.

---

## 3. 실행 진입점을 찾는다

Spring Boot 프로젝트에서는 보통 `main` 메서드가 있는 클래스를 찾는다.

```
@SpringBootApplication
public class MyApplication {

    public static void main(String[] args) {
        SpringApplication.run(MyApplication.class, args);
    }
}
```

이 클래스가 애플리케이션의 실행 진입점이다.

Spring Boot 공식 문서에서는 이 main application class를 프로젝트의 root package에 두는 방식을 권장한다.

그 이유는 `@SpringBootApplication`이 붙은 클래스의 패키지가 기본적인 컴포넌트 탐색 범위의 기준이 되기 때문이다.

예를 들어 다음과 같은 구조가 있다.

```
com.example.app
├── AppApplication.java
├── user
│   ├── UserController.java
│   ├── UserService.java
│   └── UserRepository.java
└── order
    ├── OrderController.java
    ├── OrderService.java
    └── OrderRepository.java
```

`AppApplication`이 `com.example.app`에 있다면 그 아래의 `user`, `order` 패키지가 자연스럽게 탐색 대상에 포함된다.

반대로 main class가 너무 깊은 위치에 있거나 주요 패키지가 그 바깥에 있다면 컴포넌트가 등록되지 않는 문제가 발생할 수 있다.

그래서 실행 진입점을 찾은 뒤에는 다음을 함께 확인한다.

- `@SpringBootApplication`이 붙은 클래스는 어디에 있는가?
- 주요 패키지들이 그 하위에 있는가?
- 별도의 `@ComponentScan` 설정이 있는가?
- 어떤 모듈이 실제 실행 가능한 애플리케이션인가?

---

## 4. 설정 파일을 확인한다

다음으로 `src/main/resources`를 확인한다.

일반적인 Maven과 Gradle 프로젝트에서는 애플리케이션 설정 파일이 이 위치에 들어간다.

```
src/main/resources
├── application.yml
├── application-dev.yml
├── application-prod.yml
└── logback-spring.xml
```

여기서는 다음 내용을 확인한다.

- 어떤 profile을 사용하는가?
- 데이터베이스 설정은 어디에서 가져오는가?
- Redis, Kafka, 외부 API 설정이 있는가?
- 서버 포트는 무엇인가?
- 로그 설정은 어떻게 되어 있는가?
- 민감한 정보가 직접 작성되어 있지는 않은가?

예를 들어 다음과 같은 설정이 있을 수 있다.

```
spring:
  datasource:
    url: ${DB_URL}
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
```

이 설정을 보면 데이터베이스 접속 정보를 코드에 직접 작성하지 않고 환경변수에서 가져온다는 것을 알 수 있다.

따라서 로컬에서 프로젝트를 실행하려면 다음 값들이 필요할 수 있다.

```
DB_URL
DB_USERNAME
DB_PASSWORD
```

설정 파일은 단순히 값을 모아놓은 파일이 아니다.

프로젝트가 실행될 때 어떤 외부 환경을 필요로 하는지 알려주는 실행 설명서에 가깝다.

---

## 5. 패키지 구조를 확인한다

이제 `src/main/java` 아래의 구조를 살펴본다.

```
src/main/java
└── com/example/app
    ├── user
    ├── order
    ├── common
    └── config
```

백엔드 프로젝트의 패키지 구조는 크게 두 가지 형태로 많이 나뉜다.

### 레이어 기준 구조

기술적 역할에 따라 패키지를 나누는 방식이다.

```
com.example.app
├── controller
│   └── UserController.java
├── service
│   └── UserService.java
├── repository
│   └── UserRepository.java
├── dto
│   └── UserResponse.java
└── entity
    └── User.java
```

같은 종류의 코드가 한곳에 모여 있기 때문에 구조가 직관적이다.

하지만 기능이 많아지면 하나의 기능을 이해하기 위해 여러 패키지를 계속 이동해야 할 수 있다.

### 도메인 기준 구조

사용자, 주문, 결제처럼 기능이나 업무 영역을 기준으로 나누는 방식이다.

```
com.example.app
├── user
│   ├── UserController.java
│   ├── UserService.java
│   ├── UserRepository.java
│   └── User.java
├── order
│   ├── OrderController.java
│   ├── OrderService.java
│   └── OrderRepository.java
└── payment
    ├── PaymentService.java
    └── PaymentClient.java
```

하나의 기능과 관련된 코드가 가까이 모여 있어 기능 단위로 흐름을 따라가기 쉽다.

어느 방식이 무조건 더 좋다고 말하기는 어렵다.

처음 프로젝트를 볼 때는 구조를 평가하기보다 다음 질문부터 확인하는 것이 좋다.

- 이 프로젝트는 무엇을 기준으로 패키지를 나누었는가?
- 동일한 기준이 프로젝트 전체에서 유지되는가?
- 하나의 기능을 따라가기 쉬운가?
- 핵심 비즈니스 로직은 어느 패키지에 있는가?

---

## 6. 외부 요청이 들어오는 지점을 찾는다

웹 API 서버라면 요청은 보통 Controller로 들어온다.

```
@RestController
@RequestMapping("/users")
public class UserController {

    private final UserService userService;

    @GetMapping("/{id}")
    public UserResponse getUser(@PathVariable Long id) {
        return userService.getUser(id);
    }
}
```

Controller를 보면 외부에 공개된 API의 형태를 알 수 있다.

- URL은 어떻게 구성되어 있는가?
- GET, POST, PUT, PATCH, DELETE 중 무엇을 사용하는가?
- 요청 데이터는 어떤 DTO로 받는가?
- 응답 데이터는 어떤 형태인가?
- 어떤 Service를 호출하는가?
- 인증이나 권한 검사가 적용되어 있는가?

낯선 기능을 파악할 때는 Controller에서 시작해 아래로 내려가는 방식이 이해하기 쉽다.

```
Controller
→ Service
→ Repository
→ Entity 또는 Query
```

다만 모든 요청이 Controller로만 들어오는 것은 아니다.

프로젝트에 따라 다음과 같은 진입점도 있을 수 있다.

- `Scheduler`: 정해진 시간마다 실행되는 작업
- `Listener`: Kafka나 RabbitMQ 메시지를 처리하는 코드
- `Batch Job`: 대량 데이터 처리
- `EventHandler`: 내부 이벤트 처리
- `Filter`: HTTP 요청 전처리
- `Interceptor`: 인증, 로깅, 권한 확인
- `CommandLineRunner`: 애플리케이션 시작 시 실행되는 작업

따라서 프로젝트의 성격에 따라 “요청 진입점”이 무엇인지 먼저 확인해야 한다.

---

## 7. 요청 하나를 끝까지 따라간다

전체 구조를 대략 확인했다면 이제 실제 기능 하나를 선택해 흐름을 따라가 본다.

예를 들어 회원 조회 기능이 있다고 해보자.

### Controller

```
@GetMapping("/users/{id}")
public UserResponse getUser(@PathVariable Long id) {
    return userService.getUser(id);
}
```

외부에서 다음과 같은 요청이 들어온다.

```
GET /users/1
```

Controller는 사용자 ID를 받아 `UserService`에 전달한다.

### Service

```
public UserResponse getUser(Long id) {
    User user = userRepository.findById(id)
            .orElseThrow(() -> new UserNotFoundException(id));

    return new UserResponse(user.getId(), user.getName());
}
```

Service에서는 사용자를 조회하고, 존재하지 않으면 예외를 발생시킨다.

조회된 Entity는 응답 DTO로 변환한다.

### Repository

```
public interface UserRepository extends JpaRepository<User, Long> {
}
```

Repository는 실제 데이터베이스 조회를 담당한다.

### Entity

```
@Entity
public class User {

    @Id
    private Long id;

    private String name;
}
```

Entity는 데이터베이스의 사용자 정보를 표현한다.

전체 흐름을 정리하면 다음과 같다.

```
GET /users/{id}
→ UserController
→ UserService
→ UserRepository
→ User Entity
→ UserResponse
```

프로젝트를 이해할 때 중요한 것은 클래스 이름을 외우는 것이 아니다.

요청 하나가 들어왔을 때 어떤 객체를 거쳐 데이터베이스에 접근하고, 결과가 어떤 형태로 반환되는지 파악하는 것이다.

---

## 8. 공통 코드와 핵심 기능 코드를 구분한다

프로젝트에는 비즈니스 기능 외에도 여러 공통 패키지가 있다.

```
common
config
exception
security
util
infra
```

이름은 프로젝트마다 다르지만 일반적으로 다음 역할을 맡는다.

### `common`

여러 기능에서 공통으로 사용하는 코드가 들어간다.

- 공통 응답 객체
- 공통 상수
- 공통 인터페이스
- 공통 타입

### `config`

Spring과 외부 라이브러리의 설정 코드가 들어간다.

- Bean 등록
- CORS 설정
- Jackson 설정
- Redis 설정
- Kafka 설정
- OpenAPI 설정

### `exception`

예외 처리와 관련된 코드가 들어간다.

- 커스텀 예외
- 에러 코드
- 전역 예외 처리
- 오류 응답 객체

### `security`

인증과 인가를 처리한다.

- Spring Security 설정
- JWT 처리
- 로그인 필터
- 권한 확인
- 인증 객체

### `infra`

외부 시스템과 연결되는 코드가 들어간다.

- 외부 API 호출
- 이메일 발송
- 파일 저장소
- 메시지 큐
- 클라우드 서비스
- 결제 시스템

### `util`

특정 비즈니스 규칙과 직접 관련이 적은 보조 코드가 들어간다.

- 날짜 변환
- 문자열 처리
- 암호화
- 파일 변환

처음부터 이 패키지들을 모두 깊게 읽을 필요는 없다.

핵심 기능을 따라가다가 해당 코드가 등장했을 때 어떤 역할인지 확인하는 정도로 시작하면 된다.

특히 `common`과 `util`은 프로젝트마다 범위가 매우 다르기 때문에 폴더 이름만 보고 역할을 단정해서는 안 된다.

---

## 9. 테스트 코드를 확인한다

테스트 코드는 보통 `src/test/java` 아래에 있다.

```
src/test/java
└── com/example/app
    ├── user
    │   └── UserServiceTest.java
    └── order
        └── OrderServiceTest.java
```

테스트는 코드의 정상 동작만 검증하는 것이 아니다.

잘 작성된 테스트는 다음 내용을 알려주는 문서 역할도 한다.

- 어떤 입력이 들어오는가?
- 어떤 조건에서 성공하는가?
- 어떤 조건에서 실패하는가?
- 어떤 예외가 발생하는가?
- 중요한 비즈니스 규칙은 무엇인가?

예를 들어 다음과 같은 테스트가 있다고 해보자.

```
@Test
void 이미_가입된_이메일이면_회원가입에_실패한다() {
    // ...
}
```

테스트 이름만 보더라도 이 서비스에는 다음 규칙이 있다는 것을 알 수 있다.

> 동일한 이메일로 중복 가입할 수 없다.

또 다른 예를 보자.

```
@Test
void 재고가_부족하면_주문할_수_없다() {
    // ...
}
```

이 테스트에서는 주문 처리 전에 재고를 확인한다는 사실을 알 수 있다.

복잡한 Service 코드를 처음부터 모두 읽는 것보다 테스트 목록을 먼저 확인하는 편이 핵심 규칙을 빠르게 파악하는 데 도움이 될 때가 많다.

---

## 파일 이름만 보고 역할을 단정하지 않는다

패키지와 클래스 이름은 프로젝트마다 다르다.

비즈니스 로직이 반드시 `Service`라는 이름의 클래스에만 있는 것은 아니다.

다음과 같은 이름이 사용될 수도 있다.

```
Manager
Facade
Processor
Handler
UseCase
ApplicationService
CommandHandler
```

예를 들어 `OrderFacade`가 여러 Service를 묶어 전체 주문 과정을 조정할 수도 있고, `PaymentProcessor`가 실제 결제 규칙을 처리할 수도 있다.

따라서 이름만 보고 판단하지 말고 실제로 어떤 객체를 호출하고 어떤 상태를 변경하는지 확인해야 한다.

---

## 데이터베이스 접근 방식도 확인한다

`Repository`를 찾았다면 어떤 방식으로 데이터베이스에 접근하는지도 살펴보는 것이 좋다.

프로젝트에서는 다음과 같은 기술을 사용할 수 있다.

- Spring Data JPA
- QueryDSL
- MyBatis
- JdbcTemplate
- jOOQ
- 직접 작성한 SQL
- 외부 저장소 접근 모듈

예를 들어 다음 코드는 Spring Data JPA 방식이다.

```
public interface UserRepository extends JpaRepository<User, Long> {
}
```

반면 다음과 같은 Mapper가 있다면 MyBatis를 사용하는 프로젝트일 수 있다.

```
@Mapper
public interface UserMapper {

    User findById(Long id);
}
```

XML 파일도 함께 있을 수 있다.

```
src/main/resources/mapper/UserMapper.xml
```

프로젝트를 볼 때는 `repository`라는 폴더 이름만 찾는 것이 아니라 실제 데이터 접근 기술과 쿼리가 어디에 있는지 확인해야 한다.

---

## 설정 파일에 민감한 값이 있는지 확인한다

설정 파일을 볼 때는 데이터베이스 비밀번호나 API Key가 직접 작성되어 있는지도 확인한다.

```
spring:
  datasource:
    password: real-password
```

이처럼 실제 비밀번호가 설정 파일에 평문으로 들어 있다면 Git 저장소, 로그, 배포 파일 등을 통해 노출될 수 있다.

일반적으로 민감한 값은 환경변수나 별도의 Secret 관리 시스템에서 주입하는 방식이 더 안전하다.

```
spring:
  datasource:
    password: ${DB_PASSWORD}
```

처음 프로젝트를 확인할 때 보안 문제를 바로 수정하지 않더라도, 어떤 값이 외부에서 주입되고 어떤 값이 저장소에 포함되어 있는지는 파악해두는 것이 좋다.

---

## 실행 방법도 함께 찾아본다

코드 구조만큼 중요한 것이 실행 방법이다.

다음 파일들이 있다면 함께 확인한다.

```
README.md
Dockerfile
docker-compose.yml
Makefile
Jenkinsfile
.github/workflows
```

이 파일들을 보면 다음 내용을 알 수 있다.

- 로컬 실행 방법
- 필요한 환경변수
- 데이터베이스 실행 방법
- Docker 사용 여부
- 테스트 실행 명령
- 배포 방식
- CI/CD 구성

예를 들어 `docker-compose.yml`에 PostgreSQL과 Redis가 정의되어 있다면, 애플리케이션을 실행하기 전에 해당 컨테이너들이 필요할 수 있다.

```
services:
  postgres:
    image: postgres

  redis:
    image: redis
```

코드를 읽었더라도 실제로 실행할 수 없다면 프로젝트를 완전히 이해하기 어렵다.

가능하다면 로컬에서 직접 실행하고 간단한 API를 호출해보는 것이 가장 빠른 확인 방법이다.

---

## 처음부터 모든 코드를 읽지 않아도 된다

처음 프로젝트를 볼 때 자주 하는 실수는 모든 파일을 처음부터 차례대로 읽으려는 것이다.

하지만 대부분의 프로젝트에는 당장 이해하지 않아도 되는 코드가 많다.

- 공통 유틸리티
- 오래된 기능
- 사용하지 않는 코드
- 특정 환경에서만 동작하는 설정
- 자동 생성된 코드
- 외부 라이브러리 설정
- 단순 DTO와 변환 코드

처음에는 핵심 기능 하나를 선택하는 것이 좋다.

예를 들어 회원 가입 기능을 본다면 다음처럼 범위를 좁힐 수 있다.

```
POST /users
→ UserController
→ UserService
→ UserRepository
→ User
→ UserServiceTest
```

이 흐름을 이해한 다음 다른 기능으로 범위를 넓히면 된다.

---

## 내가 실제로 사용하는 확인 순서

낯선 Spring Boot 프로젝트를 처음 받으면 나는 대체로 다음 순서대로 살펴본다.

### 1단계: 프로젝트의 기본 정보 확인

```
README.md
build.gradle 또는 pom.xml
settings.gradle
```

사용 기술, Java 버전, Spring Boot 버전, 모듈 구조를 확인한다.

### 2단계: 실행 구조 확인

```
@SpringBootApplication
application.yml
application-*.yml
Dockerfile
docker-compose.yml
```

실행 진입점과 필요한 외부 환경을 확인한다.

### 3단계: 패키지 지도 만들기

```
controller
service
repository
domain
config
common
infra
```

프로젝트가 레이어 기준인지 도메인 기준인지 확인한다.

### 4단계: 핵심 요청 하나 선택

```
Controller
→ Service
→ Repository
→ Database
```

실제 요청 하나를 처음부터 끝까지 따라간다.

### 5단계: 테스트와 예외 확인

```
src/test/java
GlobalExceptionHandler
ErrorCode
```

정상 동작과 실패 조건, 핵심 비즈니스 규칙을 확인한다.

### 6단계: 외부 시스템 확인

```
Redis
Kafka
외부 API
파일 저장소
메일
결제
```

프로젝트 외부와 연결되는 부분을 확인한다.

---

## 처음 프로젝트를 볼 때 사용할 수 있는 체크리스트

### 빌드와 실행

- Maven인가 Gradle인가?
- Java 버전은 무엇인가?
- Spring Boot 버전은 무엇인가?
- 실행 가능한 모듈은 무엇인가?
- main class는 어디에 있는가?
- 로컬 실행에 필요한 환경변수는 무엇인가?

### 프로젝트 구조

- 단일 모듈인가 멀티 모듈인가?
- 레이어 기준인가 도메인 기준인가?
- 핵심 비즈니스 로직은 어디에 있는가?
- 공통 코드와 기능 코드가 어떻게 구분되어 있는가?

### 요청 처리

- API 요청은 어느 Controller로 들어오는가?
- 어떤 Service를 호출하는가?
- 트랜잭션은 어디에서 시작되는가?
- 데이터베이스 접근은 어떤 기술을 사용하는가?
- 응답 DTO는 어디에서 만들어지는가?

### 외부 연결

- 데이터베이스는 무엇인가?
- Redis나 메시지 큐를 사용하는가?
- 외부 API를 호출하는가?
- 파일이나 이미지는 어디에 저장하는가?
- 인증은 어떤 방식으로 처리하는가?

### 테스트와 운영

- 단위 테스트와 통합 테스트가 있는가?
- 주요 비즈니스 규칙이 테스트되어 있는가?
- 로그는 어떻게 남기는가?
- 예외는 어떻게 처리하는가?
- 배포와 CI/CD는 어떻게 구성되어 있는가?

---

## 공식 문서에서는 어떻게 설명할까?

Spring Boot 공식 문서의 `Structuring Your Code`에서는 반드시 하나의 프로젝트 구조만 사용해야 한다고 강제하지는 않는다.

다만 main application class를 root package에 두는 방식을 권장한다. `@SpringBootApplication`이 붙은 클래스의 위치가 컴포넌트 탐색 범위에 영향을 주기 때문이다.

Maven의 `Standard Directory Layout`에서는 일반적인 Java 프로젝트 구조를 다음과 같이 설명한다.

```
src/main/java
src/main/resources
src/test/java
src/test/resources
```

Gradle의 Java Plugin도 기본적으로 비슷한 source directory 구조를 사용한다.

```
src/main/java
src/main/resources
src/test/java
src/test/resources
```

빌드 결과는 일반적으로 다음 위치에 생성된다.

```
build/
```

이러한 표준 구조를 따르면 새로운 개발자가 프로젝트를 더 쉽게 이해할 수 있고, 빌드 도구와 IDE도 별도 설정 없이 프로젝트 구조를 인식하기 쉽다.

결국 처음 프로젝트를 볼 때는 이 프로젝트가 표준 구조를 얼마나 따르고 있는지, 표준에서 벗어났다면 어떤 이유로 다르게 구성했는지를 확인하면 된다.

---

## 마무리

낯선 백엔드 프로젝트를 처음 볼 때 모든 파일을 한 번에 이해하려고 하면 어렵다.

대신 다음 순서로 접근하면 훨씬 수월하다.

1. 빌드 파일을 확인한다.
2. 모듈 구성을 확인한다.
3. 실행 진입점을 찾는다.
4. 설정 파일과 실행 환경을 확인한다.
5. 패키지가 어떤 기준으로 나뉘어 있는지 본다.
6. 요청이 들어오는 지점을 찾는다.
7. 요청 하나를 Service와 Repository까지 따라간다.
8. 테스트를 통해 성공 조건과 실패 조건을 확인한다.

프로젝트 구조를 이해한다는 것은 폴더 이름과 클래스 목록을 외우는 일이 아니다.

외부 요청 하나가 어디에서 시작되고, 어떤 비즈니스 로직을 거쳐, 어디에서 데이터를 조회하거나 변경하고, 어떤 결과로 반환되는지 지도를 그리는 일이다.

처음에는 폴더가 많아 보여도 괜찮다.

빌드, 실행, 설정, 요청 진입점, 비즈니스 흐름만 차례대로 잡으면 낯선 프로젝트도 조금씩 읽히기 시작한다.
