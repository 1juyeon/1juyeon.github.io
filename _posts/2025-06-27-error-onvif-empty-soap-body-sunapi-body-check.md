---
title: "[Error] 200 OK인데 gSOAP SOAP_NO_TAG가 나는 이유"
date: 2025-06-27 20:00:00 +0900
categories: [error]
tags: [onvif, soap, sunapi, gsoap, http, digest, troubleshooting]
layout: single
review_required: true
---

# 200 OK인데 gSOAP SOAP_NO_TAG가 나는 이유

## 한 줄 결론

`HTTP/1.1 200 OK`가 왔다고 해서 ONVIF 명령이 정상 응답까지 반환했다는 뜻은 아니다.

내가 봤던 케이스는 **HTTP Body는 존재하지만, 그 안에 들어 있는 SOAP Body가 비어 있는 응답**이었다. 그래서 gSOAP은 내가 보낸 ONVIF 요청에 대응되는 응답 태그를 찾지 못했고, `SOAP_NO_TAG(6)` 오류를 반환했다.

정확히 표현하면 다음과 같다.

```text
틀린 표현: HTTP Body가 비어 있다.
맞는 표현: HTTP Body에는 SOAP XML이 있지만, SOAP Body 안이 비어 있다.
```

이 차이를 구분하지 못하면 `인증 실패인지`, `네트워크 실패인지`, `장비가 명령을 처리하지 않은 것인지`, `응답 XML만 불완전한 것인지`를 계속 헷갈리게 된다.

HTTP Digest, ONVIF, SUNAPI, HTTP Body와 SOAP Body 차이는 아래 개념 정리 글에 따로 묶었다.

- [HTTP Digest와 네트워크 카메라 API 통신 구조 이해하기]({% post_url 2025-06-28-note-http-digest-camera-api-communication %})

## 문제 상황

네트워크 장비에 ONVIF 기반 사용자 정보 변경 요청을 보냈다. 요청 자체는 HTTP 레벨에서 정상 응답을 받았다.

응답 상태는 `200 OK`였고, `Content-Type`도 SOAP XML이었다. 그런데 클라이언트 쪽 gSOAP에서는 `SOAP_NO_TAG(6)` 오류가 발생했다.

처음에는 다음처럼 생각하기 쉽다.

- HTTP 200이니까 성공 아닌가?
- Body가 없어서 파싱을 못 한 건가?
- Digest 인증이 실패한 건가?
- 요청이 실제로 적용되지 않은 건가?

하지만 실제 응답을 계층별로 나눠 보면 원인이 더 명확해진다.

## 실제로 봐야 하는 응답 구조

응답을 단순화하면 다음과 같은 형태였다.

```http
HTTP/1.1 200 OK
Content-Type: application/soap+xml; Charset=utf-8
Content-Length: 323
Connection: close

<?xml version="1.0" encoding="UTF-8"?>
<SOAP-ENV:Envelope xmlns:SOAP-ENV="http://www.w3.org/2003/05/soap-envelope">
  <SOAP-ENV:Body></SOAP-ENV:Body>
</SOAP-ENV:Envelope>
```

여기서 먼저 계층을 분리해서 봐야 한다.

```text
HTTP 응답
├─ HTTP Status: 200 OK
├─ HTTP Header
│  ├─ Content-Type: application/soap+xml
│  └─ Content-Length: 323
└─ HTTP Body
   └─ SOAP Envelope
      └─ SOAP Body
         └─ 비어 있음
```

중요한 점은 `Content-Length: 323`이다.

이 값은 **SOAP Body 내부가 323바이트라는 뜻이 아니다.** HTTP Header가 끝난 뒤에 오는 **HTTP Body 전체 길이**가 323바이트라는 뜻이다.

즉, 323바이트 안에는 다음이 모두 포함된다.

- XML 선언
- SOAP Envelope 태그
- XML namespace 선언
- 비어 있는 SOAP Body 태그

그래서 `Content-Length`가 0이 아니어도 SOAP Body 내부는 비어 있을 수 있다.

## SOAP Body가 비어 있다는 건 어느 부분으로 알 수 있나

핵심은 이 부분이다.

```xml
<SOAP-ENV:Body></SOAP-ENV:Body>
```

시작 태그와 종료 태그 사이에 아무 내용이 없다.

줄바꿈해서 보면 더 명확하다.

```xml
<SOAP-ENV:Body>
</SOAP-ENV:Body>
```

정상적인 ONVIF 응답이라면 이 사이에 요청에 대응되는 응답 태그가 있어야 한다. 예를 들어 사용자 정보 변경 요청이었다면 다음처럼 응답 태그가 들어오는 것이 정상에 가깝다.

```xml
<SOAP-ENV:Body>
  <tds:SetUserResponse/>
</SOAP-ENV:Body>
```

`SetUserResponse` 내부에 값이 없을 수는 있다. 하지만 태그 자체는 중요하다.

이 태그는 gSOAP 입장에서 다음 의미를 가진다.

```text
내가 보낸 SetUser 요청에 대한 정상 SOAP 응답을 받았다.
```

반대로 실제 응답처럼 Body 안에 아무 태그가 없으면 gSOAP은 어떤 요청에 대한 응답인지 확정할 수 없다.

## 정상 응답과 실제 응답 비교

| 구분 | 실제 응답 | 정상적으로 기대한 응답 |
| --- | --- | --- |
| HTTP 상태 | `200 OK` | `200 OK` |
| HTTP Body | 있음 | 있음 |
| Content-Type | `application/soap+xml` | `application/soap+xml` |
| SOAP Envelope | 있음 | 있음 |
| SOAP Body | 있음, 하지만 내부가 비어 있음 | 있음 |
| 요청 대응 태그 | 없음 | `SetUserResponse` 같은 응답 태그 존재 |
| SOAP Fault | 없음 | 없음 |
| gSOAP 해석 | 기대한 태그를 못 찾아 실패 | 정상 응답으로 해석 가능 |

여기서 중요한 판단은 이것이다.

```text
HTTP 통신은 성공했다.
하지만 ONVIF/SOAP 응답 형식은 완전하지 않다.
```

`200 OK`는 HTTP 서버가 요청을 받아 응답했다는 뜻이다. ONVIF 명령이 규격에 맞는 SOAP 응답까지 반환했다는 뜻은 아니다.

## gSOAP에서 SOAP_NO_TAG(6)가 나는 흐름

gSOAP은 내가 호출한 함수에 맞는 응답 태그를 기다린다.

예를 들어 내부적으로는 이런 흐름이 된다.

```text
1. 클라이언트가 ONVIF SetUser 요청을 보냄
2. 장비가 HTTP 200 OK를 반환함
3. gSOAP이 HTTP Body의 SOAP Envelope를 읽음
4. gSOAP이 SOAP Body를 읽음
5. gSOAP이 SetUserResponse 태그를 찾으려고 함
6. 그런데 SOAP Body가 바로 종료됨
7. 더 읽을 태그가 없음
8. SOAP_NO_TAG(6) 발생
```

따라서 `SOAP_NO_TAG`를 다음처럼 해석하면 안 된다.

- 네트워크 연결 실패
- Digest 인증 실패
- HTTP Body가 아예 없음
- 반드시 명령 실행 실패

더 정확한 해석은 다음이다.

```text
gSOAP이 기대한 SOAP 응답 태그를 찾지 못했다.
```

이 오류는 인증이나 네트워크보다 **응답 XML 구조 문제**에 가깝다.

## 이 응답만으로 확정할 수 있는 것과 없는 것

이 응답만 보고 확실히 말할 수 있는 것은 다음이다.

| 확실히 알 수 있는 것 | 이유 |
| --- | --- |
| HTTP 응답은 왔다 | `HTTP/1.1 200 OK`가 있음 |
| HTTP Body는 존재한다 | `Content-Length: 323`이고 XML이 있음 |
| SOAP 형식 응답이다 | `Content-Type: application/soap+xml`이고 `SOAP-ENV:Envelope`가 있음 |
| SOAP Body는 비어 있다 | `<SOAP-ENV:Body></SOAP-ENV:Body>` 사이에 태그가 없음 |
| SOAP Fault는 아니다 | `<SOAP-ENV:Fault>`가 없음 |
| 정상 응답 태그도 없다 | `SetUserResponse` 같은 태그가 없음 |

반대로 이 응답만으로는 다음을 확정할 수 없다.

| 확정할 수 없는 것 | 추가 확인 방법 |
| --- | --- |
| 실제 설정이 적용됐는지 | 변경된 값으로 재조회 |
| 비밀번호가 실제 변경됐는지 | 변경된 비밀번호로 재로그인 |
| 장비 내부 처리가 완전히 성공했는지 | 후속 조회 API 또는 실제 동작 확인 |

내가 봤던 케이스에서는 변경된 정보로 실제 로그인이 가능했다. 이 사실까지 합치면 결론은 이렇게 정리할 수 있다.

```text
인증: 성공
요청 전달: 성공
장비 내부 처리: 성공으로 보임
응답 생성: ONVIF 규격상 불완전
gSOAP 파싱: SOAP_NO_TAG로 실패
```

즉, 장비는 명령을 처리했지만, ONVIF 클라이언트가 기대하는 `<SetUserResponse/>` 형태의 응답을 돌려주지 않은 상황으로 판단했다.

## ONVIF와 SUNAPI는 Body를 보는 위치가 다르다

이 부분도 같이 헷갈리기 쉽다.

ONVIF는 HTTP Body 안에 SOAP XML이 들어간다. 그래서 Body를 두 단계로 봐야 한다.

```text
ONVIF 응답
└─ HTTP Body
   └─ SOAP Envelope
      └─ SOAP Body
         └─ ONVIF 응답 태그
```

반면 SUNAPI는 일반적인 HTTP API에 가깝다. SOAP Envelope나 SOAP Body를 한 번 더 찾지 않는다.

```text
SUNAPI 응답
└─ HTTP Body
   └─ OK, Parameter=Value, JSON 등 실제 결과
```

그래서 ONVIF에서의 “SOAP Body가 비었다”와 SUNAPI에서의 “HTTP Body가 비었다”는 서로 다른 말이다.

## SUNAPI 기준으로 Body를 확인하는 방법

SUNAPI에서는 HTTP Header가 끝나는 빈 줄 다음부터가 바로 HTTP Body다.

예를 들어 이런 응답이 있다.

```http
HTTP/1.1 200 OK
Content-Type: text/plain
Content-Length: 2

OK
```

이 경우 Body는 `OK`다.

설정 조회 응답이라면 이런 식일 수 있다.

```http
HTTP/1.1 200 OK
Content-Type: text/plain
Content-Length: 48

SomeSetting.Enable=True
SomeSetting.Mode=Auto
```

JSON 응답이라면 다음처럼 볼 수 있다.

```http
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 22

{"Response":"Success"}
```

반대로 SUNAPI에서 HTTP Body가 비어 있다고 말하려면 보통 이런 형태여야 한다.

```http
HTTP/1.1 200 OK
Content-Type: text/plain
Content-Length: 0
Connection: close

```

Header 뒤에 아무 내용이 없고, `Content-Length`도 0이다.

이때는 다음 표현이 맞다.

```text
SUNAPI HTTP Body가 비어 있다.
```

하지만 다음 표현은 맞지 않다.

```text
SUNAPI SOAP Body가 비어 있다.
```

SUNAPI에는 애초에 SOAP Body 계층이 없기 때문이다.

## ONVIF 응답인지 SUNAPI 응답인지 구분하는 기준

가장 정확한 것은 요청 URL과 `Content-Type`, 그리고 Body 구조를 같이 보는 것이다.

| 확인 지점 | ONVIF 가능성이 높은 경우 | SUNAPI 가능성이 높은 경우 |
| --- | --- | --- |
| 요청 URL | `/onvif/...` 계열 | `/stw-cgi/...` 계열 |
| Content-Type | `application/soap+xml`, `text/xml` | `text/plain`, `application/json` |
| Body 구조 | `<SOAP-ENV:Envelope>` 포함 | `OK`, `Parameter=Value`, JSON 등 |
| 확인해야 할 Body | SOAP Body 내부 | HTTP Header 뒤의 Body 전체 |
| 성공 판단 | 요청 대응 응답 태그 또는 Fault 확인 | 응답 값, 상태 값, 재조회 결과 확인 |

이번 응답은 `application/soap+xml`이고 `SOAP-ENV:Envelope`, `SOAP-ENV:Body`가 있으므로 SUNAPI 평문 응답이 아니라 ONVIF/SOAP 응답으로 보는 것이 맞다.

## 다음에 같은 로그를 보면 볼 순서

같은 대화를 다시 반복하지 않으려면 아래 순서대로 보면 된다.

```text
1. HTTP Status를 본다.
   - 200인지, 401인지, 500인지 먼저 확인한다.

2. Header와 Body의 경계를 찾는다.
   - HTTP Header가 끝나는 빈 줄 다음부터가 HTTP Body다.

3. Content-Length를 확인한다.
   - 0이면 HTTP Body 자체가 없을 가능성이 높다.
   - 0이 아니면 HTTP Body는 존재한다.

4. Content-Type을 확인한다.
   - application/soap+xml이면 ONVIF/SOAP 구조로 본다.
   - text/plain 또는 application/json이면 SUNAPI처럼 HTTP Body를 바로 본다.

5. SOAP 응답이면 SOAP Body 안을 확인한다.
   - <SOAP-ENV:Body>와 </SOAP-ENV:Body> 사이를 본다.

6. 요청에 대응되는 응답 태그를 찾는다.
   - 예: SetUser 요청이면 SetUserResponse가 있는지 본다.

7. 응답 태그가 없으면 SOAP Fault가 있는지도 본다.
   - Fault도 없고 응답 태그도 없으면 비정상 응답 구조다.

8. HTTP 200인데 응답 구조가 이상하면 실제 적용 여부를 별도로 확인한다.
   - 재조회
   - 재로그인
   - 실제 동작 확인
```

내가 앞으로 기억해야 할 핵심은 이것이다.

```text
HTTP 200은 통신 레벨 성공이다.
ONVIF 성공 여부는 SOAP Body 안의 응답 태그까지 봐야 한다.
SUNAPI 성공 여부는 HTTP Body의 실제 값이나 후속 조회 결과로 판단한다.
```

## 이 이슈에서 내가 정리한 판단 기준

이 문제를 처리하면서 가장 중요했던 것은 오류 메시지를 바로 믿기보다, 응답을 계층별로 쪼개서 보는 것이었다.

내가 한 판단은 다음 순서였다.

| 단계 | 내가 본 것 | 판단 |
| --- | --- | --- |
| 1 | `HTTP/1.1 200 OK` | 네트워크 요청 자체는 응답을 받음 |
| 2 | `Content-Length: 323` | HTTP Body가 완전히 빈 것은 아님 |
| 3 | `Content-Type: application/soap+xml` | ONVIF/SOAP 응답으로 봐야 함 |
| 4 | `<SOAP-ENV:Body></SOAP-ENV:Body>` | SOAP Body 내부가 비어 있음 |
| 5 | 응답 태그 없음 | gSOAP이 기대한 태그를 찾지 못함 |
| 6 | 변경값으로 실제 확인 | 요청 처리는 됐지만 응답 생성이 불완전한 케이스로 판단 |

이렇게 정리해 두면 다음에 `200 OK + SOAP_NO_TAG` 조합을 봤을 때 바로 인증 실패나 네트워크 실패로 가지 않고, 먼저 SOAP Body 내부 구조를 확인할 수 있다.

## 정리

이번 문제는 `HTTP 200`과 `API 성공`을 같은 의미로 보면 안 된다는 걸 보여 준 케이스였다.

ONVIF에서는 HTTP Body 안에 SOAP Envelope가 있고, 그 안에 다시 SOAP Body가 있다. 그래서 `HTTP Body가 있다/없다`와 `SOAP Body가 비어 있다/아니다`를 분리해서 봐야 한다.

이번 응답은 HTTP Body에 SOAP XML이 존재했다. 하지만 SOAP Body 안에 `SetUserResponse` 같은 정상 응답 태그가 없었다. 그래서 gSOAP은 기대한 태그를 찾지 못했고 `SOAP_NO_TAG(6)`를 반환했다.

SUNAPI에서는 SOAP Body를 찾지 않는다. Header 뒤의 HTTP Body를 그대로 보고, `OK`, `Parameter=Value`, JSON, 또는 `Content-Length: 0` 여부로 판단한다.

결론적으로 다음 한 문장을 기억하면 된다.

```text
ONVIF는 HTTP Body 안의 SOAP Body를 보고, SUNAPI는 HTTP Body 자체를 본다.
```
