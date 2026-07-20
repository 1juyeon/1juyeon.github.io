---
title: "[Note] HTTP Digest와 네트워크 카메라 API 통신 구조 이해하기"
date: 2025-06-28 20:00:00 +0900
categories: [note]
tags: [http, digest, camera-api, onvif, sunapi, soap, troubleshooting]
layout: single
review_required: true
---

# HTTP Digest와 네트워크 카메라 API 통신 구조 이해하기

## 왜 이 글을 정리했나

네트워크 카메라나 장비 API를 테스트하다 보면 `curl`, Digest 인증, HTTPS, ONVIF, SUNAPI, SOAP, gSOAP 같은 단어가 한꺼번에 나온다.

처음에는 로그를 봐도 어디가 인증 문제이고, 어디가 통신 문제이고, 어디가 응답 파싱 문제인지 헷갈렸다. 특히 `HTTP/1.1 200 OK`를 받았는데도 클라이언트에서는 오류가 나는 상황이 있었다.

이 글은 다음에 같은 상황을 다시 만났을 때, 대화를 다시 반복하지 않고도 이해할 수 있도록 정리한 개념 노트다.

핵심은 이것이다.

```text
HTTP는 통신의 바깥 형식이다.
Digest는 HTTP 요청을 인증하는 방식이다.
ONVIF는 HTTP Body 안에 SOAP XML을 넣어 통신한다.
SUNAPI는 HTTP Body에 결과를 바로 담는 HTTP API에 가깝다.
```

즉, `200 OK` 하나만 보고 성공/실패를 판단하면 안 된다. HTTP 레벨, 인증 레벨, API 레벨, 응답 Body 레벨을 나눠서 봐야 한다.

## 전체 구조 먼저 보기

네트워크 카메라 API 통신을 크게 보면 다음과 같다.

```text
클라이언트
  └─ HTTP 요청
      ├─ TLS/HTTPS로 암호화될 수 있음
      ├─ Basic 또는 Digest 인증 정보를 포함할 수 있음
      └─ ONVIF 또는 SUNAPI 명령을 보냄

카메라/장비
  └─ HTTP 응답
      ├─ HTTP Status
      ├─ HTTP Header
      └─ HTTP Body
          ├─ ONVIF라면 SOAP XML
          └─ SUNAPI라면 text/json/OK/설정값
```

각 레이어의 역할을 나누면 더 이해하기 쉽다.

| 레이어 | 역할 | 예시 |
| --- | --- | --- |
| TCP/IP | 장비와 연결 가능한지 | IP, Port, 방화벽, 라우팅 |
| TLS/HTTPS | 통신 구간 암호화 | 인증서, TLS 버전, cipher |
| HTTP | 요청/응답 형식 | `GET`, `POST`, `200 OK`, Header, Body |
| HTTP 인증 | 요청한 사용자가 누구인지 확인 | Basic, Digest |
| API 프로토콜 | 어떤 기능을 실행할지 | ONVIF, SUNAPI |
| 응답 파싱 | 결과를 어떻게 해석할지 | SOAP XML, text, JSON |

문제가 생겼을 때는 이 레이어 중 어디에서 실패했는지 분리해야 한다.

## HTTP는 가장 바깥쪽 약속이다

HTTP 응답은 보통 이렇게 생겼다.

```http
HTTP/1.1 200 OK
Content-Type: text/plain
Content-Length: 2

OK
```

구조를 나누면 다음과 같다.

```text
HTTP 응답
├─ Status Line
│  └─ HTTP/1.1 200 OK
├─ Header
│  ├─ Content-Type: text/plain
│  └─ Content-Length: 2
├─ 빈 줄
└─ Body
   └─ OK
```

HTTP Header와 Body는 **빈 줄**을 기준으로 나뉜다. Header가 끝난 뒤에 나오는 내용이 HTTP Body다.

`Content-Length`는 HTTP Body 전체 길이를 의미한다.

```text
Content-Length: 0   -> HTTP Body가 없음
Content-Length: 2   -> HTTP Body가 2바이트 있음
Content-Length: 323 -> HTTP Body가 323바이트 있음
```

하지만 ONVIF처럼 HTTP Body 안에 다시 SOAP XML이 들어가면 이야기가 한 단계 더 복잡해진다. HTTP Body가 있다고 해서 SOAP Body 안에 실제 응답 태그가 있다는 뜻은 아니기 때문이다.

## HTTPS와 Digest 인증은 역할이 다르다

HTTPS와 Digest 인증은 둘 다 보안과 관련이 있지만 담당하는 일이 다르다.

| 구분 | 역할 |
| --- | --- |
| HTTPS/TLS | 통신 구간을 암호화하고 서버 인증서를 검증 |
| HTTP Digest | HTTP 요청을 보낸 사용자가 올바른 계정인지 인증 |

쉽게 말하면 HTTPS는 **통신 터널을 안전하게 만드는 것**이고, Digest는 **그 터널 안에서 요청한 사용자가 누구인지 확인하는 것**이다.

따라서 HTTPS가 성공했다고 Digest 인증도 성공한 것은 아니다. 반대로 Digest 인증 정보가 맞아도 TLS 인증서 문제로 연결이 실패할 수 있다.

## HTTP Digest 인증 흐름

Digest 인증은 비밀번호 원문을 그대로 보내지 않는다. 서버가 먼저 인증에 필요한 값을 알려주고, 클라이언트가 그 값을 이용해 해시를 계산해서 다시 요청한다.

흐름은 보통 다음과 같다.

```text
1. 클라이언트가 인증 없이 요청
2. 서버가 401 Unauthorized 반환
3. 서버가 WWW-Authenticate 헤더로 Digest 조건 전달
4. 클라이언트가 realm, nonce, qop, method, uri, password 등을 이용해 response 해시 계산
5. 클라이언트가 Authorization: Digest ... 헤더를 붙여 재요청
6. 서버가 같은 방식으로 계산한 값과 비교
7. 맞으면 200 OK 또는 API 결과 반환
```

예시로 보면 더 쉽다.

첫 번째 요청이다.

```http
GET /api/path HTTP/1.1
Host: <device-host>
```

서버는 인증이 필요하다고 응답한다.

```http
HTTP/1.1 401 Unauthorized
WWW-Authenticate: Digest realm="Device",
  nonce="abc123",
  qop="auth",
  algorithm=MD5
Content-Length: 0
```

여기서 `401 Unauthorized`는 반드시 최종 실패를 의미하지 않는다. Digest 인증에서는 **인증 방식을 협상하기 위한 정상적인 첫 응답**일 수 있다.

클라이언트는 서버가 준 값들을 이용해 다시 요청한다.

```http
GET /api/path HTTP/1.1
Host: <device-host>
Authorization: Digest username="<user>",
  realm="Device",
  nonce="abc123",
  uri="/api/path",
  qop=auth,
  response="<calculated-hash>"
```

서버가 계산한 해시와 클라이언트가 보낸 해시가 맞으면 요청을 처리한다.

```http
HTTP/1.1 200 OK
Content-Type: text/plain
Content-Length: 2

OK
```

Digest 인증에서 기억할 점은 다음이다.

- 비밀번호 원문을 직접 보내지 않는다.
- 서버가 준 `nonce`를 이용해 요청마다 해시를 만든다.
- 첫 번째 `401`은 실패가 아니라 인증 challenge일 수 있다.
- 최종 성공 여부는 재요청 이후의 HTTP Status와 API 응답을 봐야 한다.

## Basic 인증과 Digest 인증 차이

장비 API를 테스트할 때 Basic과 Digest를 헷갈릴 수 있다.

| 구분 | Basic | Digest |
| --- | --- | --- |
| 비밀번호 전달 방식 | `user:password`를 Base64 인코딩 | 비밀번호를 이용한 해시값 전달 |
| 서버의 첫 응답 | 바로 처리되거나 401 | 보통 401로 nonce 전달 |
| 보안성 | 낮음 | Basic보다 안전 |
| curl 옵션 | `-u user:pass` | `--digest -u user:pass` |
| 주의점 | HTTPS 없이 쓰면 위험 | 클라이언트/서버 알고리즘 호환성 확인 필요 |

Basic의 Base64는 암호화가 아니다. 단순 인코딩이라 누구든 디코딩할 수 있다. 그래서 Basic을 쓴다면 HTTPS가 반드시 필요하다.

Digest도 완벽한 보안 방식은 아니지만, 비밀번호 원문을 직접 보내지 않는다는 점에서 Basic보다 낫다.

## curl로 Digest 요청을 보낼 때 보는 것

Digest 인증 테스트는 보통 이렇게 한다.

```bash
curl --digest -u "<user>:<password>" -k "https://<device-host>/<api-path>"
```

옵션 의미는 다음과 같다.

| 옵션 | 의미 |
| --- | --- |
| `--digest` | HTTP Digest 인증 사용 |
| `-u` | 사용자명과 비밀번호 지정 |
| `-k` | TLS 인증서 검증 생략 |

`-k`는 테스트 환경에서 자체 서명 인증서를 쓰는 장비를 확인할 때 사용할 수 있다. 하지만 운영 환경이나 외부망에서는 신중해야 한다. 인증서 검증을 생략하면 중간자 공격을 막기 어렵기 때문이다.

문제 분석 중에는 `-v` 옵션을 붙이면 HTTP 흐름을 더 잘 볼 수 있다.

```bash
curl -v --digest -u "<user>:<password>" -k "https://<device-host>/<api-path>"
```

이때 봐야 하는 것은 다음이다.

- 첫 응답이 `401 Unauthorized`인지
- `WWW-Authenticate: Digest ...`가 있는지
- 재요청에 `Authorization: Digest ...`가 붙는지
- 최종 응답이 `200 OK`인지
- 최종 Body에 API 결과가 있는지

## ONVIF는 SOAP XML을 HTTP Body에 담는다

ONVIF 요청은 HTTP 위에서 SOAP XML을 주고받는다.

구조는 다음과 같다.

```text
ONVIF HTTP 응답
├─ HTTP Status
├─ HTTP Header
└─ HTTP Body
   └─ SOAP Envelope
      └─ SOAP Body
         └─ ONVIF 응답 태그
```

예를 들어 정상 응답은 이런 형태가 될 수 있다.

```http
HTTP/1.1 200 OK
Content-Type: application/soap+xml; Charset=utf-8
Content-Length: 180

<SOAP-ENV:Envelope xmlns:SOAP-ENV="http://www.w3.org/2003/05/soap-envelope">
  <SOAP-ENV:Body>
    <tds:SetUserResponse/>
  </SOAP-ENV:Body>
</SOAP-ENV:Envelope>
```

여기서 `HTTP Body`는 SOAP XML 전체다.

그리고 `SOAP Body`는 그 XML 안쪽의 이 부분이다.

```xml
<SOAP-ENV:Body>
  <tds:SetUserResponse/>
</SOAP-ENV:Body>
```

ONVIF에서는 HTTP Status뿐 아니라 SOAP Body 안의 응답 태그까지 봐야 한다.

## HTTP Body와 SOAP Body는 다르다

가장 헷갈렸던 부분이 이것이었다.

```text
HTTP Body != SOAP Body
```

HTTP Body는 HTTP Header가 끝난 뒤 나오는 전체 내용이다.

SOAP Body는 HTTP Body 안에 들어 있는 SOAP XML의 일부분이다.

예를 들어 다음 응답을 보자.

```http
HTTP/1.1 200 OK
Content-Type: application/soap+xml; Charset=utf-8
Content-Length: 323

<SOAP-ENV:Envelope xmlns:SOAP-ENV="http://www.w3.org/2003/05/soap-envelope">
  <SOAP-ENV:Body></SOAP-ENV:Body>
</SOAP-ENV:Envelope>
```

이 응답은 HTTP Body가 비어 있는 것이 아니다.

HTTP Body에는 이 XML이 들어 있다.

```xml
<SOAP-ENV:Envelope xmlns:SOAP-ENV="http://www.w3.org/2003/05/soap-envelope">
  <SOAP-ENV:Body></SOAP-ENV:Body>
</SOAP-ENV:Envelope>
```

하지만 SOAP Body는 비어 있다.

```xml
<SOAP-ENV:Body>
</SOAP-ENV:Body>
```

그래서 정확한 해석은 다음이다.

```text
HTTP Body는 있다.
하지만 SOAP Body 안에는 응답 태그가 없다.
```

이 차이를 구분해야 `Content-Length`가 있는데 왜 gSOAP이 오류를 냈는지 이해할 수 있다.

## gSOAP은 ONVIF 응답 태그를 기다린다

gSOAP은 SOAP XML을 읽어 C/C++ 코드에서 사용할 수 있게 해 주는 도구다. ONVIF 클라이언트를 구현할 때 gSOAP으로 생성된 함수를 사용하는 경우가 많다.

예를 들어 클라이언트가 `SetUser` 요청을 보냈다면 gSOAP은 응답에서 `SetUserResponse`에 해당하는 태그를 찾으려고 한다.

정상 응답이라면 다음처럼 읽힌다.

```xml
<SOAP-ENV:Body>
  <tds:SetUserResponse/>
</SOAP-ENV:Body>
```

그런데 실제 응답이 다음과 같으면 문제가 된다.

```xml
<SOAP-ENV:Body>
</SOAP-ENV:Body>
```

gSOAP 입장에서는 다음 흐름이 된다.

```text
SOAP Envelope는 찾음
SOAP Body도 찾음
요청에 맞는 Response 태그를 찾으려 함
그런데 Body가 바로 끝남
읽을 태그가 없음
SOAP_NO_TAG(6) 발생
```

따라서 `SOAP_NO_TAG`는 보통 다음 의미로 이해하는 것이 좋다.

```text
gSOAP이 기대한 SOAP 태그를 찾지 못했다.
```

이 오류만 보고 바로 Digest 인증 실패라고 단정하면 안 된다. HTTP 인증은 이미 통과했을 수 있고, 실제 명령도 적용됐을 수 있다. 다만 응답 XML이 ONVIF 클라이언트가 기대한 형태가 아닐 수 있다.

## SUNAPI는 SOAP Body를 찾지 않는다

SUNAPI는 ONVIF와 다르게 HTTP Body에 결과를 바로 담는 형태로 이해하면 쉽다.

예를 들어 성공 응답이 다음처럼 올 수 있다.

```http
HTTP/1.1 200 OK
Content-Type: text/plain
Content-Length: 2

OK
```

또는 설정값 조회 결과가 이렇게 올 수 있다.

```http
HTTP/1.1 200 OK
Content-Type: text/plain
Content-Length: 48

SomeSetting.Enable=True
SomeSetting.Mode=Auto
```

JSON 형태라면 다음처럼 볼 수 있다.

```http
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 22

{"Response":"Success"}
```

SUNAPI에서는 `SOAP-ENV:Envelope`나 `SOAP-ENV:Body`를 찾지 않는다. Header 뒤에 오는 HTTP Body 자체를 읽으면 된다.

그래서 SUNAPI에서는 이렇게 말해야 한다.

```text
HTTP Body가 비어 있다.
HTTP Body에 OK가 있다.
HTTP Body에 설정값이 있다.
HTTP Body에 JSON 결과가 있다.
```

반대로 다음 표현은 맞지 않다.

```text
SUNAPI의 SOAP Body가 비어 있다.
```

SUNAPI에는 SOAP Body 계층이 없기 때문이다.

## ONVIF와 SUNAPI 구분 기준

응답을 보기 전에 요청 URL과 Content-Type을 같이 보면 더 빠르다.

| 기준 | ONVIF | SUNAPI |
| --- | --- | --- |
| 요청 URL | `/onvif/...` 계열 | `/stw-cgi/...` 계열 |
| 응답 Content-Type | `application/soap+xml`, `text/xml` | `text/plain`, `application/json` |
| HTTP Body 내용 | SOAP XML | OK, 설정값, JSON |
| 확인 위치 | SOAP Body 내부 | HTTP Body 전체 |
| 클라이언트 구현 | gSOAP 같은 SOAP 클라이언트 | HTTP 클라이언트, curl 등 |

ONVIF는 표준 SOAP 메시지를 해석해야 하고, SUNAPI는 HTTP 응답 Body 값을 직접 해석하는 쪽에 가깝다.

## 200 OK를 받았을 때 성공 여부를 판단하는 방법

`200 OK`는 HTTP 레벨의 성공이다.

하지만 장비 API에서는 보통 다음을 더 확인해야 한다.

| 확인 항목 | 봐야 하는 이유 |
| --- | --- |
| HTTP Status | 서버가 요청을 받았는지 확인 |
| Content-Type | Body를 어떤 방식으로 해석할지 결정 |
| Content-Length | HTTP Body 존재 여부 확인 |
| Body 내용 | 실제 API 결과 확인 |
| ONVIF 응답 태그 | SOAP 요청에 대응되는 정상 응답인지 확인 |
| Fault 또는 Error 필드 | API 레벨 실패 여부 확인 |
| 재조회 결과 | 실제 설정 적용 여부 확인 |

특히 설정 변경 API에서는 `200 OK`를 받아도 실제 적용 여부를 다시 확인하는 습관이 필요하다.

예를 들어 다음처럼 확인할 수 있다.

```text
1. set/change 요청 전송
2. HTTP 응답 확인
3. API 응답 Body 확인
4. view/get 요청으로 변경값 재조회
5. 인증 정보 변경이라면 변경된 값으로 재로그인
```

응답 형식이 불완전한 장비도 있을 수 있기 때문에, 최종 판단은 실제 상태 확인까지 포함하는 것이 안전하다.

## 문제를 만났을 때 볼 순서

카메라 API 통신 문제를 보면 아래 순서로 분리해서 보는 것이 좋다.

```text
1. 연결 문제인가?
   - IP, Port, 방화벽, timeout 확인

2. TLS 문제인가?
   - 인증서, TLS 버전, cipher, curl -k 사용 여부 확인

3. HTTP 인증 문제인가?
   - 401, WWW-Authenticate, Authorization 헤더 확인

4. API 프로토콜 문제인가?
   - ONVIF인지 SUNAPI인지 요청 URL과 Content-Type으로 확인

5. Body 파싱 문제인가?
   - ONVIF라면 SOAP Body 안의 응답 태그 확인
   - SUNAPI라면 HTTP Body의 text/json 값 확인

6. 실제 적용 문제인가?
   - 재조회 또는 재로그인으로 상태 확인
```

이 순서대로 보면 모든 오류를 `인증 실패`나 `통신 실패`로 뭉뚱그리지 않게 된다.

## 내가 기억하려고 정리한 문장들

다음 문장만 기억해도 같은 로그를 다시 읽을 때 훨씬 덜 헷갈린다.

```text
401은 Digest 인증에서 정상적인 challenge일 수 있다.
200 OK는 HTTP 레벨 성공이지 API 결과 성공을 보장하지 않는다.
Content-Length는 HTTP Body 전체 길이다.
HTTP Body와 SOAP Body는 다르다.
ONVIF는 SOAP Body 안의 응답 태그를 확인한다.
SUNAPI는 HTTP Body 자체를 확인한다.
gSOAP 오류는 HTTP 오류가 아니라 SOAP 파싱 오류일 수 있다.
설정 변경은 응답만 보지 말고 재조회나 재로그인으로 실제 적용을 확인한다.
```

## 관련 오류 분석

개념을 정리한 뒤, 실제로 `200 OK`인데 SOAP Body가 비어 있어서 gSOAP이 `SOAP_NO_TAG(6)`를 반환한 케이스는 아래 글에 따로 정리했다.

- [200 OK인데 gSOAP SOAP_NO_TAG가 나는 이유]({% post_url 2025-06-27-error-onvif-empty-soap-body-sunapi-body-check %})

## 정리

네트워크 카메라 API 통신은 한 번에 보면 복잡하지만, 레이어를 나누면 읽을 수 있다.

먼저 HTTP 응답의 Status, Header, Body를 분리한다. 그다음 인증이 Basic인지 Digest인지 확인한다. 그리고 API가 ONVIF인지 SUNAPI인지 구분한다.

ONVIF라면 HTTP Body 안에 SOAP XML이 있고, 그 안의 SOAP Body와 응답 태그를 봐야 한다. SUNAPI라면 HTTP Body에 들어 있는 `OK`, 설정값, JSON을 바로 보면 된다.

결국 핵심은 다음이다.

```text
Digest는 인증 방식이고,
ONVIF/SUNAPI는 API 통신 방식이고,
HTTP Body/SOAP Body는 응답을 읽는 위치다.
```

이 셋을 분리해서 보면 `200 OK인데 왜 오류가 나지?` 같은 상황도 차분하게 분석할 수 있다.
