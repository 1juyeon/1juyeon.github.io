---
title: "[Error] 카메라 비밀번호 변경 API가 curl (94)로 실패한 문제"
date: 2025-06-25 20:00:00 +0900
categories: [error]
tags: [curl, libcurl, digest, sspi, openssl, camera-api]
layout: single
---

카메라 비밀번호를 변경하고 새 비밀번호가 맞는지 확인하는 기능을 구현하던 중, Windows에서 두 API가 모두 아래 오류로 끝났다.

```text
curl: (94) An authentication function returned an error
```

처음에는 새 비밀번호의 특수문자나 변경 URL을 의심했다. 하지만 장비 정보 조회처럼 상태를 바꾸지 않는 API도 같은 오류가 났다. 이 비교로 문제 범위를 비밀번호 변경 로직이 아니라 두 요청이 공통으로 사용하는 HTTP Digest 인증과 Windows용 libcurl 쪽으로 좁혔다.

## 구현하고 있던 흐름

네이티브 통신 모듈에서 요청 종류에 따라 URL을 만들고, 공통으로 Digest 인증을 설정하는 구조였다. 핵심만 줄이면 다음과 같다.

```cpp
std::string credentials = userId + ":" + currentPassword;

curl_easy_setopt(handle, CURLOPT_URL, requestUrl.c_str());
curl_easy_setopt(handle, CURLOPT_HTTPAUTH, CURLAUTH_DIGEST);
curl_easy_setopt(handle, CURLOPT_USERPWD, credentials.c_str());
curl_easy_setopt(handle, CURLOPT_WRITEFUNCTION, writeCallback);
```

비밀번호 변경 요청은 새 비밀번호를 URL 인코딩해 전송했고, 확인 요청은 장비 정보 조회 API를 새 자격증명으로 다시 호출하는 방식이었다. 애플리케이션 코드에는 이미 `CURLAUTH_DIGEST`가 들어가 있었기 때문에 단순한 옵션 누락은 아니었다.

## 먼저 401 응답을 다시 봤다

인증 정보를 빼고 `curl -v -k`로 요청해 서버가 제시하는 인증 방식을 확인했다.

```bash
curl -v -k "https://<camera-host>/<api-path>"
```

응답에는 다음과 같은 challenge가 있었다.

```http
HTTP/1.1 401 Unauthorized
WWW-Authenticate: Digest realm="...", nonce="...", qop="auth"
```

Digest 인증에서 첫 번째 `401`은 최종 실패가 아닐 수 있다. 서버가 `nonce`, `realm`, `qop` 같은 계산 조건을 알려주면 클라이언트가 `Authorization: Digest ...` 헤더를 만들어 재요청하는 것이 정상 흐름이다.

따라서 확인해야 할 것은 `401` 자체가 아니라 다음 두 가지였다.

1. 서버가 Basic과 Digest 중 어떤 방식을 요구하는가
2. challenge를 받은 curl이 Digest 헤더를 만들어 두 번째 요청을 보내는가

## 오류를 계층별로 분리했다

같은 HTTPS 요청이라도 실패 지점은 서로 다르다.

| 관찰한 결과 | 판단한 지점 |
| --- | --- |
| 연결 자체가 안 됨 | IP, 포트, 방화벽 |
| `curl: (60)` | 서버 인증서 검증 |
| 첫 `401`과 Digest challenge | 정상적인 인증 협상 시작 가능 |
| 인증 재요청 뒤 다시 `401` | 계정, 비밀번호, Digest 계산 확인 |
| `curl: (94)` | 클라이언트 인증 함수 또는 빌드 구성 확인 |

인증서 검증을 임시로 생략하는 `-k`는 Digest 문제를 해결하는 옵션이 아니다. TLS 인증서 오류와 HTTP 인증 오류를 분리해서 보기 위한 테스트 옵션일 뿐이다.

## 비밀번호 문자열보다 libcurl 빌드가 의심스러웠던 이유

처음에는 셸에서 특수문자가 잘못 해석되는지 확인하기 위해 자격증명을 따옴표로 감싸고, 같은 계정으로 여러 번 비교했다. 하지만 조회 API와 변경 API가 모두 같은 `(94)`로 실패했다.

Windows용 curl은 빌드 구성에 따라 동작 경로가 달라진다.

- `Schannel`: Windows의 TLS 처리 기능
- `SSPI`: Windows의 인증 처리 인터페이스
- `OpenSSL`: libcurl이 선택할 수 있는 별도의 TLS 백엔드

문제가 난 빌드는 Digest 처리를 Windows SSPI 경로에 맡기고 있었다. challenge에 포함된 `qop`와 알고리즘 조합을 처리하는 과정에서 `SEC_E_QOP_NOT_SUPPORTED` 계열 오류가 나면서 curl 오류 94로 이어졌다.

여기서 중요한 점은 TLS와 Digest를 섞어 생각하지 않는 것이다. OpenSSL과 Schannel은 HTTPS 연결을 담당하는 TLS 백엔드이고, Digest는 HTTP 인증이다. 이번 빌드의 핵심 변경은 **TLS 백엔드를 OpenSSL로 고정한 것과 함께, Digest 인증을 SSPI에 넘기지 않도록 비활성화한 것**이었다.

## 직접 빌드한 libcurl로 기준을 고정했다

설치된 curl을 계속 바꿔가며 비교하면 테스트 조건도 같이 바뀐다. 그래서 다음 조건으로 Windows용 curl과 libcurl을 직접 빌드했다.

```text
libcurl 8.6.0
TLS backend: OpenSSL
Schannel: OFF
SSPI: OFF
shared library: ON
```

빌드 후에는 먼저 실행 파일의 구성을 확인했다.

```bash
curl.exe --version
```

출력에서 OpenSSL이 표시되고 Schannel/SSPI가 빠졌는지 확인한 뒤, 동일한 장비와 계정으로 다시 테스트했다. 명령행 테스트가 끝난 뒤에는 애플리케이션이 사용하는 `libcurl.dll`도 같은 빌드 결과로 교체했다.

## DLL만 바꾸고 끝내지 않은 이유

비밀번호 변경 기능은 “HTTP 요청이 성공했다”만으로 완료되지 않는다. 실제 장비 상태까지 확인해야 한다.

```text
1. 기존 비밀번호로 조회 API 호출
2. 비밀번호 변경 API 호출
3. 새 비밀번호로 조회 API 재호출
4. 새 비밀번호가 맞을 때만 내부 저장값 갱신
5. 기존 비밀번호가 여전히 맞으면 변경 실패로 처리
6. 둘 다 확인되지 않으면 상태 불명으로 남기고 재확인
```

이 순서로 확인하면서 통신 라이브러리 교체가 조회 요청뿐 아니라 실제 변경과 사후 검증에도 같은 결과를 주는지 봤다.

## 이후 보완

처음에는 SSPI를 비활성화한 libcurl 8.6.0 빌드로 교체했다. 이후 다른 장비가 Digest challenge에서 SHA-256 알고리즘을 제시하는 경우까지 대응하기 위해 SHA-256 지원을 확인한 DLL로 다시 갱신했다.

즉 한 번의 curl 테스트로 끝난 작업이 아니라 다음 순서로 이어졌다.

```text
카메라 변경/조회 API의 curl (94) 확인
→ Digest challenge 비교
→ Windows SSPI 경로 문제 확인
→ OpenSSL TLS + SSPI OFF libcurl 직접 빌드
→ 애플리케이션 DLL 교체
→ 새 비밀번호로 실제 상태 검증
→ Digest SHA-256 장비까지 범위 확대
```

## 정리

이 문제의 핵심은 “장비 없이 TLS 업그레이드를 검증했다”가 아니다. 실제 카메라의 비밀번호 변경 API가 실패했고, 조회 요청과 변경 요청의 공통점을 비교해 Digest 인증 처리 경로를 찾은 것이 시작이었다.

내가 한 일은 API 응답을 Basic/Digest로 구분하고, 인증서 오류와 HTTP 인증 오류를 분리하고, Windows libcurl의 SSPI 의존을 끈 빌드를 만든 뒤 애플리케이션에 적용한 것이다. 마지막에는 새 비밀번호로 다시 접속해 장비 상태와 내부 저장 상태가 같이 맞는지까지 확인했다.

관련 기록:

- [SSPI를 끈 Windows용 libcurl을 직접 빌드한 기록]({% post_url 2025-06-30-note-curl-openssl-직접-빌드-정리 %})
- [카메라 API 문제를 풀기 위해 정리한 HTTP Digest 인증 흐름]({% post_url 2026-07-21-note-http-digest-camera-api-communication %})
