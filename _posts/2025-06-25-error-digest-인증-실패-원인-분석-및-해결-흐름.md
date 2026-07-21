---
title: "[Error] Windows curl에서 Digest 인증이 (94)로 실패한 문제"
date: 2025-06-25 20:00:00 +0900
categories: [error]
layout: single
---

# Windows curl에서 Digest 인증이 (94)로 실패한 문제

## 상황

HTTPS API에 Digest 인증으로 요청을 보내는 테스트를 하고 있었다.

브라우저에서는 계정으로 접속이 됐고, API도 Digest 인증을 요구하고 있었다. 그래서 처음에는 아래처럼 curl 명령어만 맞추면 될 것이라고 생각했다.

```bash
curl -vk --digest -u "<user>:<password>" "https://<device-host>/<api-path>"
```

그런데 Windows 환경에서 아래와 비슷한 오류가 나왔다.

```text
schannel: InitializeSecurityContext failed: SEC_E_QOP_NOT_SUPPORTED
curl: (94) An authentication function returned an error
```

처음에는 계정이 틀렸거나 URL이 틀린 줄 알았다. 그런데 확인할수록 단순 계정 문제가 아니라, Windows curl의 인증 처리 방식과 Digest 알고리즘 조합 문제에 가까웠다.

## 먼저 확인한 것

바로 원인을 단정하지 않고, 실패 위치를 나눠서 봤다.

| 확인한 것 | 이유 |
| --- | --- |
| URL과 포트가 맞는지 | 연결 자체가 되는지 먼저 확인 |
| `-k` 옵션 유무 | 인증서 오류와 Digest 오류를 분리하기 위해 |
| `--digest` 문법 | `-digest`처럼 잘못 쓰지 않았는지 확인 |
| `WWW-Authenticate` 헤더 | 서버가 실제로 Digest를 요구하는지 확인 |
| 계정 문자열 따옴표 | 특수문자가 셸에서 깨지는지 확인 |
| `curl --version` | 어떤 TLS/인증 백엔드로 빌드된 curl인지 확인 |

이 과정에서 `curl: (60)`은 인증서 검증 실패이고, `401 Unauthorized`는 Digest challenge일 수 있고, `(94)`는 인증 처리 함수 쪽 실패일 수 있다는 식으로 나눠서 보게 됐다.

## Digest에서 401은 항상 실패가 아니다

Digest 인증은 서버가 먼저 인증 조건을 알려주고, 클라이언트가 그 조건으로 해시를 계산해서 다시 요청하는 방식이다.

흐름은 아래처럼 진행된다.

```text
1. 클라이언트가 요청
2. 서버가 401 + WWW-Authenticate: Digest ... 응답
3. 클라이언트가 nonce, realm, uri, method 등을 이용해 response 해시 계산
4. Authorization: Digest ... 헤더를 붙여 재요청
5. 서버가 계산값을 비교
```

그래서 아래 응답은 단순 실패가 아니라 정상적인 협상 시작일 수 있다.

```http
HTTP/1.1 401 Unauthorized
WWW-Authenticate: Digest realm="Device", algorithm=SHA-256, qop="auth", nonce="..."
```

내가 실제로 봐야 했던 것은 401 자체가 아니라, curl이 이 challenge를 받은 뒤 다시 `Authorization: Digest ...` 헤더를 붙여 요청하는지였다.

## 문제가 된 조합

핵심은 Windows에서 사용하는 curl 빌드 구성이었다.

`Schannel`은 Windows 기본 TLS 처리 기능이다. HTTPS 연결에서 인증서 검증이나 TLS handshake를 담당한다.

`SSPI`는 Windows 인증 처리 인터페이스다. curl이 SSPI를 사용하도록 빌드되어 있으면 Digest 인증 일부를 Windows 쪽에 맡길 수 있다.

문제는 서버가 Digest 인증에서 `SHA-256`, `qop="auth"` 같은 조건을 제시할 때, Windows curl 조합에서 이 처리가 기대대로 되지 않을 수 있다는 점이었다.

```text
서버: Digest + SHA-256 + qop=auth 제시
Windows curl: SSPI/Schannel 조합으로 인증 처리 시도
결과: SEC_E_QOP_NOT_SUPPORTED, curl (94)
```

여기서 `qop`는 Quality of Protection의 줄임말이다. Digest 인증에서 어떤 보호 수준으로 계산할지 나타내는 값이다. 보통 API에서는 `auth`가 많이 보인다.

## 내가 시도한 것

문제를 좁히기 위해 여러 방식으로 같은 요청을 다시 확인했다.

| 시도 | 확인한 내용 |
| --- | --- |
| 인증 없이 `curl -vk` | 서버가 Digest challenge를 주는지 확인 |
| `-k` 없이 요청 | 인증서 오류와 인증 오류를 분리 |
| `--digest` 붙여 요청 | Digest 인증 흐름 진입 여부 확인 |
| 계정 문자열 따옴표 처리 | 특수문자 깨짐 가능성 제거 |
| Postman 요청 | Windows SSPI 영향을 받지 않는 클라이언트와 비교 |
| WSL/Linux curl 요청 | Windows Schannel/SSPI 조합을 벗어나서 비교 |
| OpenSSL 기반 curl 확인 | `curl --version`에서 TLS 백엔드 확인 |

이 과정에서 단순히 명령어 하나를 고치는 문제가 아니라, "어떤 curl이 인증을 처리하고 있는가"가 중요하다는 걸 알게 됐다.

## 해결 방향

최종적으로는 Windows 기본 curl에만 의존하지 않는 쪽으로 정리했다.

가능한 대응은 아래였다.

| 방법 | 의미 |
| --- | --- |
| Postman으로 비교 테스트 | 계정/API 문제가 아닌 클라이언트 구현 문제인지 확인 |
| WSL 또는 Linux curl 사용 | Windows 인증 처리 경로를 배제 |
| OpenSSL 기반 curl 사용 | Schannel/SSPI 영향을 줄이고 curl 내부 처리 기준으로 확인 |
| 직접 빌드한 libcurl 사용 | 애플리케이션에 붙일 DLL까지 같은 기준으로 맞춤 |

내가 선택한 방향은 OpenSSL 기반 curl/libcurl을 직접 빌드해서 테스트 기준을 고정하는 것이었다.

## 이후 확인 기준

이 문제 이후로 Digest 인증 오류를 볼 때는 아래 순서로 확인했다.

```text
1. curl -vk로 TLS 연결과 401 challenge 확인
2. WWW-Authenticate의 Digest 조건 확인
3. algorithm, qop, realm, nonce 값 확인
4. curl --version으로 Schannel/SSPI/OpenSSL 여부 확인
5. Postman 또는 WSL curl로 같은 요청 비교
6. 필요하면 OpenSSL 기반 curl/libcurl로 테스트 환경 고정
```

이 순서가 생기고 나서는 `(94)` 오류를 계정 오류로만 보지 않게 됐다.

## 관련해서 이어진 글

- [curl로 HTTPS API 인증 방식을 먼저 확인한 기록]({% post_url 2025-06-20-note-curl-api-test %})
- [Digest SHA-256 테스트를 위해 Windows용 curl/libcurl을 직접 빌드한 기록]({% post_url 2025-06-30-note-curl-openssl-직접-빌드-정리 %})
- [HTTP Digest와 네트워크 카메라 API 통신 구조 이해하기]({% post_url 2026-07-21-note-http-digest-camera-api-communication %})

## 정리

이 오류는 단순히 "비밀번호가 틀렸다"로 볼 문제가 아니었다.

내가 한 일은 TLS 오류, 인증서 오류, Digest challenge, Windows curl 빌드 구성을 하나씩 분리해서 확인한 것이다. 혼자 테스트하면서 제일 힘들었던 부분은 실패 메시지가 전부 비슷하게 보인다는 점이었다. 그래서 각 오류가 어느 계층에서 난 것인지 나누는 기준을 만들었고, 그 결과 OpenSSL 기반 curl/libcurl을 직접 빌드하는 방향까지 이어졌다.
