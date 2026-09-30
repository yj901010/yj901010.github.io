---
title: "[샘플] MySQL Index와 조회 성능"
date: 2026-09-30 14:30:00 +0900
categories: [Database, MySQL]
tags: [mysql, index, performance]
description: 인덱스를 적용할 때 어떤 관점으로 조회 성능을 분석할지 정리한 샘플 글입니다.
pin: true
---

> 이 글은 블로그 디자인 확인을 위한 샘플 글입니다. 실제 이전 작업 전에 삭제하거나 교체합니다.

## 문제 상황

데이터가 늘어나면서 특정 조회 쿼리의 응답 시간이 길어졌다고 가정합니다.

## 확인 순서

1. 실행 계획을 확인합니다.
2. 조건절과 정렬 조건을 확인합니다.
3. 후보 인덱스를 검토합니다.
4. 인덱스 적용 전후의 실행 계획과 응답 시간을 비교합니다.

~~~sql
EXPLAIN
SELECT *
FROM items
WHERE status = 'FOUND'
ORDER BY created_at DESC;
~~~

## 회고

새 기술을 도입하기 전에 현재 저장소에서 해결 가능한 범위를 먼저 검증하는 것이 중요합니다.
