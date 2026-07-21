---
title: "[Note] SSPI를 끈 Windows용 libcurl을 직접 빌드한 기록"
date: 2025-06-30 20:00:00 +0900
categories: [note]
tags: [curl, libcurl, openssl, cmake, windows, sspi]
layout: single
---

카메라 API의 Digest 인증이 Windows curl에서 오류 94로 실패했다. 명령 옵션만 바꾸는 것으로는 해결되지 않았고, 애플리케이션에서 사용하는 `libcurl.dll`까지 같은 조건으로 만들어야 했다.

이번 빌드의 목표는 다음과 같았다.

```text
curl.exe          명령행 재현용
libcurl.dll       C/C++ 네이티브 모듈 연동용
libcurl.lib       Visual Studio 링크용
include/curl/*    헤더
OpenSSL           TLS 백엔드
SSPI OFF          Digest 인증의 Windows 위임 제거
```

## OpenSSL과 Digest의 역할을 구분했다

이 작업에서 가장 헷갈렸던 부분은 OpenSSL을 붙이면 Digest SHA-256도 자동으로 해결된다고 생각하기 쉽다는 점이었다.

- OpenSSL은 HTTPS의 TLS 연결과 암호 기능을 제공한다.
- HTTP Digest는 libcurl의 HTTP 인증 처리다.
- SSPI가 켜진 Windows 빌드는 Digest 처리를 Windows 인증 계층에 맡길 수 있다.

따라서 핵심은 “OpenSSL로 Digest를 처리한다”가 아니라 **TLS 백엔드는 OpenSSL로 고정하고, Digest는 SSPI에 위임하지 않는 libcurl을 만드는 것**이었다.

## 1. OpenSSL 3.2.1 빌드

Visual Studio Developer Command Prompt에서 Perl과 NASM을 준비한 뒤 x64용 OpenSSL을 빌드했다.

```cmd
perl Configure VC-WIN64A no-tests
nmake
nmake install
```

처음에는 `no-shared` 옵션을 넣었다. 빌드는 됐지만 내가 필요로 한 `libssl-3-x64.dll`, `libcrypto-3-x64.dll`이 나오지 않았다. `no-shared`는 정적 라이브러리를 만드는 선택이므로 DLL 배포 구성을 원한다면 맞지 않았다.

이때부터 “명령이 성공했는가”보다 “필요한 결과물이 만들어졌는가”를 기준으로 보기 시작했다.

## 2. curl 8.6.0 CMake 구성

curl은 OpenSSL을 사용하고 Schannel과 SSPI를 사용하지 않도록 옵션을 고정했다.

```cmd
cmake ..\curl ^
  -G "Visual Studio 15 2017" -A x64 ^
  -DCURL_USE_OPENSSL=ON ^
  -DOPENSSL_ROOT_DIR="D:/deps/openssl" ^
  -DENABLE_SSPI=OFF ^
  -DCURL_USE_SCHANNEL=OFF ^
  -DBUILD_SHARED_LIBS=ON ^
  -DCURL_DISABLE_LIBPSL=ON ^
  -DCMAKE_INSTALL_PREFIX="D:/deps/curl-install"
```

| 옵션 | 사용한 이유 |
| --- | --- |
| `CURL_USE_OPENSSL=ON` | TLS 백엔드를 OpenSSL로 선택 |
| `ENABLE_SSPI=OFF` | Windows 인증 위임 비활성화 |
| `CURL_USE_SCHANNEL=OFF` | Schannel TLS 백엔드 제외 |
| `BUILD_SHARED_LIBS=ON` | 애플리케이션에 배포할 DLL 생성 |
| `CMAKE_INSTALL_PREFIX` | 결과물을 한 폴더에 모으기 위해 사용 |

## CMake 캐시 때문에 같은 빌드를 반복했다

OpenSSL 경로나 옵션을 고쳤는데도 결과가 바뀌지 않는 일이 있었다. 원인은 이전 설정이 `CMakeCache.txt`와 `CMakeFiles`에 남아 있었기 때문이다.

```cmd
del /f CMakeCache.txt
rmdir /s /q CMakeFiles
```

캐시를 지운 뒤 CMake 생성부터 다시 실행했다. 빌드 오류보다 이 문제가 더 시간을 많이 썼다. 명령줄은 바뀌었는데 생성된 Visual Studio 프로젝트는 이전 설정을 계속 사용하고 있었기 때문이다.

## 3. 빌드와 설치 결과 확인

Visual Studio에서는 `Release | x64`로 다음 순서를 확인했다.

```text
1. libcurl 공유 라이브러리 빌드
2. curl 실행 파일 빌드
3. INSTALL 프로젝트 실행
4. install 폴더의 bin/lib/include 확인
```

프로젝트 하나만 빌드하면 DLL은 생겨도 헤더와 import library가 배포 폴더에 정리되지 않았다. `INSTALL`까지 실행한 뒤 다음 파일을 직접 확인했다.

```text
curl-install/
  bin/curl.exe
  bin/libcurl.dll
  bin/libssl-3-x64.dll
  bin/libcrypto-3-x64.dll
  lib/libcurl.lib
  include/curl/
```

## 4. 빌드 성공과 원하는 빌드는 다르다

결과물이 생겼다고 바로 애플리케이션에 넣지 않았다.

```cmd
curl.exe --version
```

확인 기준은 다음과 같았다.

- libcurl 버전이 의도한 버전인가
- TLS 백엔드에 OpenSSL이 표시되는가
- HTTPS 프로토콜이 포함되는가
- Schannel/SSPI가 빠져 있는가
- 실행 시 내가 만든 DLL을 로드하는가

그 뒤에야 Digest 요청을 다시 보냈다.

```bash
curl -v -k --digest -u "<user>:<password>" "https://<camera-host>/<api-path>"
```

첫 `401` 뒤에 `Authorization: Digest ...`가 붙은 재요청이 나가는지와 최종 응답을 함께 확인했다.

## 5. 애플리케이션에 적용할 때 확인한 것

`curl.exe`가 성공하는 것과 네이티브 모듈이 `libcurl.dll`을 제대로 사용하는 것은 별개다. 적용 후에는 다음을 다시 확인했다.

1. 실행 폴더에 새 DLL과 필요한 OpenSSL DLL이 있는가
2. 기존 버전 DLL이 다른 경로에서 먼저 로드되지 않는가
3. C++ 코드의 `CURLAUTH_DIGEST` 옵션이 그대로 전달되는가
4. 조회 API와 변경 API가 모두 성공하는가
5. 변경 후 새 비밀번호로 실제 인증되는가

이 과정에서 DLL 교체만으로 작업을 끝내지 않고 장비의 최종 상태까지 검증했다.

## 정리

직접 빌드하면서 가장 많이 확인한 것은 네 가지였다.

```text
컴파일 성공
≠ 필요한 DLL 생성
≠ 원하는 TLS/인증 옵션 적용
≠ 애플리케이션에서 실제 로드
```

OpenSSL의 정적/동적 빌드 차이, CMake 캐시, Visual Studio의 `INSTALL`, DLL 로딩 경로를 하나씩 확인한 뒤에야 재현 가능한 libcurl 빌드를 만들 수 있었다.

이 빌드가 필요했던 실제 장애는 다음 글에 정리했다.

- [카메라 비밀번호 변경 API가 curl (94)로 실패한 문제]({% post_url 2025-06-25-error-digest-인증-실패-원인-분석-및-해결-흐름 %})
