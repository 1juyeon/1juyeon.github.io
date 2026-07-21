---
title: "[Error] HTTP 200인데 gSOAP SOAP_NO_TAG가 발생한 응답 분석"
date: 2026-07-21 21:10:00 +0900
categories: [error]
tags: [onvif, soap, gsoap, http, xml]
layout: single
---

ONVIF 요청에서 `HTTP/1.1 200 OK`를 받았는데 gSOAP 반환값은 6이었다. 사용 중인 gSOAP 헤더에서 코드 6은 `SOAP_NO_TAG`다.

```cpp
#define SOAP_NO_TAG 6
```

원인은 HTTP Body가 아예 없어서가 아니라, SOAP Body 안에 gSOAP이 기다리던 응답 태그가 없었기 때문이다.

## HTTP Body와 SOAP Body는 다르다

실제 응답을 단순화하면 다음과 같았다.

```http
HTTP/1.1 200 OK
Content-Type: application/soap+xml
Content-Length: 300

<?xml version="1.0" encoding="UTF-8"?>
<soap:Envelope xmlns:soap="http://www.w3.org/2003/05/soap-envelope">
  <soap:Body></soap:Body>
</soap:Envelope>
```

여기에는 세 가지 서로 다른 “본문” 개념이 있다.

```text
HTTP Body
  XML 전체

SOAP Envelope
  <soap:Envelope> ... </soap:Envelope>

SOAP Body
  <soap:Body> ... </soap:Body> 안의 실제 응답 내용
```

`Content-Length`가 0보다 크다는 것은 HTTP Body에 XML이 있다는 뜻이다. SOAP Body 안에 `SetUserResponse`가 있다는 뜻은 아니다.

## 정상 응답과 비교했다

ONVIF `SetUser`의 정상 응답에서는 SOAP Body 안에 대응되는 응답 요소가 있어야 한다.

```xml
<soap:Envelope xmlns:soap="http://www.w3.org/2003/05/soap-envelope">
  <soap:Body>
    <tds:SetUserResponse
      xmlns:tds="http://www.onvif.org/ver10/device/wsdl"/>
  </soap:Body>
</soap:Envelope>
```

문제가 된 응답은 `Envelope`와 `Body`까지만 있고 `SetUserResponse`가 없었다.

```xml
<soap:Envelope xmlns:soap="http://www.w3.org/2003/05/soap-envelope">
  <soap:Body/>
</soap:Envelope>
```

gSOAP이 생성한 클라이언트 코드는 호출한 메서드에 맞는 응답 태그로 XML을 역직렬화한다. HTTP 200을 읽은 뒤 `SetUserResponse`를 찾지 못했기 때문에 `SOAP_NO_TAG`를 반환한 것이다.

## 계층별 결과를 나누면 모순이 아니다

```text
TCP/TLS           연결 및 전송 성공
HTTP              200 OK
XML               파싱 가능한 Envelope 수신
SOAP 메서드 응답   예상 태그 없음
gSOAP             SOAP_NO_TAG
카메라 내부 상태   별도 확인 필요
```

HTTP 200과 gSOAP 오류가 같이 나오는 것은 모순이 아니다. 서로 다른 계층의 결과다.

## 이 응답만으로 알 수 없는 것

빈 SOAP Body만 보고 카메라가 명령을 실행했는지는 확정할 수 없다.

- 요청은 적용했지만 응답을 잘못 만든 경우
- 요청을 무시하고 빈 성공 응답만 보낸 경우
- 내부 처리 도중 비표준 응답을 만든 경우

그래서 상태 변경 API는 후속 검증이 필요하다. 비밀번호 변경이라면 새 비밀번호와 기존 비밀번호를 각각 확인해야 한다.

```text
새 비밀번호만 성공  실제 변경됨
기존 비밀번호만 성공  변경되지 않음
둘 다 실패          통신 오류 또는 상태 불명
```

## ONVIF와 일반 HTTP API를 섞지 않았다

같은 카메라가 ONVIF와 제조사 HTTP API를 모두 제공할 수 있다. 두 방식은 HTTP 위에서 동작하지만 응답 해석 기준이 다르다.

| 구분 | ONVIF | 일반 HTTP API |
| --- | --- | --- |
| 대표 요청 | SOAP XML을 POST | URL/쿼리 또는 JSON 요청 |
| 응답 판단 | SOAP Body의 메서드 응답 또는 Fault | API가 정의한 JSON, XML, text |
| gSOAP 사용 | 사용 가능 | 보통 사용하지 않음 |

XML 응답이라고 모두 SOAP인 것도 아니다. `<Envelope><Body>...</Body></Envelope>` 구조와 SOAP namespace가 있어야 SOAP 응답으로 볼 수 있다.

## 다음에 같은 로그를 볼 순서

```text
1. 실제 요청 URL과 호출한 프로토콜 확인
2. HTTP status 확인
3. Content-Type과 전체 HTTP Body 저장
4. SOAP Envelope/Body 존재 확인
5. 호출한 메서드의 Response 태그 확인
6. gSOAP stdsoap2.h에서 숫자 코드 의미 확인
7. 상태 변경 요청이면 실제 대상 상태 재검증
```

오류 코드 이름은 기억에 의존하지 않고 현재 빌드가 사용하는 헤더에서 확인해야 한다. 같은 숫자에 임의로 EOF 같은 의미를 붙이면 이후 로그와 사용자 메시지도 모두 잘못된다.

## 정리

이 사례의 정확한 표현은 다음과 같다.

```text
HTTP Body는 있었다.
SOAP Envelope와 SOAP Body도 있었다.
하지만 SOAP Body 안에 기대한 SetUserResponse 태그가 없었다.
그래서 gSOAP은 SOAP_NO_TAG(6)를 반환했다.
```

이 응답 때문에 내부 비밀번호 저장값이 어긋났던 실제 수정은 다음 글에 정리했다.

- [SOAP 응답은 실패했지만 카메라 비밀번호는 바뀐 문제]({% post_url 2026-06-06-error-onvif-password-changed-soap-result-mismatch %})
