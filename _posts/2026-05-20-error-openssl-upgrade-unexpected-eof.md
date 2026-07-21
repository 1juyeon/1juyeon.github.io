---
title: "[Error] OpenSSL 업그레이드 후 장비의 TLS 종료를 오류로 처리한 문제"
date: 2026-05-20 20:00:00 +0900
categories: [error]
tags: [openssl, tls, gsoap, native, compatibility]
layout: single
---

OpenSSL을 3.x 계열로 올린 뒤, gSOAP으로 HTTPS 통신하는 일부 장비에서 `unexpected eof while reading` 오류가 발생했다. 요청과 응답 내용보다 TLS 연결이 끝나는 방식에서 문제가 생긴 경우였다.

## EOF가 왜 TLS 오류가 되는가

정상적인 TLS 연결 종료에서는 상대가 `close_notify`라는 알림을 보낸다. 이 알림은 "암호화된 데이터 전송이 여기서 정상적으로 끝났다"는 뜻이다.

하지만 일부 장비는 HTTP 응답을 보낸 다음 `close_notify` 없이 TCP 연결부터 닫는다. 예전 라이브러리에서는 넘어가던 동작이 업그레이드 후에는 예기치 않은 EOF로 더 엄격하게 보고될 수 있다.

```text
정상 종료: HTTP 응답 -> TLS close_notify -> TCP 종료
문제 장비: HTTP 응답 -> TCP 종료
```

여기서 EOF는 파일의 끝뿐 아니라 스트림에서 더 읽을 데이터가 없다는 뜻으로도 쓰인다.

## 적용 위치를 먼저 찾았다

오류 문자열만 보고 모든 TLS 코드에 옵션을 넣지는 않았다. HTTPS 장비 연결을 만드는 gSOAP 프록시의 TLS 컨텍스트 생성 지점을 찾았다. 기존 코드는 TLS 초기화가 성공하면 허용 버전을 TLS 1.2로 고정하고 있었다.

```cpp
if (soap_ssl_client_context(
        proxy.soap,
        SOAP_SSL_NO_AUTHENTICATION,
        nullptr, nullptr, nullptr, nullptr, nullptr) == SOAP_OK) {
    SSL_CTX_set_min_proto_version(proxy.ctx, TLS1_2_VERSION);
    SSL_CTX_set_max_proto_version(proxy.ctx, TLS1_2_VERSION);
}
```

이 컨텍스트에 OpenSSL이 제공하는 호환 옵션을 조건부로 추가했다.

```cpp
#ifdef SSL_OP_IGNORE_UNEXPECTED_EOF
SSL_CTX_set_options(proxy.ctx, SSL_OP_IGNORE_UNEXPECTED_EOF);
#endif
```

`#ifdef`를 둔 이유는 이전 OpenSSL 헤더로 빌드하는 환경에서도 해당 심볼 때문에 컴파일이 깨지지 않게 하기 위해서다. TLS 1.2 최소·최대 설정은 그대로 유지했다.

## 이 옵션이 해결하는 범위

`SSL_OP_IGNORE_UNEXPECTED_EOF`는 `close_notify` 없이 연결이 끊긴 상황을 정상 종료처럼 다루게 한다. 장비의 비표준 종료 방식과의 호환에는 도움이 되지만, 모든 통신 실패를 성공으로 바꾸는 옵션은 아니다.

다음 상황까지 성공으로 처리하면 안 된다.

- TLS handshake가 끝나지 않은 경우
- HTTP 상태 코드나 본문을 받지 못한 경우
- SOAP 응답에 필요한 결과 태그가 없는 경우
- 응답이 중간에서 잘려 XML을 완성하지 못한 경우

특히 이 옵션은 데이터가 중간에 잘린 경우와 정상 응답 뒤 비정상 종료를 TLS 계층에서 구별하기 어렵게 만들 수 있다. 따라서 상위 계층에서 HTTP 본문과 SOAP 결과를 끝까지 검증해야 한다.

## 확인한 순서

1. 문제가 라이브러리 교체 이후에만 발생하는지 비교했다.
2. TLS handshake와 실제 요청 전송 여부를 로그로 분리했다.
3. 사용 중인 OpenSSL 헤더에 옵션이 정의되어 있는지 확인했다.
4. 전체 애플리케이션이 아니라 장비용 gSOAP TLS 컨텍스트에만 옵션을 적용했다.
5. 기존 TLS 1.2 제한과 다른 오류 분기가 유지되는지 확인했다.

컴파일 성공만으로 장비 호환성을 검증했다고 보지는 않았다. 실제 장비에서 정상 응답, 인증 실패, 응답 누락을 다시 구분해보는 과정이 필요했다.

## 남은 판단 기준

이번 수정은 "구형 장비의 종료 방식도 받아준다"는 호환성 조치다. 보안 검증을 끄거나 SOAP 오류를 무조건 성공으로 덮은 것은 아니다.

TLS 계층에서 종료 오류를 완화했다면, 애플리케이션 계층에서는 다음 질문을 더 엄격하게 확인해야 한다.

```text
HTTP 응답을 끝까지 받았는가?
SOAP Body에 기대한 결과가 있는가?
상태 변경 요청이라면 실제 장비 상태도 바뀌었는가?
```

라이브러리 업그레이드에서 어려웠던 점은 빌드 오류보다 기존 장비의 비표준 동작을 어디까지 호환으로 인정할지 정하는 일이었다.
