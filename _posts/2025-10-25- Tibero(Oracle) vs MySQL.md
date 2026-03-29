---
title: Tibero(Oracle) vs MySQL
author: leedohyun
date: 2025-10-25 18:13:00 -0500
categories: [회사, 정리]
tags: [Tibero, MySQL]
---

취준 당시 MySQL만 사용했었다. 하지만 취업을 하게 되고 실무에서는 Tibero DB를 사용하게 되었다.

단순히 DB 종류만 다른 것이 아니기때문에 어떤 부분들이 다른지 정리해보려고 한다.

## 쿼리 문법

### LIMIT vs ROWNUM

MySQL에서는 페이징이나 조회 개수 제한을 할 때 LIMIT을 사용했다.

```sql
SELECT * FROM orders ORDER BY created_at DESC
LIMIT 10;
```

하지만 Tibero에서는 LIMIT이 없고 ROWNUM을 사용한다.

```sql
SELECT * 
FROM (
	SELECT *
	FROM orders
	ORDER BY created_at DESC
)
WHERE ROWNUM <= 10;
```

페이징 공통 sql이 사내에서 사용되고 있어 이 쿼리를 직접 작성할 일은 없었지만 실행계획 분석 시 실행되었던 쿼리를 입력, 분석할 때 이러한 부분에서 차이를 발견했다.

ROWNUM은 정렬 이후가 아니라 먼저 적용되어 정렬 이후 ROWNUM이 적용되도록 한번 더 감싸주어야 됐다.

### IFNULL vs NVL

MySQL에서는 NULL 처리를 IFNULL로 했다.

```sql
SELECT IFNULL(name, 'empty')
FROM user
```

Tibero에서는 NVL을 사용한다.

```sql
SELECT NVL(name, 'empty')
FROM user;
```

이건 직관적으로 알 수 있었지만 생각보다 다른 부분들이 좀 있구나를 느꼈었다.

## 페이징 방식 차이

MySQL에서는 LIMIT offset, size 방식을 통해 페이징을 쉽게 처리했다.

하지만 Tibero에서는 ROWNUM이나 ROW_NUMBER()를 활용해야 한다.

```sql
SELECT *
FROM (
	SELECT t.*, ROW_NUMBER() OVER (ORDER BY created_at DESC) AS rn
	FROM orders tags
)
WHERE rn BETWEEN 20 AND 30;
```

마찬가지로 쿼리에서의 차이이다.

## 실행계획과 성능 차이

같은 쿼리라도 DB에 따라 실행계획이 다르게 나오는 경우가 있다.

MySQL에서는 인덱스를 잘 타던 쿼리가 Tibero에서는 Full Scan으로 동작하거나 반대로 Tibero에서는 괜찮던 쿼리가 MySQL에서는 비효율적으로 동작할 수 있다.

단순히 인덱스 문제라고 보일 수 있지만 DB마다 다른 옵티마이저 때문일 수 있다.

MySQL과 Tibero 모두 비용 기반으로 실행계획을 선택한다.

하지만 통계 정보를 해석하는 방식이나 접근 경로, 조인 방식 선택 기준이 다르기 때문에 같은 SQL이어도 다른 실행계획이 나올 수 있다.

따라서 쿼리 개선을 진행할 경우 MySQL에서 사용했던 개념들이 통할 확률이 높지만 다르게 동작할 수 있기때문에 실행계획을 확인하고 접근 방식을 조정하는 과정이 필요하다.

## 트랜잭션/락 동작 차이

MySQL과 Tibero 모두 MVCC 기반으로 동작하고 row-level locking을 사용한다. 그래서 기본적으로 읽기와 쓰기가 동시에 발생해도 어느정도 충돌없이 처리된다.

하지만 실제 동작을 보면 차이가 있을 수 있다고 한다.

MySQL은 next-key locking 같은 방식으로 인덱스 범위까지 함께 잠그는 경우가 있어, 조건에 따라 보다 넓은 범위에 락 영향이 퍼질 수 있다.

Tibero는 row-level locking을 기반으로 동작하면서 MVCC를 통해 읽기 일관성을 유지하는 구조를 가지고 있어, 동시성 처리 방식이 조금 다르게 체감될 수 있다.

둘 다 행 단위 락을 사용하지만 실제 락이 걸리는 범위나 대기 방식, 실행계획과의 결합 방식이 다를 수 있다.

따라서 단순히 트랜잭션을 사용하는 것에서 끝나는 것이 아닌, 사용하는 DB를 기준으로 락 동작과 실행계획을 함께 확인하는 과정이 필요할 수 있다.

## 정리

Tibero를 사용하면서 크게 불편하다고 느낀 부분은 없었지만,
툴 사용법이나 쿼리 작성 방식에서 MySQL과 다른 부분에 적응하는 과정이 필요했다.

특히 쿼리를 그대로 가져와 사용할 수 있는 수준이 아니라,
DB에 맞게 다시 해석하고 실행계획을 확인해야 한다는 점이 가장 크게 느껴졌다.

정리해보면서 MySQL의 문법이 직관적이고 단순하다는 것을 알았다.

물론 Tibero는 Oracle 계열과 유사한 구조를 가지고 있어 대규모 시스템이나 안정성이 중요한 환경에서는 장점이 있을 수 있지만 신입 개발자의 입장에서는 아직 MySQL이 더 익숙하고 다루기 쉬운 DB인 것 같다.

> Oracle이 대규모 시스템에서 더 좋은 이유

- 안정적인 트랜잭션 처리
	- 읽기와 쓰기가 동시에 일어나도 충돌을 최대한 줄이도록 설계되었다.
	- 동시성 상황에서 안정적으로 버티는 구조.
- 강력한 옵티마이저
	- 조인 순서, 방식 최적화가 강력하다.
	- 실행계획을 강제할 수 있다.
- 고급 기능 (파티셔닝, 병렬처리)
- 운영/튜닝 도구