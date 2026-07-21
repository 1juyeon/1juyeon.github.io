---
title: "[Note] curl로 HTTPS API 인증 방식을 먼저 확인한 기록"
date: 2025-06-20 10:00:00 +0900
categories: [note]
tags: [curl, api, 인증, digest, basic]
layout: single
---

# curl로 HTTPS API 인증 방식을 먼저 확인한 기록

## 상황

장비나 외부 API를 연동할 때, 문서만 보고 바로 구현하면 생각보다 자주 막힌다.

특히 HTTPS API는 실패 원인이 여러 단계로 나뉜다.

- 네트워크가 안 되는지
- TLS 인증서에서 막히는지
- Basic 인증을 요구하는지
- Digest 인증을 요구하는지
- 계정 정보가 틀린 것인지
- API 경로나 파라미터가 틀린 것인지

처음부터 애플리케이션 코드에 붙이면 어느 단계에서 실패했는지 보기 어렵다. 그래서 나는 먼저 `curl -v`로 서버가 어떤 인증 방식을 요구하는지 확인하는 순서를 잡았다.

## curl을 먼저 쓴 이유

`curl`은 HTTP 요청을 명령어로 바로 보내볼 수 있는 도구다.

애플리케이션 코드보다 먼저 curl로 확인하면, 최소한 아래 내용을 빠르게 분리할 수 있다.

| 확인할 것 | curl로 보는 방법 |
| --- | --- |
| 서버까지 연결되는지 | 응답 코드나 연결 오류 확인 |
| TLS 인증서에서 막히는지 | `(60)`, certificate 관련 메시지 확인 |
| 인증 방식이 무엇인지 | `WWW-Authenticate` 헤더 확인 |
| Basic/Digest 중 무엇을 요구하는지 | `Basic`, `Digest` 문자열 확인 |
| 계정 정보가 적용되는지 | 인증 옵션을 바꿔가며 응답 비교 |

이 단계에서 중요한 건 API 성공보다 **실패 위치를 나누는 것**이었다.

## 1. 인증 없이 먼저 요청했다

처음에는 일부러 인증 정보를 넣지 않고 요청했다.

```bash
curl -vk "https://<device-host>/<api-path>"
```

`-v`는 요청과 응답 헤더를 자세히 출력한다.
`-k`는 테스트 환경에서 서버 인증서 검증을 임시로 건너뛰는 옵션이다. 운영 환경에서 무조건 쓰는 옵션은 아니다.

인증이 필요한 API라면 보통 아래처럼 `401 Unauthorized`가 나온다.

```http
HTTP/1.1 401 Unauthorized
WWW-Authenticate: Digest realm="Device", nonce="...", qop="auth"
```

여기서 `401`만 보고 실패라고 끝내면 안 된다. 이 응답은 서버가 "이 방식으로 인증해서 다시 요청해라"라고 알려주는 단계일 수 있다.

## 2. Basic과 Digest를 구분했다

`WWW-Authenticate` 헤더를 보면 서버가 요구하는 인증 방식을 알 수 있다.

```http
WWW-Authenticate: Basic realm="Device"
```

이렇게 나오면 Basic 인증이다.

```http
WWW-Authenticate: Digest realm="Device", nonce="...", qop="auth"
```

이렇게 나오면 Digest 인증이다.

Basic은 `아이디:비밀번호` 값을 Base64로 인코딩해서 보내는 방식이다. Base64는 암호화가 아니라 단순 인코딩이라서, HTTP에서는 사용하면 안 되고 HTTPS가 전제되어야 한다.

Digest는 비밀번호 원문을 바로 보내지 않고, 서버가 준 `nonce` 같은 값을 섞어서 해시를 만들어 보내는 방식이다. Basic보다 안전하지만, 클라이언트 라이브러리나 서버가 지원하는 알고리즘에 따라 호환성 문제가 날 수 있다.

## 3. 인증 방식에 맞게 다시 요청했다

Basic 인증이면 아래처럼 요청했다.

```bash
curl -vk -u "<user>:<password>" "https://<device-host>/<api-path>"
```

Digest 인증이면 `--digest`를 붙여서 요청했다.

```bash
curl -vk --digest -u "<user>:<password>" "https://<device-host>/<api-path>"
```

여기서 계정 문자열은 항상 따옴표로 감쌌다. 비밀번호에 `!`, `^`, `&` 같은 문자가 있으면 Windows CMD나 PowerShell에서 다르게 해석될 수 있기 때문이다.

## 4. 실패 메시지를 단계별로 나눴다

같은 실패처럼 보여도 의미가 다르다.

| 결과 | 내가 본 의미 |
| --- | --- |
| `curl: (6)` | 호스트 이름을 찾지 못함 |
| `curl: (7)` | 대상 포트로 연결 실패 |
| `curl: (60)` | 서버 인증서 검증 실패 |
| `401 Unauthorized` + `WWW-Authenticate` | 인증 방식 확인 가능, 재요청 필요 |
| `401 Unauthorized`가 계속 반복 | 계정 정보, 인증 방식, Digest 계산값 확인 필요 |
| `curl: (94)` | Windows 인증 처리 또는 Digest 호환성 문제 가능성 |

이렇게 정리해두면 "안 된다"라는 말 대신 "TLS에서 막혔다", "HTTP 인증까지는 갔다", "Digest 처리에서 막힌다"처럼 말할 수 있다.

## 내가 실제로 남긴 확인 순서

```text
1. curl -vk로 연결과 인증 요구 헤더 확인
2. WWW-Authenticate에서 Basic/Digest 구분
3. Basic이면 -u 옵션만 사용
4. Digest이면 --digest -u 옵션 사용
5. 인증서 오류와 인증 오류를 분리
6. 그래도 실패하면 curl --version으로 TLS/인증 백엔드 확인
```

처음에는 `curl` 명령어 하나로만 보면 될 줄 알았는데, 실제로는 `-v` 로그를 읽는 것이 더 중요했다.

특히 `401 Unauthorized`가 항상 최종 실패는 아니라는 점을 알게 된 뒤부터는, 인증 문제를 볼 때 먼저 헤더를 확인하게 됐다.

## 관련해서 이어진 글

- [Digest SHA-256 테스트를 위해 Windows용 curl/libcurl을 직접 빌드한 기록]({% post_url 2025-06-30-note-curl-openssl-직접-빌드-정리 %})
- [HTTP Digest와 네트워크 카메라 API 통신 구조 이해하기]({% post_url 2026-07-21-note-http-digest-camera-api-communication %})

## 정리

이 기록은 curl 사용법을 외우기 위한 글이라기보다, HTTPS API 연동에서 실패 위치를 먼저 나누기 위해 남긴 것이다.

내가 한 일은 API를 바로 코드에 붙이기 전에, 서버가 요구하는 인증 방식과 TLS 실패 여부를 명령어로 분리해본 것이다. 이 순서를 만들어두니 이후 Digest 인증 실패나 TLS 라이브러리 문제를 볼 때도 원인 후보를 훨씬 빨리 줄일 수 있었다.
