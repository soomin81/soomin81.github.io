+++
date = '2026-05-10T21:31:29+09:00'
draft = false
title = 'WSL2로 개발환경 구축하기 2편 - SDKMAN + JDK 설치'
categories = ["개발"]
+++
# WSL2로 개발환경 구축하기 2편 - SDKMAN + JDK 설치

## 들어가며

이번 편에서는 SDKMAN을 사용해 JDK를 설치합니다.
SDKMAN은 JDK, Kotlin, Gradle 등 JVM 관련 SDK를 버전별로 쉽게 관리할 수 있는 도구입니다.

### 시리즈 구성
- 1편 - WSL2 설치 및 Ubuntu 기본 세팅
- **2편 - SDKMAN + JDK 설치** ← 현재 글
- 3편 - Docker Desktop 설치 및 WSL 연동
- 4편 - IntelliJ WSL 원격 개발 환경 설정

---

## SDKMAN이란?

SDKMAN은 JVM 기반 SDK를 관리하는 CLI 도구입니다.
여러 JDK 버전을 설치하고 프로젝트별로 다르게 사용할 수 있습니다.

---

## 1. SDKMAN 설치

```bash
curl -s "https://get.sdkman.io" | bash
```

설치 완료 후 적용합니다.

```bash
source ~/.bashrc
```

버전 확인으로 설치를 검증합니다.

```bash
sdk version
```

```
SDKMAN!
script: 5.23.0
native: 0.7.34 (linux x86_64)
```

---

## 2. JDK 설치

### 설치 가능한 JDK 목록 확인

```bash
sdk list java
```

다양한 벤더의 JDK 목록이 표시됩니다.

```
================================================================================
Available Java Versions for Linux X64
================================================================================
 Vendor        | Use | Version      | Dist    | Status     | Identifier
--------------------------------------------------------------------------------
 Liberica      |     | 25.0.2       | librca  |            | 25.0.2-librca
 Liberica      |     | 25.0.2.fx    | librca  |            | 25.0.2.fx-librca
 Temurin       |     | 21.0.7       | tem     |            | 21.0.7-tem
 ...
================================================================================
```

### fx 버전과 일반 버전 차이

| 버전 | 포함 내용 | 용도 |
|------|----------|------|
| `25.0.2-librca` | JDK | 서버 개발 (Spring Boot 등) |
| `25.0.2.fx-librca` | JDK + JavaFX | 데스크탑 GUI 앱 개발 |

서버 개발이 목적이라면 fx 없는 버전을 선택합니다.

### JDK 설치

```bash
sdk install java 25.0.2-librca
```

### 설치 확인

```bash
java -version
```

```
openjdk version "25.0.2" 2026-01-20 LTS
OpenJDK Runtime Environment (build 25.0.2+12-LTS)
OpenJDK 64-Bit Server VM (build 25.0.2+12-LTS, mixed mode, sharing)
```

---

## 참고 - 유용한 SDKMAN 명령어

```bash
# 설치된 JDK 목록
sdk list java | grep installed

# JDK 버전 변경
sdk use java 21.0.7-tem

# 기본 JDK 변경
sdk default java 21.0.7-tem
```

---

## 마치며

SDKMAN과 JDK 설치가 완료되었습니다.
다음 편에서는 Docker Desktop을 설치하고 WSL과 연동하는 방법을 다룹니다.

