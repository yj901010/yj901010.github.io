---
title: "Java 객체지향을 다시 정리하며: 오버로딩, 오버라이딩, 다형성"
date: 2026-10-01 11:40:00 +0900
categories: [Backend, Java]
tags: [java, oop, polymorphism, overloading, overriding]
description: 과거 Java 기초 수업 노트 여러 편을 하나로 합쳐 객체지향의 핵심 개념을 다시 정리합니다.
---

> Tistory의 Java 기초 학습 글 중 오버로딩, 오버라이딩, 객체 형변환 관련 내용을 하나로 통합했다.

Java를 처음 배울 때는 각각의 용어를 따로 외웠다.

- Overloading
- Overriding
- Up Casting
- Down Casting

하지만 실제로는 이 개념들이 타입, 상속, 다형성이라는 하나의 흐름으로 연결된다.

## 오버로딩

오버로딩은 같은 클래스 안에서 같은 이름의 메서드를 여러 개 정의하는 것이다.

단, 파라미터 목록이 달라야 한다.

~~~java
public class Calculator {

    public int add(int a, int b) {
        return a + b;
    }

    public long add(long a, long b) {
        return a + b;
    }

    public int add(int a, int b, int c) {
        return a + b + c;
    }
}
~~~

메서드 이름은 같지만 호출 시점에 전달한 인자에 따라 어떤 메서드를 사용할지 결정된다.

반환 타입만 다르게 만드는 것은 오버로딩이 아니다.

## 오버라이딩

오버라이딩은 상위 타입에서 정의한 메서드를 하위 클래스에서 다시 구현하는 것이다.

~~~java
public class Animal {

    public void sound() {
        System.out.println("animal");
    }
}

public class Dog extends Animal {

    @Override
    public void sound() {
        System.out.println("bow-wow");
    }
}
~~~

오버라이딩의 핵심은 런타임 다형성과 연결된다는 점이다.

~~~java
Animal animal = new Dog();
animal.sound();
~~~

변수 타입은 Animal이지만 실제 객체는 Dog이므로 Dog의 sound()가 호출된다.

## 업캐스팅

하위 타입의 객체를 상위 타입 참조변수로 다루는 것이다.

~~~java
Animal animal = new Dog();
~~~

이 방식 덕분에 여러 구현체를 하나의 상위 타입으로 처리할 수 있다.

~~~java
List<Animal> animals = List.of(
    new Dog(),
    new Cat()
);

for (Animal animal : animals) {
    animal.sound();
}
~~~

호출하는 코드는 Dog와 Cat의 구체 구현을 몰라도 된다.

## 다운캐스팅

상위 타입 참조를 다시 구체 하위 타입으로 변환하는 것이다.

~~~java
if (animal instanceof Dog dog) {
    dog.fetch();
}
~~~

다운캐스팅이 자주 필요하다면 상위 타입의 추상화가 충분한지 다시 살펴볼 필요가 있다.

호출자가 계속 구체 타입을 확인해야 한다면 다형성의 장점을 제대로 활용하지 못하고 있을 수 있다.

## 인터페이스와 다형성

실제 Spring 애플리케이션에서는 클래스 상속보다 인터페이스를 통한 다형성을 더 자주 접한다.

~~~java
public interface PaymentClient {
    PaymentResult pay(PaymentCommand command);
}
~~~

~~~java
@Component
public class KakaoPaymentClient implements PaymentClient {
    // ...
}
~~~

~~~java
@Component
public class TossPaymentClient implements PaymentClient {
    // ...
}
~~~

Service는 구체 구현보다 인터페이스에 의존할 수 있다.

~~~java
@Service
public class PaymentService {

    private final PaymentClient paymentClient;

    public PaymentService(PaymentClient paymentClient) {
        this.paymentClient = paymentClient;
    }
}
~~~

이 지점에서 객체지향 기초와 Spring의 의존성 주입이 연결된다.

## 예전에는 문법으로 봤고, 지금은 의존성으로 본다

처음에는 다음처럼 기억했다.

- 오버로딩: 같은 이름, 다른 파라미터
- 오버라이딩: 상속받은 메서드 재정의
- 업캐스팅: 자식을 부모 타입으로
- 다운캐스팅: 부모 타입을 자식 타입으로

지금은 이 개념의 목적을 더 중요하게 본다.

구체 구현을 직접 알고 있는 코드를 줄이고, 공통된 추상화에 의존하도록 만들기 위해 다형성을 사용한다.

문법 자체보다 변화가 생겼을 때 어떤 코드까지 영향을 받는지를 생각하면 객체지향 개념이 실제 설계와 연결된다.
