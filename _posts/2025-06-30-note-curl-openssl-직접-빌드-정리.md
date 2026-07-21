---
title: "[Note] Digest SHA-256 테스트를 위해 Windows용 curl/libcurl을 직접 빌드한 기록"
date: 2025-06-25 20:00:00 +0900
categories: [note]
layout: single
---

# Digest SHA-256 테스트를 위해 Windows용 curl/libcurl을 직접 빌드한 기록

## 상황

HTTPS 기반 API를 테스트하면서 Digest 인증 문제가 있었다.

처음에는 단순히 `curl --digest` 옵션을 주면 될 것이라고 생각했다. 그런데 Windows에서 쓰는 curl은 빌드 방식에 따라 TLS 처리 라이브러리와 인증 처리 방식이 달라진다. 같은 명령어를 실행해도 어떤 curl을 쓰느냐에 따라 결과가 달라질 수 있었다.

특히 내가 확인해야 했던 부분은 아래였다.

- Digest 인증에서 SHA-256 알고리즘을 처리할 수 있는지
- Windows SSPI에 인증 처리를 맡기지 않는지
- TLS 백엔드가 Schannel이 아니라 OpenSSL인지
- 애플리케이션에 붙일 수 있는 `libcurl.dll`을 만들 수 있는지

그래서 최종적으로는 Windows에서 사용할 `curl.exe`와 `libcurl.dll`을 직접 빌드했다.

## 용어 정리

`curl`은 명령 프롬프트에서 HTTP 요청을 보내는 도구다.

`libcurl`은 curl의 기능을 C/C++ 애플리케이션에서 사용할 수 있게 만든 라이브러리다. Java 애플리케이션에서도 JNI 같은 방식으로 네이티브 DLL을 붙이면 내부 통신에 사용할 수 있다.

`OpenSSL`은 TLS/SSL 통신과 암호화 기능을 제공하는 라이브러리다.

`Schannel`은 Windows에 기본으로 들어 있는 TLS 처리 기능이다. Windows 인증서 저장소와 잘 붙는 장점이 있지만, 내가 원하는 테스트에서는 OpenSSL 기반 동작을 확인해야 했다.

`SSPI`는 Windows가 제공하는 인증 처리 인터페이스다. curl이 SSPI를 사용하면 Digest 인증 일부를 Windows 쪽에 맡길 수 있는데, 이 경우 SHA-256 Digest 같은 조합에서 기대와 다르게 동작할 수 있었다.

즉, 내가 원한 구성은 단순히 "curl이 되는 것"이 아니라 아래 조합이었다.

```text
curl/libcurl + OpenSSL + SSPI 비활성화 + Schannel 비활성화
```

## 왜 직접 빌드했는지

문제 상황에서 가장 헷갈렸던 건 같은 `curl` 명령어라도 결과가 환경에 따라 달라진다는 점이었다.

```bash
curl --digest -u "<user>:<password>" -k "https://<device-host>/<api-path>"
```

명령어는 같아도 curl이 어떤 TLS 백엔드를 쓰는지, Digest 인증을 자체 처리하는지, Windows SSPI에 넘기는지에 따라 성공/실패가 달라질 수 있다.

그래서 이미 설치된 curl을 그대로 쓰면 아래 질문에 답하기 어려웠다.

| 질문 | 기존 curl만으로 어려웠던 이유 |
| --- | --- |
| 지금 OpenSSL로 통신 중인가? | Windows 기본 curl은 Schannel 기반일 수 있음 |
| Digest SHA-256을 누가 처리하는가? | curl 내부 처리인지 SSPI 위임인지 헷갈릴 수 있음 |
| 애플리케이션에 같은 동작을 붙일 수 있는가? | `curl.exe` 성공과 `libcurl.dll` 연동은 별도 문제 |
| 실패가 장비 문제인가, 클라이언트 빌드 문제인가? | 빌드 구성을 고정하지 않으면 원인 분리가 어려움 |

결국 내가 통제할 수 있는 curl/libcurl을 만들어야 했다.

## 내가 목표로 잡은 빌드 결과

목표는 아래 결과물을 만드는 것이었다.

```text
curl-install/
├── bin/
│   ├── curl.exe
│   ├── libcurl.dll
│   ├── libssl-3-x64.dll
│   └── libcrypto-3-x64.dll
├── lib/
│   └── libcurl.lib
└── include/
    └── curl/
```

`curl.exe`는 명령어 테스트용이고, `libcurl.dll`은 애플리케이션에 붙이기 위한 결과물이다.

`libssl-3-x64.dll`, `libcrypto-3-x64.dll`은 OpenSSL 동적 라이브러리다. OpenSSL을 DLL 방식으로 빌드하면 실행 시 이 파일들도 같이 필요하다.

## 1. OpenSSL을 먼저 빌드했다

OpenSSL 빌드에는 Visual Studio Developer Command Prompt, Perl, NASM이 필요했다.

- Visual Studio Developer Command Prompt: MSVC 빌드 환경을 잡아주는 명령 프롬프트
- Perl: OpenSSL 설정 스크립트 실행에 필요
- NASM: 일부 어셈블리 최적화 코드 빌드에 필요

대략적인 흐름은 아래와 같다.

```cmd
git clone https://github.com/openssl/openssl.git
cd openssl
git checkout openssl-3.2.1

perl Configure VC-WIN64A no-tests
nmake
nmake install DESTDIR="D:\deps\openssl-build"
```

처음에 헷갈렸던 부분은 `no-shared` 옵션이었다.

`no-shared`를 넣으면 정적 라이브러리 중심으로 빌드되고 DLL이 나오지 않을 수 있다. 내가 필요한 건 애플리케이션에서 같이 배포할 수 있는 DLL 구성이었기 때문에, 최종적으로는 DLL이 생성되는 구성을 기준으로 확인했다.

이런 부분은 빌드 로그만 보면 놓치기 쉽다. 그래서 설치 폴더에 실제로 `libssl-3-x64.dll`, `libcrypto-3-x64.dll`이 만들어졌는지까지 봤다.

## 2. curl은 OpenSSL을 쓰도록 CMake 옵션을 고정했다

curl은 CMake로 Visual Studio 솔루션을 만들었다.

핵심은 OpenSSL을 켜고, Windows 기본 TLS/인증 쪽 의존을 끄는 것이었다.

```cmd
cmake ..\curl ^
 -DCMAKE_BUILD_TYPE=Release ^
 -DCURL_USE_OPENSSL=ON ^
 -DOPENSSL_ROOT_DIR="D:/deps/openssl-build/Program Files/OpenSSL" ^
 -DOPENSSL_INCLUDE_DIR="D:/deps/openssl-build/Program Files/OpenSSL/include" ^
 -DCMAKE_INSTALL_PREFIX="D:/deps/curl-install" ^
 -DENABLE_SSPI=OFF ^
 -DCURL_USE_SCHANNEL=OFF ^
 -DCURL_DISABLE_LIBPSL=ON ^
 -DBUILD_SHARED_LIBS=ON ^
 -G "Visual Studio 15 2017" -A x64
```

내가 중요하게 본 옵션은 이렇다.

| 옵션 | 확인한 이유 |
| --- | --- |
| `-DCURL_USE_OPENSSL=ON` | curl이 OpenSSL을 TLS 백엔드로 쓰게 하기 위해 |
| `-DENABLE_SSPI=OFF` | Windows SSPI 인증 위임을 피하기 위해 |
| `-DCURL_USE_SCHANNEL=OFF` | Windows Schannel TLS 백엔드를 쓰지 않게 하기 위해 |
| `-DBUILD_SHARED_LIBS=ON` | `libcurl.dll`을 만들기 위해 |
| `-DOPENSSL_ROOT_DIR` | 직접 빌드한 OpenSSL 경로를 명확히 지정하기 위해 |

여기서 한 번 설정을 잘못하면 CMake 캐시 때문에 계속 이전 경로를 물고 가는 경우가 있었다. 그래서 옵션을 바꾼 뒤에는 캐시를 지우고 다시 생성했다.

```cmd
del /f CMakeCache.txt
rmdir /s /q CMakeFiles
```

이 부분에서 시간을 꽤 썼다. 명령어는 맞게 고쳤는데 결과가 그대로라서 이상했는데, 실제로는 CMake 캐시에 이전 OpenSSL 경로가 남아 있었다.

## 3. Visual Studio 빌드 순서를 확인했다

CMake로 솔루션을 만든 뒤에는 Visual Studio에서 `Release | x64`로 빌드했다.

내가 확인한 순서는 아래였다.

```text
1. ALL_BUILD 또는 libcurl_shared 빌드
2. curl.exe 빌드
3. INSTALL 프로젝트 빌드
4. curl-install 폴더에 exe, dll, lib, include가 복사됐는지 확인
```

처음에는 빌드가 성공해도 최종 설치 폴더에 필요한 파일이 다 모이지 않는 경우가 있었다. Visual Studio 안에서 프로젝트 하나만 빌드하면 `libcurl.dll`은 만들어졌지만, 배포용 폴더가 정리되지 않을 수 있었다.

그래서 마지막에는 항상 `INSTALL` 프로젝트까지 실행하고, 결과 폴더를 직접 확인했다.

## 4. 진짜 OpenSSL curl인지 확인했다

빌드가 끝난 뒤에는 바로 Digest 테스트로 가지 않고 버전부터 확인했다.

```cmd
cd D:\deps\curl-install\bin
curl.exe --version
```

기대했던 출력은 이런 형태였다.

```text
curl 8.6.0 (Windows) libcurl/8.6.0 OpenSSL/3.2.1
Protocols: http https ...
Features: SSL ...
```

여기서 내가 확인한 기준은 단순했다.

- `OpenSSL`이 보여야 한다.
- `Schannel`이 보이면 안 된다.
- `SSPI`가 보이면 안 된다.
- `https` 프로토콜이 포함되어야 한다.
- DLL 경로가 내가 빌드한 결과물 쪽이어야 한다.

이 확인을 하고 나서야 "지금 테스트하는 curl은 내가 만든 curl"이라고 말할 수 있었다.

## 5. Digest 인증 테스트를 다시 했다

마지막으로 같은 요청을 직접 빌드한 curl로 다시 테스트했다.

```bash
curl -vk --digest -u "<user>:<password>" "https://<device-host>/<api-path>"
```

여기서 본 것은 성공 여부만이 아니었다.

- `WWW-Authenticate: Digest ...`가 내려오는지
- `algorithm=SHA-256` 같은 값이 있는지
- curl이 다시 `Authorization: Digest ...`를 붙여 보내는지
- TLS 오류인지 HTTP 인증 오류인지 구분되는지

Digest 인증은 처음 401이 나올 수 있기 때문에, 단순히 401만 보고 실패라고 보면 안 된다. `WWW-Authenticate` 헤더를 보고 인증 방식 협상이 시작됐는지 확인해야 한다.

## 작업하면서 고생한 부분

이 작업은 코드 몇 줄 수정처럼 깔끔하게 끝나는 일이 아니었다.

| 막혔던 부분 | 내가 확인한 방식 |
| --- | --- |
| 빌드는 됐는데 DLL이 없음 | OpenSSL `no-shared` 옵션과 설치 결과 폴더 확인 |
| 옵션을 바꿨는데 결과가 안 바뀜 | CMake 캐시 삭제 후 재생성 |
| OpenSSL 경로를 못 찾음 | `OPENSSL_ROOT_DIR`, `OPENSSL_INCLUDE_DIR`를 명시 |
| curl은 되는데 libcurl 배포 파일이 부족함 | Visual Studio `INSTALL` 프로젝트까지 실행 |
| Digest 실패가 서버 문제인지 헷갈림 | `curl -v`로 401 challenge와 Authorization 재요청 확인 |
| 내가 만든 curl이 맞는지 불안함 | `curl --version`과 DLL 로딩 경로를 반복 확인 |

가장 많이 느낀 건 "빌드 성공"이라는 말이 너무 넓다는 점이었다. 컴파일이 성공한 것, 실행 파일이 만들어진 것, 원하는 SSL 백엔드를 쓰는 것, 애플리케이션에서 같은 DLL을 로딩하는 것은 전부 다른 확인 항목이었다.

## 내가 남긴 검증 기준

이후 비슷한 작업을 할 때는 아래 순서로 보면 된다.

```text
1. curl --version에서 OpenSSL/Schannel/SSPI 표시 확인
2. OpenSSL DLL이 실행 폴더 또는 PATH에서 정상 로딩되는지 확인
3. curl -vk로 TLS 연결 단계 확인
4. curl --digest -v로 Digest challenge와 Authorization 재요청 확인
5. 애플리케이션에 libcurl.dll 적용 후 동일 옵션이 전달되는지 확인
6. 실제 장비에서는 장비 API 응답과 상태 변경을 별도로 확인
```

이 순서가 있으면 문제를 만났을 때 처음부터 감으로 보지 않아도 된다.

## 관련해서 이어진 글

- [실제 장비 테스트 전 TLS/libcurl 업그레이드 위험을 줄인 방법]({% post_url 2025-06-26-note-tls-upgrade-validation-without-device %})
- [HTTP Digest와 네트워크 카메라 API 통신 구조 이해하기]({% post_url 2026-07-21-note-http-digest-camera-api-communication %})

## 정리

이 작업에서 내가 한 일은 단순히 curl을 설치한 것이 아니었다.

Windows 기본 curl 동작을 그대로 믿지 않고, Digest SHA-256 테스트에 필요한 조건을 직접 통제할 수 있게 빌드 구성을 잡았다. 그리고 `curl.exe` 테스트와 `libcurl.dll` 배포까지 이어질 수 있도록 결과물을 확인했다.

전문적인 빌드 시스템을 처음부터 완벽하게 안 상태에서 한 작업은 아니었다. 오히려 옵션 하나 바꾸고, 실패 로그 보고, 캐시 지우고, 다시 빌드하고, 버전 출력 확인하는 과정을 반복하면서 기준을 잡았다.

그래도 이 과정을 거치면서 "내가 어떤 curl로 테스트하고 있는지"를 설명할 수 있게 됐다. 그게 이후 TLS/libcurl 업그레이드나 장비 API 인증 문제를 볼 때 가장 큰 기준이 됐다.
