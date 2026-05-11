+++
date = '2026-05-09T12:55:25+09:00'
draft = false
title = 'WSL2로 개발환경 구축하기 1편 - WSL2 설치 및 Ubuntu 기본 세팅'
draft = false
categories = ["개발"]
tags = ["wsl2", "ubuntu", "windows", "linux"]
+++
## 들어가며

Windows에서 개발하다 보면 Linux 환경이 필요한 경우가 많습니다.
WSL2(Windows Subsystem for Linux 2)를 사용하면 Windows에서도 Linux 환경을 네이티브에 가깝게 사용할 수 있습니다.

이 시리즈에서는 Windows 11에서 WSL2 기반 Java/Kotlin 개발환경을 구축하는 전 과정을 다룹니다.

### 시리즈 구성

- **1편 - WSL2 설치 및 Ubuntu 기본 세팅** ← 현재 글
- 2편 - SDKMAN + JDK 설치
- 3편 - Docker Desktop 설치 및 WSL 연동
- 4편 - IntelliJ WSL 원격 개발 환경 설정

---

## 환경

- OS: Windows 11 Home
- 목표: WSL2 + Ubuntu 24.04 설치

---

## WSL2란?

WSL2는 Windows 안에서 Linux를 실행할 수 있게 해주는 기술입니다.
가상머신과 달리 Windows와 파일 시스템을 공유하고, 성능도 훨씬 뛰어납니다.

| 비교 | 가상머신 | WSL2 |
|------|---------|------|
| 부팅 시간 | 수십 초 | 1~2초 |
| 파일 공유 | 별도 설정 필요 | 자동 |
| 성능 | 낮음 | 네이티브에 가까움 |

---

## 1. WSL2 + Ubuntu 설치

PowerShell을 **관리자 권한**으로 실행합니다.

```powershell
wsl --install
```

이 명령어 하나로 다음이 자동으로 설치됩니다.

- WSL2 기능 활성화
- Ubuntu 24.04 LTS 설치

설치 완료 후 **재부팅**합니다.

> 오류가 발생하는 경우 아래 명령어로 수동 활성화할 수 있습니다.
>
> ```powershell
> dism.exe /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux /all /norestart
> dism.exe /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart
> ```

---

## 2. Ubuntu 초기 설정

재부팅 후 Ubuntu가 자동으로 실행되면 username과 password를 설정합니다.

```
Enter new UNIX username: username
Enter new password: (입력해도 화면에 표시되지 않는 것이 정상)
```

---

## 3. 설치 확인

PowerShell에서 아래 명령어로 확인합니다.

```powershell
wsl -l -v
```

```
  NAME            STATE           VERSION
* Ubuntu          Stopped         2
```

**VERSION이 2**이면 WSL2로 정상 설치된 것입니다.

---

## 4. Ubuntu 기본 세팅

Ubuntu 터미널을 열고 패키지 목록 업데이트 및 업그레이드를 진행합니다.

```bash
sudo apt update && sudo apt upgrade -y
```

이후 작업에 필요한 기본 도구들을 설치합니다.

```bash
sudo apt install -y curl zip unzip git build-essential
```

| 패키지 | 용도 |
|--------|------|
| curl | URL로 파일 다운로드 |
| zip / unzip | 압축 / 압축 해제 |
| git | Git |
| build-essential | C/C++ 컴파일러 등 기본 빌드 도구 |

---

## 마치며

WSL2와 Ubuntu 기본 세팅이 완료되었습니다.
다음 편에서는 SDKMAN을 사용해 JDK를 설치하는 방법을 다룹니다.

