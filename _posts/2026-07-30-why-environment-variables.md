---
title: "환경변수는 왜 사용하는 걸까?"
date: 2026-07-30 09:20:24 +0900
last_modified_at: 2026-10-01
categories: [Infrastructure, Configuration]
tags: [environment-variable, config, deployment]
source_url: "https://velog.io/@joker901010/%ED%99%98%EA%B2%BD%EB%B3%80%EC%88%98%EB%8A%94-%EC%99%9C-%EC%82%AC%EC%9A%A9%ED%95%98%EB%8A%94-%EA%B1%B8%EA%B9%8C"
---
## 환경변수를 처음 알았을 때의 오해

처음 환경변수를 접했을 때는 단순히 비밀번호를 숨기기 위한 기능이라고 생각했다.

DB 비밀번호나 API Key처럼 코드에 직접 적으면 위험한 값을 밖으로 빼놓는 용도라고만 이해했다.

물론 이것도 환경변수를 사용하는 중요한 이유다. 하지만 실제로 배포 환경을 구성해보니 환경변수의 역할은 그것보다 조금 더 넓었다.

환경변수는 **환경마다 달라지는 값을 코드와 분리하기 위해 사용하는 설정값**이다.

DB 주소, 서버 포트, 로그 레벨, 외부 API 주소처럼 개발 환경과 운영 환경에서 달라질 수 있는 값을 코드 밖에서 전달하는 것이다.

## 같은 코드로 서로 다른 환경에서 실행하기

개발 서버와 운영 서버는 사용하는 설정이 다르다.

예를 들어 개발 환경에서는 로컬에 설치된 DB를 사용할 수 있다.

```
jdbc:postgresql://localhost:5432/app_dev
```

운영 환경에서는 별도의 운영 DB에 연결해야 한다.

```
jdbc:postgresql://prod-db:5432/app_prod
```

이 값을 코드에 직접 작성하면 다음과 같은 문제가 생긴다.

```
String dbUrl = "jdbc:postgresql://localhost:5432/app_dev";
```

운영 서버에 배포할 때마다 코드를 수정해야 하고, 실수로 개발 DB 주소를 그대로 배포할 수도 있다.

설정이 바뀔 때마다 코드를 다시 수정하고 빌드해야 한다는 점도 불편하다.

환경변수를 사용하면 애플리케이션 코드는 그대로 두고, 실행되는 환경에서 값만 다르게 전달할 수 있다.

개발 환경에서는 다음 값을 전달한다.

```
DB_URL=jdbc:postgresql://localhost:5432/app_dev
```

운영 환경에서는 다른 값을 전달한다.

```
DB_URL=jdbc:postgresql://prod-db:5432/app_prod
```

결국 같은 애플리케이션도 어떤 환경변수를 받느냐에 따라 개발 서버가 될 수도 있고 운영 서버가 될 수도 있다.

```
같은 애플리케이션 코드
+ 개발 환경 설정
= 개발 서버

같은 애플리케이션 코드
+ 운영 환경 설정
= 운영 서버
```

이 구조의 장점은 코드가 특정 실행 환경에 종속되지 않는다는 것이다.

## 환경변수는 애플리케이션에 전달하는 설정값이다

환경변수는 애플리케이션이 직접 가지고 있는 값이라기보다, 실행 환경이 애플리케이션에 전달해주는 값에 가깝다.

예를 들어 여행 가방은 같아도 목적지에 따라 붙이는 수하물 태그는 달라진다.

애플리케이션 코드가 여행 가방이라면 환경변수는 목적지를 알려주는 태그라고 볼 수 있다.

개발 환경에서는 다음과 같은 설정이 필요할 수 있다.

```
개발 DB
테스트 결제 API
DEBUG 로그
로컬 파일 저장 경로
```

운영 환경에서는 설정이 달라진다.

```
운영 DB
실제 결제 API
INFO 로그
운영 파일 저장 경로
```

환경이 바뀐다고 애플리케이션 코드를 새로 만들지는 않는다.

같은 코드를 사용하되, 실행할 때 필요한 설정만 다르게 전달한다.

이렇게 생각하면 환경변수는 단순히 값을 숨겨놓는 공간이 아니다.

**실행 환경이 애플리케이션에게 필요한 설정을 전달하는 방법**이다.

## Spring Boot에서는 어떻게 사용할까?

Spring Boot에서는 설정값을 `application.yml` 또는 `application.properties`에 작성할 수 있다.

예를 들어 DB 연결 정보를 `application.yml`에 다음과 같이 작성할 수 있다.

```
spring:
  datasource:
    url: ${DB_URL}
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
```

`${DB_URL}`은 실행 환경에서 `DB_URL`이라는 값을 찾아 사용하겠다는 의미다.

개발 환경에서는 다음과 같이 설정할 수 있다.

```
DB_URL=jdbc:postgresql://localhost:5432/app_dev
DB_USERNAME=dev_user
DB_PASSWORD=dev_password
```

운영 환경에서는 다른 값을 전달한다.

```
DB_URL=jdbc:postgresql://prod-db:5432/app_prod
DB_USERNAME=prod_user
DB_PASSWORD=very-secret-password
```

애플리케이션 코드는 변경되지 않는다.

비즈니스 로직을 담당하는 서비스 코드도 DB가 개발 DB인지 운영 DB인지 알 필요가 없다.

```
@Service
public class UserService {

    public UserResponse getUser(Long id) {
        return new UserResponse(id, "junior-backend");
    }
}
```

애플리케이션이 어느 DB에 연결되는지는 서비스 코드가 아니라 실행 환경에서 전달한 설정값이 결정한다.

비즈니스 로직과 실행 환경의 설정이 서로 분리되는 것이다.

## Spring Boot 설정과 환경변수 이름

Spring Boot의 설정 이름은 보통 점으로 구분한다.

```
spring.datasource.url
spring.datasource.username
spring.datasource.password
```

하지만 운영체제의 환경변수 이름에는 점을 사용하기 어렵거나 제한이 있을 수 있다.

그래서 환경변수에서는 일반적으로 점을 밑줄로 바꾸고 대문자로 작성한다.

```
SPRING_DATASOURCE_URL
SPRING_DATASOURCE_USERNAME
SPRING_DATASOURCE_PASSWORD
```

Spring Boot는 이러한 이름을 기존 설정 이름과 연결해서 읽을 수 있다.

예를 들어 다음 환경변수는

```
SERVER_PORT=8081
```

Spring Boot의 다음 설정과 연결된다.

```
server.port=8081
```

따라서 서버 포트를 바꾸기 위해 코드를 수정하거나 설정 파일을 다시 만들 필요 없이, 실행할 때 환경변수만 전달할 수 있다.

```
SERVER_PORT=8081 java -jar app.jar
```

실제 환경변수를 전달하는 방법은 운영체제나 Docker, Kubernetes, Jenkins 같은 배포 환경에 따라 달라질 수 있다.

## 어떤 값을 환경변수로 빼야 할까?

환경변수로 관리하기 좋은 값은 크게 두 종류다.

첫 번째는 환경마다 달라지는 값이다.

두 번째는 코드나 Git 저장소에 포함되면 위험한 값이다.

실무에서는 보통 다음과 같은 값을 환경변수로 관리한다.

```
DB URL
DB username
DB password
Redis host
외부 API URL
외부 API Key
JWT secret
메일 서버 계정
파일 저장 경로
로그 레벨
서버 포트
활성 profile
```

예를 들어 결제 API 주소는 개발 환경과 운영 환경이 다를 수 있다.

개발 환경에서는 테스트용 API를 사용한다.

```
PAYMENT_API_URL=https://test-payment.example.com
```

운영 환경에서는 실제 결제 API를 사용한다.

```
PAYMENT_API_URL=https://payment.example.com
```

JWT 서명에 사용하는 비밀 키도 코드에 직접 작성해서는 안 된다.

```
JWT_SECRET=some-very-secret-key
```

이런 값이 Git 저장소에 올라가면 저장소에 접근할 수 있는 사람이 비밀 값을 확인할 수 있다.

공개 저장소라면 외부에 그대로 노출될 수도 있다.

## 환경변수를 사용한다고 무조건 안전한 것은 아니다

환경변수를 사용하면 민감한 값이 코드와 Git 저장소에 포함되는 것을 막을 수 있다.

하지만 환경변수로 옮겼다고 해서 그 값이 완전히 안전해지는 것은 아니다.

서버에 접근 권한이 있는 사람은 환경변수를 확인할 수 있다. 애플리케이션이 환경변수를 로그에 출력하면 로그를 통해 값이 노출될 수도 있다.

운영 환경에서는 환경변수와 함께 별도의 Secret 관리 방식을 사용하는 경우가 많다.

예를 들면 다음과 같다.

```
Kubernetes Secret
Docker Secret
AWS Secrets Manager
Google Secret Manager
Azure Key Vault
HashiCorp Vault
```

이런 도구들은 민감한 값을 저장하고, 필요한 애플리케이션에만 전달하고, 접근 권한을 관리하는 역할을 한다.

환경변수는 비밀 값을 전달하는 방법이 될 수 있지만, 비밀 값 자체를 안전하게 관리하는 모든 문제를 해결해주는 것은 아니다.

## 환경변수 이름은 일관되게 관리하기

같은 의미의 값을 프로젝트마다 다른 이름으로 사용하면 관리하기 어려워진다.

예를 들어 DB 주소를 다음처럼 제각각 작성할 수 있다.

```
DB_URL
DATABASE_URL
SPRING_DATASOURCE_URL
```

모두 비슷한 의미지만 이름이 다르면 새로운 개발자가 프로젝트를 실행할 때 헷갈릴 수 있다.

배포 설정을 작성할 때도 어떤 값을 전달해야 하는지 확인하는 데 시간이 걸린다.

팀에서는 환경변수 이름을 어떤 방식으로 작성할지 규칙을 정하는 것이 좋다.

Spring Boot의 설정 이름을 그대로 변환해서 사용할 수도 있고,

```
SPRING_DATASOURCE_URL
SPRING_DATASOURCE_USERNAME
SPRING_DATASOURCE_PASSWORD
```

프로젝트에서 의미를 이해하기 쉬운 별도의 이름을 사용할 수도 있다.

```
DB_URL
DB_USERNAME
DB_PASSWORD
```

어떤 방식을 사용하든 한 프로젝트 안에서는 이름을 일관되게 유지하는 것이 중요하다.

## 필요한 환경변수는 문서로 남겨야 한다

환경변수를 코드 밖으로 빼면 코드만 받아서는 애플리케이션을 실행할 수 없는 경우가 생긴다.

새로운 개발자가 프로젝트를 실행하려고 했는데 어떤 환경변수가 필요한지 모르면 실행 단계부터 막히게 된다.

따라서 README나 별도의 설정 문서에 필요한 환경변수를 정리해두는 것이 좋다.

```
필수 환경변수

DB_URL=jdbc:postgresql://localhost:5432/app_dev
DB_USERNAME=dev_user
DB_PASSWORD=dev_password
JWT_SECRET=local-dev-secret
```

여기에는 실제 운영 값을 작성하면 안 된다.

로컬에서 사용할 수 있는 예시 값이나 값의 형식만 작성해야 한다.

필요하다면 `.env.example` 파일을 만들어 관리할 수도 있다.

```
DB_URL=
DB_USERNAME=
DB_PASSWORD=
JWT_SECRET=
```

실제 값을 가진 `.env` 파일은 Git에 올리지 않고, 필요한 환경변수 목록만 `.env.example`을 통해 공유하는 방식이다.

## 환경변수가 없을 때의 동작도 정해야 한다

환경변수를 사용할 때는 해당 값이 없으면 애플리케이션이 어떻게 동작해야 하는지도 생각해야 한다.

반드시 필요한 값이라면 애플리케이션 실행이 실패하도록 하는 편이 안전하다.

서버 포트처럼 기본값을 사용해도 괜찮은 설정에는 기본값을 줄 수 있다.

```
server:
  port: ${SERVER_PORT:8080}
```

이 설정은 `SERVER_PORT`가 존재하면 해당 값을 사용하고, 값이 없으면 `8080`을 사용한다.

하지만 DB 비밀번호, JWT Secret, 외부 서비스 인증 정보처럼 중요한 값에는 기본값을 함부로 지정하면 안 된다.

예를 들어 다음과 같이 기본 비밀번호를 지정할 수는 있다.

```
spring:
  datasource:
    password: ${DB_PASSWORD:password}
```

하지만 운영 환경에서 `DB_PASSWORD`를 빠뜨렸는데 애플리케이션이 기본값으로 실행되어 버리면 문제를 발견하기 어려울 수 있다.

중요한 설정이라면 값이 없을 때 바로 실행에 실패하도록 두는 편이 낫다.

## 민감한 값은 로그에 출력하지 않기

환경변수와 관련해 가장 조심해야 하는 부분 중 하나가 로그다.

문제를 확인하기 위해 설정값을 출력했다가 비밀번호나 인증 정보가 그대로 남을 수 있다.

특히 다음 값은 로그에 직접 출력하지 않는 것이 좋다.

```
DB_PASSWORD
JWT_SECRET
API_KEY
ACCESS_TOKEN
REFRESH_TOKEN
```

설정이 정상적으로 전달됐는지 확인해야 한다면 값 자체보다는 값의 존재 여부만 확인하는 방식이 낫다.

```
if (jwtSecret == null || jwtSecret.isBlank()) {
    throw new IllegalStateException("JWT_SECRET 환경변수가 필요합니다.");
}
```

로그가 필요하다면 민감한 부분을 가리는 방법도 있다.

```
API_KEY=abcd****
```

다만 가능하면 민감한 값은 애초에 로그에 남기지 않는 것이 가장 안전하다.

## 설정 파일과 환경변수는 무엇이 다를까?

모든 설정을 반드시 환경변수로 관리해야 하는 것은 아니다.

환경에 따라 거의 바뀌지 않고, 공개되어도 문제가 없는 값은 설정 파일에 작성할 수 있다.

예를 들어 다음과 같은 값은 설정 파일에 둘 수 있다.

```
spring:
  jackson:
    time-zone: Asia/Seoul

logging:
  pattern:
    console: "%d{yyyy-MM-dd HH:mm:ss} %msg%n"
```

반대로 환경마다 달라지거나 민감한 값은 환경변수로 분리하는 것이 좋다.

```
spring:
  datasource:
    url: ${DB_URL}
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
```

결국 중요한 것은 무조건 환경변수를 사용하는 것이 아니라, 값의 성격에 따라 관리 위치를 정하는 것이다.

## 공식 문서에서는 어떻게 설명할까?

Spring Boot는 애플리케이션 설정을 코드 외부에서 관리할 수 있도록 다양한 설정 방식을 제공한다.

대표적으로 다음과 같은 방법이 있다.

```
properties 파일
YAML 파일
환경변수
Java System Property
Command Line Argument
```

같은 애플리케이션 코드라도 어떤 설정값을 전달하느냐에 따라 서로 다른 환경에서 실행할 수 있다.

The Twelve-Factor App에서도 설정을 코드와 분리하고, 배포 환경에서 관리하는 방식을 권장한다.

여기서 말하는 설정은 단순히 비밀번호만 의미하지 않는다.

DB 주소, 외부 서비스 인증 정보, 호스트 이름, 캐시 서버 주소처럼 배포 환경마다 달라질 수 있는 값을 포함한다.

공통된 핵심은 하나다.

**코드와 실행 환경의 설정을 분리하는 것**이다.

## 환경변수를 이해하고 나서 달라진 점

처음에는 환경변수를 비밀번호를 숨기는 용도로만 생각했다.

하지만 실제 배포 구조와 Spring Boot의 설정 방식을 함께 살펴보니, 환경변수의 더 중요한 역할은 코드와 실행 환경을 분리하는 데 있다는 것을 알게 됐다.

환경변수를 사용하면 같은 코드를 개발, 테스트, 스테이징, 운영 환경에서 그대로 사용할 수 있다.

환경이 달라져도 코드를 수정할 필요가 없고, 배포할 때 필요한 설정만 다르게 전달하면 된다.

그래서 지금은 환경변수를 다음과 같이 이해하고 있다.

> 환경변수는 코드가 아니라 실행 환경이 애플리케이션에 건네주는 설정 쪽지다.

새로운 설정값을 추가할 때는 다음과 같은 질문을 해볼 수 있다.

```
이 값은 환경마다 달라지는가?
이 값이 Git에 올라가도 안전한가?
운영 중에 변경될 가능성이 있는가?
이 값이 없으면 애플리케이션 실행이 실패해야 하는가?
```

이 질문에 답하다 보면 어떤 값을 코드에 둘지, 설정 파일에 둘지, 환경변수로 분리할지 조금씩 판단할 수 있다.

## 정리

환경변수는 애플리케이션이 실행되는 환경에서 전달받는 설정값이다.

환경변수를 사용하는 이유는 크게 세 가지다.

1. 환경마다 달라지는 값을 코드와 분리하기 위해
2. 비밀번호나 API Key 같은 민감한 값을 Git 저장소에 올리지 않기 위해
3. 같은 애플리케이션 코드를 여러 환경에서 다르게 실행하기 위해

Spring Boot에서는 환경변수를 외부 설정 소스 중 하나로 읽는다.

예를 들어 `SPRING_DATASOURCE_URL` 환경변수를 `spring.datasource.url` 설정과 연결할 수 있다.

다만 환경변수만 사용한다고 보안 문제가 모두 해결되는 것은 아니다. 민감한 값은 로그에 출력하지 않아야 하고, 운영 환경에서는 Secret 관리 도구와 접근 권한 관리도 함께 고려해야 한다.

결국 환경변수의 핵심은 값을 숨기는 데만 있지 않다.

**애플리케이션 코드와 실행 환경을 분리해, 같은 코드를 여러 환경에서 안정적으로 실행할 수 있게 만드는 것**에 있다.
