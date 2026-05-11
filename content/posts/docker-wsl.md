+++
date = '2026-05-10T21:42:28+09:00'
draft = false
title = 'WSL2로 개발환경 구축하기 3편 - Docker Desktop 설치 및 WSL 연동'
categories = ["개발"]
tags = ["wsl2", "docker", "docker-desktop", "windows"]
+++

## 들어가며

이번 편에서는 Docker Desktop을 설치하고 WSL2와 연동합니다.
WSL2와 연동하면 Windows Docker Desktop의 GUI 편의성과 Linux Docker의 성능을 함께 누릴 수 있습니다.

### 시리즈 구성

- 1편 - WSL2 설치 및 Ubuntu 기본 세팅
- 2편 - SDKMAN + JDK 설치
- **3편 - Docker Desktop 설치 및 WSL 연동** ← 현재 글
- 4편 - IntelliJ WSL 원격 개발 환경 설정

---

## 1. Docker Desktop 다운로드 및 설치

[Docker Desktop 다운로드](https://www.docker.com/products/docker-desktop/) 에서 Windows용 Docker Desktop을 다운로드합니다.

설치 파일 실행 시 **반드시 관리자 권한으로 실행**합니다.

> **설치 오류 발생 시**
>
> `C:\ProgramData\DockerDesktop must be owned by an elevated account` 오류가 발생하면
> PowerShell 관리자 권한에서 아래 명령어를 실행 후 재시도합니다.
>
> ```powershell
> takeown /f "C:\ProgramData\DockerDesktop" /r /d y
> icacls "C:\ProgramData\DockerDesktop" /grant Administrators:F /t
> ```

---

## 2. 라이선스 확인

설치 완료 후 Docker Subscription Service Agreement가 표시됩니다.

개인 및 소규모 회사는 **무료**로 사용할 수 있습니다.

| 조건 | 유/무료 |
|------|--------|
| 개인 사용 | 무료 ✅ |
| 직원 250명 미만 | 무료 ✅ |
| 연매출 $10M 미만 | 무료 ✅ |
| 위 조건 초과 기업 | 유료 💰 |

**Accept**를 클릭합니다.

---

## 3. WSL Integration 설정

Docker Desktop 실행 후 Settings로 이동합니다.

**Settings → Resources → WSL Integration**

- `Enable integration with my default WSL distro` 체크
- Ubuntu 토글 **ON**

**Apply & restart** 클릭합니다.

---

## 4. 설치 확인

WSL Ubuntu 터미널에서 확인합니다.

```bash
docker --version
docker compose version
```

정상 설치 시 아래와 같이 출력됩니다.

```
Docker version 29.4.1, build 055a478
Docker Compose version v5.1.3
```

---

## 마치며

Docker Desktop WSL 연동이 완료되었습니다.
이제 WSL 터미널에서 `docker` 및 `docker compose` 명령어를 바로 사용할 수 있습니다.

다음 편에서는 IntelliJ에서 WSL 원격 개발 환경을 설정하는 방법을 다룹니다.
