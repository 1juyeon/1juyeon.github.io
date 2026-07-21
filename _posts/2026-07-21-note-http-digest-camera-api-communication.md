---
title: "[Note] 카메라 API 문제를 풀기 위해 정리한 HTTP Digest 인증 흐름"
date: 2026-07-21 21:00:00 +0900
categories: [note]
tags: [http, digest, curl, authentication, camera-api]
layout: single
---

카메라 API를 다루면서 `401`, Digest, TLS, 인증서 오류가 한꺼번에 보이면 실패 지점을 섞어서 판단하기 쉽다. 실제 비밀번호 변경 API의 curl 오류 94를 해결하면서 필요했던 HTTP Digest 개념만 따로 정리했다.

## TLS와 Digest는 다른 계층이다

HTTPS 요청에는 두 가지 보안 과정이 겹쳐 있다.

```text
TLS
  서버와 암호화된 통신 경로를 만든다.
  인증서, TLS 버전, cipher가 여기에 속한다.

HTTP Digest
  HTTP 요청을 보낸 사용자가 올바른 계정 정보를 아는지 확인한다.
  realm, nonce, qop, algorithm이 여기에 속한다.
```

그래서 `-k`로 인증서 검증을 끈다고 Digest 인증이 해결되지는 않는다. 반대로 Digest 계정이 맞아도 TLS 연결이 실패하면 HTTP 요청까지 도달하지 못한다.

## 첫 번째 401은 challenge일 수 있다

Digest는 보통 두 번의 요청으로 진행된다.

```text
클라이언트                         서버
    |  인증 정보 없는 요청           |
    | ----------------------------> |
    |  401 + Digest challenge        |
    | <---------------------------- |
    |  Authorization: Digest ...     |
    | ----------------------------> |
    |  200 또는 최종 인증 실패         |
```

첫 응답 예시는 다음과 같다.

```http
HTTP/1.1 401 Unauthorized
WWW-Authenticate: Digest realm="camera",
                         nonce="server-generated-value",
                         algorithm=SHA-256,
                         qop="auth"
```

이 401은 서버가 “이 조건으로 인증값을 계산해서 다시 보내라”고 알려주는 단계다. challenge를 받은 뒤에도 두 번째 요청이 없거나 인증 함수 오류가 난다면 클라이언트의 Digest 처리 경로를 봐야 한다.

## challenge에 들어 있는 값

| 값 | 의미 |
| --- | --- |
| `realm` | 인증 영역 이름. 사용자 이름과 함께 해시 계산에 들어간다. |
| `nonce` | 서버가 만든 일회성 값. 이전 인증값의 재사용을 어렵게 한다. |
| `algorithm` | MD5, SHA-256 등 해시에 사용할 알고리즘 |
| `qop` | 보호 방식. API에서는 `auth`가 주로 보인다. |
| `opaque` | 서버가 그대로 돌려받기 위해 보내는 선택 값 |

두 번째 요청에는 클라이언트가 만든 값도 추가된다.

| 값 | 의미 |
| --- | --- |
| `uri` | 인증 계산에 사용한 요청 경로 |
| `cnonce` | 클라이언트가 만든 nonce |
| `nc` | 같은 nonce를 사용한 요청 횟수 |
| `response` | 비밀번호를 직접 보내는 대신 계산한 최종 해시 |

## response가 만들어지는 방식

`qop=auth`인 일반적인 흐름을 단순화하면 다음과 같다. `H()`는 challenge가 지정한 MD5 또는 SHA-256 같은 해시 함수다.

```text
HA1 = H(username : realm : password)
HA2 = H(method : request-uri)

response = H(
  HA1 : nonce : nc : cnonce : qop : HA2
)
```

서버도 같은 입력으로 계산해 `response`를 비교한다. 비밀번호 원문을 요청 헤더에 그대로 넣지는 않지만, Digest만으로 전송 내용 전체가 암호화되는 것은 아니므로 HTTPS와 함께 사용해야 한다.

## Basic과 Digest를 로그로 구분하는 방법

인증 정보 없이 verbose 요청을 보내면 서버가 요구하는 방식을 확인하기 쉽다.

```bash
curl -v -k "https://<camera-host>/<api-path>"
```

```http
WWW-Authenticate: Basic realm="camera"
```

위 응답이면 Basic이다.

```http
WWW-Authenticate: Digest realm="camera", nonce="...", qop="auth"
```

위 응답이면 Digest다. 같은 장비라도 API나 펌웨어에 따라 응답이 다를 수 있으므로 장비 종류만 보고 인증 방식을 고정하지 않고 실제 헤더를 확인하는 편이 안전하다.

## curl로 볼 때의 순서

### 1. 인증 방식만 확인

```bash
curl -v -k "https://<camera-host>/<api-path>"
```

`WWW-Authenticate`를 확인한다.

### 2. Digest로 재요청

```bash
curl -v -k --digest \
  -u "<user>:<password>" \
  "https://<camera-host>/<api-path>"
```

verbose 로그에서 다음 순서를 찾는다.

```text
< HTTP/1.1 401 Unauthorized
< WWW-Authenticate: Digest ...
> Authorization: Digest ...
< HTTP/1.1 200 OK
```

민감한 값이 포함되므로 실제 로그를 공유하거나 저장할 때는 `Authorization`, 사용자 이름, IP, URL 파라미터를 가려야 한다.

## 응답별로 판단한 기준

| 결과 | 먼저 확인할 것 |
| --- | --- |
| `curl: (60)` | 인증서 신뢰 체인, 호스트 이름 |
| 첫 `401`만 보임 | challenge를 받은 뒤 재요청이 나가는지 |
| 두 번째 `401` | 계정, 비밀번호, URI, nonce, 알고리즘 |
| `curl: (94)` | curl 인증 백엔드와 Windows SSPI 빌드 여부 |
| `200 OK` | API 본문과 실제 장비 상태까지 확인 |

`200 OK`도 최종 성공을 보장하지 않는다. HTTP 요청이 처리됐다는 뜻과 비밀번호가 실제로 바뀌었다는 뜻은 다르다. 상태를 바꾸는 API라면 새 비밀번호로 다시 인증해 최종 상태를 확인해야 한다.

## 정리

Digest 오류를 볼 때는 아래 순서를 기억하면 된다.

```text
TLS 연결 확인
→ 첫 401의 WWW-Authenticate 확인
→ Basic/Digest 구분
→ Digest challenge 값 확인
→ Authorization 재요청 확인
→ HTTP 응답 본문 확인
→ 상태 변경 API라면 실제 상태 재검증
```

이 흐름을 실제로 적용해 Windows libcurl을 다시 빌드했던 기록은 다음 글에 정리했다.

- [카메라 비밀번호 변경 API가 curl (94)로 실패한 문제]({% post_url 2025-06-25-error-digest-인증-실패-원인-분석-및-해결-흐름 %})
