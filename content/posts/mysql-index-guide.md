+++
date = '2026-05-11T22:24:17+09:00'
draft = false
title = 'Mysql Index Guide'
categories = ["Database"]
tags = ["mysql", "database", "index", "performance", "backend"]
+++

## 들어가며

쿼리가 느릴 때 가장 먼저 보게 되는 게 인덱스입니다.  
그런데 인덱스를 무작정 추가하다 보면 오히려 성능이 떨어지거나, 저장공간만 낭비하는 경우가 생깁니다.

이 포스팅에서는 **"왜 이렇게 만들어야 하는지"** 이유와 함께, 실전에서 바로 쓸 수 있는 인덱스 설계 전략을 정리했습니다.

---

## 1. 인덱스가 뭔지 다시 짚고 가기

MySQL InnoDB 기준으로, 인덱스는 **B+Tree** 자료구조로 관리됩니다.

- **리프 노드**에 실제 데이터(또는 PK)가 저장됨
- **클러스터형 인덱스(Clustered Index)**: PK 기준으로 물리적으로 정렬된 테이블 자체
- **보조 인덱스(Secondary Index)**: 리프 노드에 PK 값을 저장하고, PK로 다시 조회함

> 보조 인덱스로 조회하면 인덱스 → PK → 데이터 순으로 두 번 조회가 일어납니다. 이를 **"더블 룩업"** 이라고 합니다.

---

## 2. 인덱스를 언제 만들어야 할까

무조건 만든다고 좋은 게 아닙니다. 인덱스는 **읽기 성능을 올리는 대신 쓰기 성능을 낮추는** 트레이드오프입니다.

### 만들면 좋은 경우

- `WHERE` 절에 자주 등장하는 컬럼
- `JOIN` 의 연결 컬럼 (특히 FK)
- `ORDER BY`, `GROUP BY` 에 쓰이는 컬럼
- 선택도(Selectivity)가 높은 컬럼 (값의 종류가 많을수록 효과적)

### 만들지 않는 게 나은 경우

- `INSERT`, `UPDATE`, `DELETE` 가 매우 빈번한 테이블
- 데이터 수가 적은 테이블 (풀 스캔이 더 빠를 수 있음)
- 선택도가 낮은 컬럼 (예: `is_deleted TINYINT`, `status ENUM(...)` 단독 인덱스)
- 거의 조회되지 않는 컬럼

---

## 3. 복합 인덱스 설계 - 핵심은 컬럼 순서

복합 인덱스는 **컬럼 순서**가 전부입니다.

```sql
-- 예시 테이블
CREATE TABLE orders (
    id         BIGINT PRIMARY KEY,
    user_id    BIGINT NOT NULL,
    status     VARCHAR(20) NOT NULL,
    created_at DATETIME NOT NULL
);
```

### 원칙: 선택도 높은 컬럼을 앞에

```sql
-- Good: user_id가 status보다 선택도가 높다
CREATE INDEX idx_orders_user_status ON orders (user_id, status);

-- Bad: status 단독으로 앞에 오면 범위가 너무 넓다
CREATE INDEX idx_orders_status_user ON orders (status, user_id);
```

### 원칙: 등호(=) 조건을 범위(<, >, BETWEEN) 조건보다 앞에

B+Tree는 **첫 번째 범위 조건 이후의 컬럼은 인덱스 범위 탐색에 사용되지 않습니다.**

```sql
-- 이 쿼리에서
SELECT * FROM orders
WHERE user_id = 1
  AND created_at >= '2026-01-01'
  AND status = 'PAID';

-- 인덱스 순서는 이렇게 해야 한다
CREATE INDEX idx ON orders (user_id, status, created_at);
-- 등호 조건(user_id, status) → 범위 조건(created_at) 순
```

### 원칙: ORDER BY 컬럼을 인덱스에 포함시켜라

```sql
-- created_at DESC 정렬을 인덱스로 처리
CREATE INDEX idx ON orders (user_id, created_at DESC);

-- 이 쿼리는 filesort 없이 인덱스만으로 정렬 가능
SELECT * FROM orders
WHERE user_id = 1
ORDER BY created_at DESC
LIMIT 20;
```

---

## 4. 커버링 인덱스 (Covering Index)

쿼리에 필요한 컬럼을 인덱스가 모두 포함하고 있으면, **테이블 접근 없이 인덱스만으로 결과를 반환**할 수 있습니다. 이를 커버링 인덱스라고 합니다.

```sql
-- 인덱스에 조회 컬럼까지 포함
CREATE INDEX idx_orders_covering
    ON orders (user_id, status, created_at);

-- 아래 쿼리는 인덱스만 읽고 끝 (Extra: Using index)
SELECT status, created_at
FROM orders
WHERE user_id = 1;
```

`EXPLAIN` 결과의 `Extra` 컬럼에 **`Using index`** 가 뜨면 커버링 인덱스가 적용된 것입니다.

> 단, 인덱스에 컬럼을 너무 많이 넣으면 인덱스 크기가 커져서 역효과가 납니다. 실제로 쿼리에서 사용하는 컬럼만 포함하는 것이 좋습니다.

---

## 5. 인덱스가 안 타는 흔한 패턴

아무리 잘 만들어도 쿼리를 잘못 짜면 인덱스를 무시합니다.

### 컬럼에 함수/연산 적용

```sql
-- Bad: 인덱스 안 탐
WHERE DATE(created_at) = '2026-05-11'
WHERE YEAR(created_at) = 2026
WHERE user_id + 0 = 100

-- Good
WHERE created_at >= '2026-05-11 00:00:00'
  AND created_at <  '2026-05-12 00:00:00'
WHERE user_id = 100
```

### 묵시적 타입 변환

```sql
-- user_id가 BIGINT인데 문자열로 비교하면 인덱스 무시
WHERE user_id = '100'  -- Bad

-- Good
WHERE user_id = 100
```

### LIKE 앞부분 와일드카드

```sql
WHERE name LIKE '%수민'   -- Bad: 풀 스캔
WHERE name LIKE '수민%'   -- Good: 인덱스 탐
```

### OR 조건

```sql
-- OR은 인덱스를 두 번 타거나 아예 안 탈 수 있습니다
WHERE user_id = 1 OR status = 'PAID'

-- 가능하면 UNION ALL로 분리하거나 별도 인덱스를 검토하세요
```

### NULL 비교

```sql
-- IS NULL / IS NOT NULL도 상황에 따라 인덱스를 안 탑니다
-- NULL이 많은 컬럼에 인덱스를 걸면 효과가 적습니다
```

---

## 6. EXPLAIN으로 인덱스 확인하기

인덱스를 만들었으면 반드시 `EXPLAIN`으로 확인해야 합니다.

```sql
EXPLAIN SELECT *
FROM orders
WHERE user_id = 1
  AND status = 'PAID'
ORDER BY created_at DESC
LIMIT 20;
```

### 주요 컬럼 체크포인트

| 컬럼 | 확인 포인트 |
|------|------------|
| `type` | `ALL` 이면 풀 스캔. `ref`, `range`, `index` 가 나와야 함 |
| `key` | 실제로 사용된 인덱스 이름 |
| `rows` | 예상 스캔 행 수. 적을수록 좋음 |
| `Extra` | `Using filesort`, `Using temporary` 는 성능 경고 신호 |

### type 등급 (빠른 순)

```
system > const > eq_ref > ref > range > index > ALL
```

`ALL` 이 뜨면 인덱스 점검이 필요합니다.

---

## 7. 실전 설계 예시

### 페이지네이션 쿼리 최적화

```sql
-- Bad: OFFSET이 커질수록 느려진다
SELECT * FROM orders
WHERE user_id = 1
ORDER BY id DESC
LIMIT 20 OFFSET 10000;

-- Good: 커서 기반 페이지네이션
SELECT * FROM orders
WHERE user_id = 1
  AND id < :lastId        -- 이전 페이지 마지막 id
ORDER BY id DESC
LIMIT 20;

-- 인덱스
CREATE INDEX idx_orders_user_id ON orders (user_id, id);
```

### 기간 조회 + 상태 필터

```sql
-- 요구사항: 특정 기간 내 PAID 주문 목록
SELECT id, user_id, created_at
FROM orders
WHERE created_at BETWEEN '2026-05-01' AND '2026-05-31'
  AND status = 'PAID';

-- 인덱스 전략: 선택도가 높은 컬럼 기준으로
-- status는 값 종류가 적으므로 created_at을 앞에
CREATE INDEX idx_orders_created_status
    ON orders (created_at, status);

-- 커버링까지 노린다면
CREATE INDEX idx_orders_created_status_covering
    ON orders (created_at, status, id, user_id);
```

---

## 8. 인덱스 관리 팁

### 사용하지 않는 인덱스 찾기

```sql
-- performance_schema 활용 (MySQL 5.7+)
SELECT object_schema, object_name, index_name,
       count_star AS total_reads
FROM performance_schema.table_io_waits_summary_by_index_usage
WHERE object_schema = 'your_db'
  AND count_star = 0
  AND index_name IS NOT NULL
ORDER BY object_name;
```

### 중복 인덱스 주의

```sql
-- 이미 (a, b) 인덱스가 있으면 (a) 인덱스는 중복
CREATE INDEX idx1 ON t (a);
CREATE INDEX idx2 ON t (a, b);  -- idx1은 idx2로 커버됨 → idx1 삭제 가능
```

### 온라인 DDL로 안전하게 추가

MySQL 5.6+, InnoDB 기준으로 인덱스 추가는 기본적으로 온라인으로 처리되지만, 트래픽이 많은 운영 환경에서는 `pt-online-schema-change` 나 `gh-ost` 사용을 권장합니다.

---

## 마무리 체크리스트

인덱스를 만들기 전에 아래를 확인해 보세요.

- [ ] `EXPLAIN` 으로 실행계획 먼저 확인했는가?
- [ ] 복합 인덱스라면 컬럼 순서가 등호 → 범위 순인가?
- [ ] 선택도가 높은 컬럼이 앞에 오는가?
- [ ] 커버링 인덱스를 적용할 수 있는가?
- [ ] 함수/타입변환으로 인덱스가 무력화되지 않는가?
- [ ] 기존에 유사한 인덱스가 없는가? (중복 인덱스 체크)
- [ ] 쓰기 빈도 대비 읽기 이득이 충분한가?

인덱스는 **실행계획을 보고, 측정하고, 판단하는** 과정이 전부입니다.  
감으로 만들지 말고 항상 `EXPLAIN` 을 먼저 확인하세요.
