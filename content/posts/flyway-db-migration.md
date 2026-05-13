+++
date = '2026-05-13T23:13:23+09:00'
draft = false
title = 'Flyway로 DB 스키마 버전 관리하기'
categories = ["개발"]
tags = ["Spring Boot", "Flyway", "MySQL", "Database", "Migration"]
+++

## Flyway란?

Flyway는 **데이터베이스 마이그레이션 관리 도구**입니다.

개발을 하다 보면 테이블 컬럼 추가, 인덱스 생성, 초기 데이터 삽입 등 DB 스키마가 지속적으로 변경됩니다. 이런 변경사항을 팀원들이 각자 수동으로 적용하면 누가 어떤 변경을 적용했는지 추적이 어렵고, 환경마다 DB 상태가 달라지는 문제가 생깁니다.

Flyway는 이런 문제를 SQL 파일로 해결합니다. 변경사항을 파일로 관리하고 **자동으로, 순서대로, 한 번만** 실행해 줍니다.

---

## 핵심 개념

### 마이그레이션 파일

Flyway는 파일 이름의 규칙을 따릅니다.

```
V1__create_user_table.sql
V2__add_email_column.sql
V3__create_order_table.sql
```

- `V` + 버전 번호 + `__` (언더바 2개) + 설명 + `.sql`
- 애플리케이션 시작 시 Flyway가 **아직 실행되지 않은 파일만** 순서대로 실행합니다.

### flyway_schema_history

Flyway는 DB에 `flyway_schema_history` 테이블을 자동으로 생성하여 어떤 버전까지 적용되었는지 기록합니다.

| version | description | script | success |
|---|---|---|---|
| 1 | create user table | V1__create_user_table.sql | true |
| 2 | add email column | V2__add_email_column.sql | true |

---

## Spring Boot + Flyway + MySQL 적용

### 1. 의존성 추가

```kotlin
// build.gradle.kts
implementation("org.flywaydb:flyway-core")
implementation("org.flywaydb:flyway-mysql")
```

MySQL/MariaDB를 사용한다면 `flyway-mysql`도 함께 추가해야 합니다.

### 2. application.yml 설정

```yaml
spring:
  flyway:
    enabled: true
    locations: classpath:db/migration  # 기본값이므로 생략 가능
```

### 3. 마이그레이션 파일 작성

파일은 아래 경로에 위치시킵니다.

```
src/main/resources/
└── db/
    └── migration/
        └── V1__init.sql
```

```sql
-- V1__init.sql
CREATE TABLE users (
    id       BIGINT AUTO_INCREMENT PRIMARY KEY,
    username VARCHAR(50)  NOT NULL,
    email    VARCHAR(100) NOT NULL,
    created_at DATETIME   NOT NULL
);
```

### 4. 애플리케이션 실행

별도의 추가 작업 없이 애플리케이션을 실행하면 Spring Boot가 시작 시 자동으로 Flyway를 실행합니다.

---

## 마이그레이션 이력 확인

### DB에서 직접 조회

```sql
SELECT * FROM flyway_schema_history;
```

### 애플리케이션 시작 로그

```
Flyway Community Edition
Database: jdbc:mysql://localhost:3306/mydb
Successfully validated 2 migrations
Current version of schema: 2
```

### Gradle 명령어로 확인

```bash
./gradlew flywayInfo
```

아직 적용되지 않은 `Pending` 상태의 파일도 함께 확인할 수 있습니다.

```
+-----------+---------+---------------------+------+---------------------+---------+
| Category  | Version | Description         | Type | Installed On        | State   |
+-----------+---------+---------------------+------+---------------------+---------+
| Versioned | 1       | init                | SQL  | 2026-05-13 10:00:00 | Success |
| Versioned | 2       | add email column    | SQL  | 2026-05-13 11:00:00 | Success |
| Versioned | 3       | create orders table | SQL  | -                   | Pending |
+-----------+---------+---------------------+------+---------------------+---------+
```

---

## 주의사항

**한 번 적용된 SQL 파일은 절대 수정하면 안 됩니다.**

Flyway는 파일마다 체크섬을 저장합니다. 이미 적용된 파일을 수정하면 체크섬 불일치 오류가 발생하면서 애플리케이션이 시작되지 않습니다.

변경이 필요하다면 반드시 새 버전 파일을 추가해야 합니다.

```
V1__init.sql          ← 수정 금지
V2__add_column.sql    ← 변경사항은 새 파일로 추가
```

---

## 프로젝트 초반 전략

스키마가 자주 바뀌는 초기 단계에서는 `V1__init.sql` 하나에 전체 스키마를 몰아넣는 방식을 추천합니다.

```sql
-- V1__init.sql
CREATE TABLE users (...);
CREATE TABLE orders (...);
CREATE TABLE products (...);
```

스키마가 안정화되기 전에는 DB를 초기화하고 `V1__init.sql`을 직접 수정하는 방식이 더 편합니다. 이후 스키마가 안정되면 그때부터 버전을 나눠 관리하면 됩니다.

```
V1__init.sql        ← 고정
V2__add_column.sql  ← 안정화 이후 변경분부터 버전 관리
```

---

## 마치며

Flyway를 적용하면 DB 변경사항을 git으로 함께 관리할 수 있습니다. 코드 변경과 DB 변경 이력을 함께 추적할 수 있어 "이 기능을 개발할 때 어떤 테이블이 추가됐는지"를 나중에도 쉽게 파악할 수 있습니다.

처음에는 다소 번거롭게 느껴질 수 있지만, 배포 환경이 늘어날수록 Flyway의 가치가 커집니다. 새 프로젝트를 시작한다면 초반부터 도입하는 것을 추천합니다.
