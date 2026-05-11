+++
date = '2026-05-10T21:48:22+09:00'
draft = false
title = 'WSL2로 개발환경 구축하기 4편 - IntelliJ WSL 원격 개발 환경 설정'
categories = ["개발"]
tags = ["wsl2", "intellij", "spring-boot", "kotlin"]
+++
## 들어가며

마지막 편에서는 IntelliJ IDEA에서 WSL 원격 개발 환경을 설정합니다.
WSL 원격 개발을 사용하면 IntelliJ의 편의성을 유지하면서 실제 Linux 환경에서 개발할 수 있습니다.

### 시리즈 구성

- 1편 - WSL2 설치 및 Ubuntu 기본 세팅
- 2편 - SDKMAN + JDK 설치
- 3편 - Docker Desktop 설치 및 WSL 연동
- **4편 - IntelliJ WSL 원격 개발 환경 설정** ← 현재 글

---

## 환경

- IntelliJ IDEA 2026.1.1 Ultimate
- WSL2 Ubuntu 24.04

> WSL 원격 개발은 **Ultimate 버전**에서만 지원됩니다.

---

## Windows IntelliJ + WSL JDK vs WSL 원격 개발

| 방식 | 설명 |
|------|------|
| Windows IntelliJ + WSL JDK | IntelliJ는 Windows, JDK만 WSL 사용 |
| **WSL 원격 개발** ✅ | IntelliJ 백엔드 자체를 WSL에서 실행 |

WSL 원격 개발 방식이 실제 Linux 환경과 동일하게 동작하므로 추천합니다.

---

## 1. 프로젝트 폴더 생성

WSL 터미널에서 미리 프로젝트 폴더를 생성합니다.

```bash
mkdir ~/projects
```

> Windows 드라이브(`/mnt/c/...`)에 프로젝트를 만들면 파일 I/O 성능이 크게 저하됩니다.
> 반드시 WSL 홈 디렉토리(`~/`) 안에 만드세요.

---

## 2. WSL 원격 개발 연결

IntelliJ 시작 화면에서

**원격 개발 → WSL → Ubuntu 선택**

최초 실행 시 IntelliJ 백엔드가 WSL 안에 자동으로 설치됩니다. (수 분 소요)

---

## 3. Spring Boot 프로젝트 생성

WSL 원격으로 연결된 IntelliJ에서

**새 프로젝트 → Spring Boot**

### 추천 설정

| 항목 | 값 |
|------|-----|
| 위치 | `~/projects/프로젝트명` |
| 언어 | Kotlin |
| 빌드 시스템 | Gradle - Kotlin |
| JDK | 25.0.2-librca |
| Java 버전 | 21 |
| Spring Boot | 3.x 최신 |

---

## 4. 터미널 설정 (선택)

Windows IntelliJ 터미널을 WSL로 변경하려면

**Settings → Tools → Terminal → Shell path**

```
wsl.exe
```

---

## 마치며

WSL2 기반 Java/Kotlin 개발환경 구축이 모두 완료되었습니다.

### 최종 환경 요약

| 항목 | 버전 |
|------|------|
| WSL2 + Ubuntu | 24.04 |
| SDKMAN | 5.23.0 |
| JDK | 25.0.2 (Liberica) |
| Docker | 29.4.1 |
| Docker Compose | v5.1.3 |
| IntelliJ IDEA | 2026.1.1 Ultimate |

이제 Windows에서도 Linux 환경과 동일하게 Spring Boot + Kotlin 개발을 할 수 있습니다.
