---
title: "curl + OpenSSL 직접 빌드 기록 (Digest 인증용)"
date: 2025-06-25 20:00:00 +0900
categories: [note]
layout: single
---


# 📘 개인 빌드 기록: Windows + Visual Studio 2017에서 libcurl.dll + OpenSSL 빌드 (Digest 인증용)

## 🧩 목적

- 최신 `curl/libcurl`을 Windows 환경에서 Visual Studio 2017로 직접 빌드
- Digest 인증 시 Windows SSPI 의존 제거
- OpenSSL 3.2.1과 연동하여 HTTPS + SHA256 인증 지원
- Java 등에서 사용 가능한 `libcurl.dll` 생성

---

## 📁 디렉터리 구조 및 역할

```plaintext
D:\Workspace\project(code)├── openssl\              # OpenSSL 소스 코드
├── openssl-build\        # OpenSSL 빌드 출력 (DLL, LIB, 헤더 등 포함)
├── curl\                 # curl 소스 코드
├── curl-build\           # CMake 빌드 대상 폴더
├── curl-install\         # INSTALL 프로젝트 실행 결과물 (최종 설치된 실행 파일들)
```

---

## 1️⃣ OpenSSL 3.2.1 빌드

### 🔹 의존성 설치

- NASM: https://www.nasm.us/
- Strawberry Perl: https://strawberryperl.com/
- Visual Studio 2017 Developer Command Prompt 실행

### 🔹 소스 다운로드 및 빌드

```bash
git clone https://github.com/openssl/openssl.git
cd openssl
git checkout openssl-3.2.1

perl Configure VC-WIN64A no-shared no-tests
nmake
nmake install DESTDIR="D:\Workspace\project(code)\openssl-build"
```

설치 결과:
- `include/`: OpenSSL 헤더
- `lib/`: `libssl.lib`, `libcrypto.lib`
- `bin/`: DLL

---

## 2️⃣ curl 8.6.0 빌드 (OpenSSL 연동, SSPI 비활성화)

### 🔹 소스 다운로드

> ❗ `curl-8_14_1` 버전은 빌드시 오류 발생하여 `curl-8_6_0`으로 변경

```bash
git clone https://github.com/curl/curl.git
cd curl
git checkout curl-8_6_0
```

---

### 🔹 CMake 설정 (Visual Studio 2017 x64)

```cmd
cmake ..\curl ^
 -DCMAKE_BUILD_TYPE=Release ^
 -DCURL_USE_OPENSSL=ON ^
 -DOPENSSL_ROOT_DIR="D:/Workspace/project(code)/openssl-build/Program Files/OpenSSL" ^
 -DOPENSSL_INCLUDE_DIR="D:/Workspace/project(code)/openssl-build/Program Files/OpenSSL/include" ^
 -DCMAKE_INSTALL_PREFIX="D:/Workspace/project(code)/curl-install" ^
 -DENABLE_SSPI=OFF ^
 -DCURL_USE_SCHANNEL=OFF ^
 -DCURL_DISABLE_LIBPSL=ON ^
 -DBUILD_SHARED_LIBS=ON ^
 -G "Visual Studio 15 2017" -A x64
```

---

### 🔹 CMakeLists.txt 수정 내용

추가 포함 디렉터리가 비어 있는 문제 해결을 위해 `CMakeLists.txt`에 다음 내용 수동 추가:

```cmake
include_directories("${CMAKE_SOURCE_DIR}/include")
include_directories("${CMAKE_SOURCE_DIR}/lib")
```

> 위치: `CMakeLists.txt` 상단 또는 `project()` 이후 적절한 위치

---

## 3️⃣ Visual Studio 빌드 순서

1. `curl-build\CURL.sln` 파일 열기
2. 솔루션 구성: `Release | x64` 선택
3. 아래 프로젝트 빌드
   - ✅ `libcurl_shared`
   - ✅ `curl`
   - ✅ `INSTALL`

---

## 4️⃣ 빌드 확인

```bash
cd D:\Workspace\project(code)\curl-install\bin
curl.exe --version
```

정상 출력 예:

```bash
curl 8.6.0-DEV (Windows) libcurl/8.6.0-DEV OpenSSL/3.2.1 zlib/1.2.13
Protocols: http https ...
Features: SSL OpenSSL ...
```

---

## 5️⃣ Digest 인증 테스트

```bash
curl --digest -u admin:0000 -k "https://192.168.40.157:443/stw-cgi/system.cgi?msubmenu=deviceinfo&action=view"
```

- SSPI 비활성화 → Windows 인증 비의존
- OpenSSL 기반 Digest 인증

---

## 6️⃣ 결과물 구성

```plaintext
curl-install/
├── bin/
│   ├── curl.exe              ← CLI 도구
│   ├── libcurl.dll           ← JNI 등 연동용 DLL
│   ├── libssl-3-x64.dll      ← OpenSSL DLL
│   └── libcrypto-3-x64.dll   ← OpenSSL DLL
├── lib/
│   └── libcurl.lib           ← DLL import 또는 정적 링크용
├── include/
│   └── curl/                 ← 헤더 파일
```

---

## ✅ 완료 요약

| 항목 | 상태 |
|------|------|
| OpenSSL 3.2.1 빌드 | ✔ 완료 |
| curl 소스 설정 및 체크아웃 | ✔ 완료 (8.6.0 사용) |
| SSPI 비활성화 및 OpenSSL 연동 | ✔ 완료 |
| Visual Studio 빌드 (DLL 포함) | ✔ 완료 |
| Digest 인증 + HTTPS 테스트 | ✔ 완료 |
| JNI 연동 및 배포 준비 | ✔ 완료 |
