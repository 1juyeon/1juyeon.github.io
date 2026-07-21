---
title: "[Error] SOAP 응답은 실패했지만 카메라 비밀번호는 바뀐 문제"
date: 2026-06-06 22:00:00 +0900
categories: [error]
tags: [onvif, soap, gsoap, password, state-consistency]
layout: single
---

ONVIF `SetUser`로 카메라 비밀번호를 변경하는 기능에서 이상한 결과가 나왔다.

```text
비밀번호 변경 요청 결과: 실패
새 비밀번호 확인 결과: 성공
```

카메라는 실제로 새 비밀번호를 적용했지만, gSOAP은 응답에서 기대한 태그를 찾지 못해 오류를 반환했다. 애플리케이션은 최초 오류만 보고 실패 처리했고, 내부에 저장된 비밀번호는 예전 값으로 남았다. 다음 접속부터 카메라와 내부 데이터가 어긋나는 상태가 됐다.

## 로그 한 줄보다 실제 상태가 중요했다

처음에는 gSOAP 오류가 났으니 변경도 실패했다고 생각했다. 그러나 뒤에 실행한 비밀번호 검증 로그에서는 새 비밀번호가 맞다고 나왔다.

```text
SetUser result: 6
Check password result: new password is correct
```

이 두 결과를 같이 놓고 보면 다음처럼 해석해야 했다.

```text
요청 전송                성공했을 가능성이 높음
카메라 내부 상태 변경     성공
SOAP 응답 역직렬화         실패
애플리케이션의 최종 판정   잘못된 실패
```

상태 변경 API에서는 “응답을 정상적으로 읽었는가”와 “대상 상태가 실제로 바뀌었는가”가 항상 같지 않다. 연결이 응답 도중 끊기거나, 장비가 표준과 다른 XML을 반환해도 요청 자체는 이미 적용됐을 수 있다.

## 오류 코드 6의 이름부터 다시 확인했다

초기 조사에서는 응답 수신 중 연결이 끝난 것으로 보고 코드 6을 EOF 계열로 생각했다. 하지만 사용 중인 gSOAP의 `stdsoap2.h`를 직접 확인하면 의미가 다르다.

```cpp
#define SOAP_TAG_MISMATCH 3
#define SOAP_TYPE         4
#define SOAP_SYNTAX_ERROR 5
#define SOAP_NO_TAG       6
```

따라서 이 경우의 정확한 의미는 `SOAP_NO_TAG`였다. HTTP 본문은 왔지만 gSOAP이 기대한 `SetUserResponse` 태그를 찾지 못한 것이다.

오류 enum과 사용자 메시지도 “SOAP EOF”가 아니라 “예상한 응답 태그를 찾지 못함”으로 고쳤다. 숫자만 보고 의미를 붙이지 않고, 실제로 링크된 라이브러리 헤더를 기준으로 다시 확인한 부분이다.

## 실제 응답에서 빠진 것

정상적인 성공 응답이라면 SOAP Body 안에 요청에 대응하는 응답 요소가 있어야 한다.

```xml
<soap:Envelope>
  <soap:Body>
    <tds:SetUserResponse/>
  </soap:Body>
</soap:Envelope>
```

문제가 된 응답은 HTTP 200과 SOAP Envelope는 있었지만 Body 안의 응답 요소가 비어 있었다.

```xml
<soap:Envelope>
  <soap:Body></soap:Body>
</soap:Envelope>
```

그래서 HTTP 계층에서는 200이지만 gSOAP 역직렬화에서는 `SOAP_NO_TAG`가 됐다. 이 문제는 네트워크 연결 실패나 Digest 인증 실패와도 다르다.

## 최종 판정을 다시 만들었다

기존 흐름은 변경 API의 반환값을 먼저 성공/실패로 확정한 뒤 비밀번호를 확인했다. 나는 사후 검증 결과가 확실한 경우 최종 판정을 다시 정하도록 수정했다.

핵심 판정 흐름을 의사 코드로 줄이면 다음과 같다.

```java
int changeResult = camera.changePassword(oldPassword, newPassword);
PasswordState state = camera.checkPassword(oldPassword, newPassword);

if (state == PasswordState.NEW_PASSWORD_VALID) {
    changeResult = SUCCESS;
    saveNewPassword(newPassword);
} else if (state == PasswordState.OLD_PASSWORD_VALID) {
    changeResult = CHANGE_FAILED;
    keepOldPassword();
} else {
    changeResult = CHANGE_STATE_UNKNOWN;
    saveForRetry(oldPassword, newPassword);
}
```

관리 계정과 일반 계정은 저장 테이블이 달랐기 때문에 두 경로 모두 같은 판정 규칙을 적용했다.

## 세 상태로 나눈 이유

성공과 실패만으로 나누면 “둘 다 확인되지 않음”을 안전하게 처리할 수 없다.

| 사후 검증 | 최종 처리 |
| --- | --- |
| 새 비밀번호 인증 성공 | 변경 성공으로 보정하고 새 값 저장 |
| 기존 비밀번호 인증 성공 | 변경 실패, 기존 값 유지 |
| 둘 다 확인 실패 | 통신/잠금 가능성을 포함한 상태 불명, 재확인 대상 |

세 번째 상태에서 새 비밀번호를 버리면 실제로 변경된 장비의 자격증명을 잃을 수 있다. 반대로 새 값만 저장하면 변경되지 않은 장비에 접근하지 못한다. 그래서 두 후보를 보존하고 이후 재검증할 수 있게 했다.

## 수정 후 확인한 흐름

```text
1. SetUser가 정상 응답하는 카메라
2. SetUser가 SOAP_NO_TAG를 내지만 새 비밀번호는 적용되는 카메라
3. 비밀번호 정책 위반으로 기존 비밀번호가 유지되는 경우
4. 통신 문제로 두 비밀번호를 모두 확인할 수 없는 경우
5. 관리자 계정과 일반 계정의 저장값 갱신
```

특히 두 번째 경우에는 최초 API 결과가 실패여도 새 비밀번호 검증이 성공하면 내부 저장값과 후속 연동까지 성공 흐름으로 이어지는지 확인했다.

## 정리

이 문제에서 수정한 것은 단순한 오류 메시지가 아니다.

- gSOAP 코드 6을 실제 헤더 기준으로 `SOAP_NO_TAG`라고 구분했다.
- 변경 API의 반환값과 실제 장비 상태를 분리했다.
- 새 비밀번호 검증이 성공하면 최종 결과를 성공으로 보정했다.
- 기존 비밀번호만 맞으면 실패로 되돌렸다.
- 둘 다 확인되지 않는 경우를 별도 상태로 남겼다.

외부 장비를 바꾸는 기능에서는 요청의 성공 여부보다 최종 상태가 더 중요하다. 이번 수정 이후에는 최초 응답 하나가 아니라 사후 검증 결과를 기준으로 카메라와 내부 저장값을 맞추게 됐다.

응답 XML과 gSOAP 오류의 관계는 다음 글에 더 자세히 정리했다.

- [HTTP 200인데 gSOAP SOAP_NO_TAG가 발생한 응답 분석]({% post_url 2026-07-21-error-http-200-gsoap-soap-no-tag %})
