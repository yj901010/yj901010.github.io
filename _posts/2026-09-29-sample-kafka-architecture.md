---
title: "[샘플] Kafka 기반 데이터 수집 구조"
date: 2026-09-29 20:00:00 +0900
categories: [Architecture, Messaging]
tags: [kafka, backend, architecture]
description: 외부 API 수집과 처리 단계를 분리하는 샘플 아키텍처 글입니다.
pin: true
mermaid: true
---

> 이 글은 블로그 디자인 확인을 위한 샘플 글입니다.

## 목표

외부 API 호출과 내부 데이터 처리 단계를 분리해 장애 전파를 줄이는 구조를 생각해봅니다.

~~~mermaid
flowchart LR
    A[External API] --> B[Collector]
    B --> C[Kafka]
    C --> D[Processor]
    D --> E[(Database)]
~~~

## 핵심 포인트

수집 속도와 처리 속도를 분리하고, 소비자 측 동시성과 재처리 정책을 별도로 관리할 수 있습니다.
