---
title: "[Note] curl + OpenSSL 빌드 방법 정리 no shared 옵션 빼고"
date: 2025-06-24 20:00:00 +0900
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

## 📦 사전 준비

### 🔧 필요 도구

| 도구 | 용도 |
|------|------|
| Visual Studio 2017 | C++ Desktop 개발 환경 |
| CMake | 빌드 설정 생성기 |
| Git | 소스 코드 다운로드 |
| Strawberry Perl | OpenSSL 빌드용 |
| NASM | OpenSSL 어셈블리 컴파일용 |
| Python | 일부 종속 빌드 시 필요 |

> 설치 후 환경 변수 `PATH`에 각 도구의 경로가 등록되어 있어야 함 (실제 빌드시에는 NASM만 등록함)

---

## 📁 디렉터리 구조 예시 및 파일 위치 안내

```plaintext
D:\Workspace\project(code)├── openssl\              # OpenSSL 소스 코드
├── openssl-build\        # OpenSSL 빌드 결과 (DLL, LIB, 헤더 포함)
├── curl\                 # curl 소스 코드
├── curl-build\           # CMake 빌드 대상 폴더
├── curl-install\         # INSTALL 프로젝트 결과물 (최종 바이너리)
```

---

## 1️⃣ OpenSSL 3.2.1 빌드

### 🔹 소스 다운로드 (🖥 Git Bash 또는 일반 명령 프롬프트에서 실행) 및 빌드  (🖥 Git Bash 또는 일반 명령 프롬프트에서 실행)

```cmd
cd D:\Workspace\project(code)
git clone https://github.com/openssl/openssl.git
cd openssl
git checkout openssl-3.2

📌 아래 명령어는 반드시 ➜ Visual Studio 2017 Developer Command Prompt (x64)에서 실행
perl Configure VC-WIN64A no-tests
nmake
nmake install DESTDIR="D:\Workspace\project(code)\openssl-build"
```

> 🔶 참고: `no-shared` 옵션을 사용할 경우 DLL 파일(`libssl-3-x64.dll`, `libcrypto-3-x64.dll`)은 생성되지 않고 `.lib`만 생성됩니다.  
> Digest 인증에서 필요한 DLL을 만들려면 `no-shared` 옵션 없이 빌드해야 합니다. 현재는 `no-shared` 없이 빌드에 실패하고 있어 해당 문제는 추후 해결 예정입니다.
> 직접 checkout 할 브랜치 명 확인 필요해보임

---

## 2️⃣ curl 8.6.0 빌드 (OpenSSL 연동, SSPI 비활성화)

### 🔹 소스 다운로드 (🖥 Git Bash 또는 일반 명령 프롬프트에서 실행)

```cmd
cd D:\Workspace\project(code)
git clone https://github.com/curl/curl.git
cd curl
git checkout curl-8_6_0

:: CMake 빌드를 위한 디렉터리 생성
cd ..
mkdir curl-build
cd curl-build

```

---

### 🔹 CMake 설정 (📌 Visual Studio 2017 Developer Command Prompt (x64)에서 실행)

```cmd
cd ..\curl-build

cmake ..\curl ^
 -DCMAKE_BUILD_TYPE=Release ^
 -DCURL_USE_OPENSSL=ON ^
 -DOPENSSL_ROOT_DIR="D:/Workspace/openssl-build/usr" ^
 -DOPENSSL_INCLUDE_DIR="D:/Workspace/openssl-build/usr/include" ^
 -DCMAKE_INSTALL_PREFIX="D:/Workspace/curl-install" ^
 -DENABLE_SSPI=OFF ^
 -DCURL_USE_SCHANNEL=OFF ^
 -DCURL_DISABLE_LIBPSL=ON ^
 -DBUILD_SHARED_LIBS=ON ^
 -G "Visual Studio 15 2017" -A x64
```
```cmd
cmake ..\curl ^
 -DCMAKE_BUILD_TYPE=Release ^
 -DCURL_USE_OPENSSL=ON ^
 -DOPENSSL_ROOT_DIR="D:/Workspace/openssl-build/Program Files/OpenSSL" ^
 -DOPENSSL_INCLUDE_DIR="D:/Workspace/openssl-build/Program Files/OpenSSL/include" ^
 -DCMAKE_INSTALL_PREFIX="D:/Workspace/curl-install" ^
 -DENABLE_SSPI=OFF ^
 -DCURL_USE_SCHANNEL=OFF ^
 -DCURL_DISABLE_LIBPSL=ON ^
 -DBUILD_SHARED_LIBS=ON ^
 -G "Visual Studio 15 2017" -A x64
 ```
### 🔹 CMake 재실행시 캐시 정리
```cmd
cd D:\Workspace\curl-build
del /f CMakeCache.txt
rmdir /s /q CMakeFiles
```


#### 🔑 주요 CMake 옵션 설명

| 옵션 | 설명 |
|------|------|
| `-DCMAKE_USE_OPENSSL=ON` | OpenSSL 사용 활성화 |
| `-DOPENSSL_ROOT_DIR`, `-DOPENSSL_INCLUDE_DIR` | OpenSSL 빌드 경로 및 헤더 경로 지정 |
| `-DCMAKE_INSTALL_PREFIX` | 최종 결과물이 설치될 디렉토리 경로 |
| `-DBUILD_SHARED_LIBS=ON` | DLL (동적 라이브러리) 생성 여부 |
| `-DENABLE_SSPI=OFF` | Windows의 SSPI 인증(네이티브 Digest 등) 비활성화 → OpenSSL 기반 인증만 사용 |
| `-DCURL_USE_SCHANNEL=OFF` | Windows 기본 SSL/TLS 대신 OpenSSL 사용 |

---

### 🔹 CMakeLists.txt 수정 (필요 시)

일부 환경에서 헤더 검색 오류가 발생할 수 있으므로, `CMakeLists.txt` 파일 상단에 아래 내용을 추가합니다.

📍 **추가 위치**: `project(...)` 호출 바로 아래

```cmake
project(CURL C)

include_directories("${CMAKE_SOURCE_DIR}/include")
include_directories("${CMAKE_SOURCE_DIR}/lib")
```

---

## 3️⃣ Visual Studio 빌드

### 🔹 빌드 방법

1. `curl-build\CURL.sln` 파일을 Visual Studio 2017로 열기
2. 상단 구성에서 `Release | x64` 선택
3. 솔루션 탐색기에서 다음 프로젝트 순서로 빌드
   - ✅ `ALL_BUILD`
   - ✅ `INSTALL` (자동으로 curl-install에 복사됨)

### 🔹 추가 설정 (프로젝트 속성)

빌드 오류가 발생할 경우, 다음 경로들을 수동으로 프로젝트 속성에 추가해야 할 수 있습니다.
( 위 설정이 잘 되지 않아 CMakeList.txt 수정 진행함)

- `libcurl` 프로젝트 우클릭 → 속성 → `C/C++ → 일반 → 추가 포함 디렉터리`:
  ```
  D:\Workspace\project(code)\openssl-build\usr\include
  ```

- `링커 → 일반 → 추가 라이브러리 디렉터리`:
  ```
  D:\Workspace\project(code)\openssl-build\usr\lib
  ```

- `링커 → 입력 → 추가 종속성`:
  ```
  libssl.lib
  libcrypto.lib
  ```

---

## 4️⃣ 빌드 결과 및 검증

```cmd
cd D:\Workspace\project(code)\curl-install\bin
curl.exe --version
```

예시 출력:

```bash
curl 8.6.0 (Windows) libcurl/8.6.0 OpenSSL/3.2.1 ...
Protocols: http https
Features: SSL OpenSSL ...
```

- `SSPI` 없음, `OpenSSL` 있음 → Digest + SHA256 인증 지원 가능

---

## 5️⃣ Digest 인증 테스트 (SHA256)

```bash
curl --digest -u admin:0000 -k "https://192.168.40.157:443/stw-cgi/system.cgi?msubmenu=deviceinfo&action=view"
```

- SSPI 비활성화 → Windows 인증 비의존
- OpenSSL 기반 Digest 인증

---

## 📁 결과물 구성 요약

```plaintext
curl-install/
├── bin/
│   ├── curl.exe              ← CLI 도구
│   ├── libcurl.dll           ← JNI 등 연동용 DLL
│   ├── libssl-3-x64.dll      ← OpenSSL DLL (no-shared 없이 빌드시만 생성)
│   └── libcrypto-3-x64.dll   ← OpenSSL DLL
├── lib/
│   └── libcurl.lib           ← DLL import용 또는 정적 링크용
├── include/
│   └── curl/                 ← 헤더 파일
```
* no-shared 포함 빌드는 libssl-3-x64.dll / libcrypto-3-x64.dll 필요없음
* no-shared 없는 빌드는 openssl 에서 빌드한 두 파일이 필요함

---

## ✅ 완료 요약

| 항목 | 상태 |
|------|------|
| OpenSSL 3.2 빌드 | ✔ 완료 |
| curl 소스 설정 및 체크아웃 | ✔ 완료 |
| SSPI 비활성화 및 OpenSSL 연동 | ✔ 완료 |
| Visual Studio 빌드 (DLL 포함) | ✔ 완료 |
| Digest 인증 + HTTPS 테스트 | ✔ 완료 |
| JNI 연동 및 배포 준비 | ✔ 완료 (DLL 기반) |
