---
title: "[Note] ONVIF 카메라 비밀번호 변경 가능성을 미리 진단하는 도구를 만든 기록"
date: 2026-05-14 20:00:00 +0900
categories: [note]
tags: [go, onvif, soap, ws-discovery, camera, diagnostics]
layout: single
---

같은 ONVIF 카메라라도 펌웨어와 인증 방식에 따라 `GetUsers`와 `SetUser` 동작이 달랐다. 실제 비밀번호 변경 작업을 실행하기 전에 여러 카메라의 지원 여부와 실패 지점을 한 번에 확인할 수 있는 Windows 진단 도구를 Go로 만들었다.

목표는 단순한 포트 스캐너가 아니었다.

```text
ONVIF 연결 가능 여부
→ 지원하는 인증 방식
→ 사용자 관리 API 지원 여부
→ SetUser 응답 형태
→ 기존 서비스 로직에서의 예상 결과
→ ONVIF 표준 기준 결과
```

## 장비를 찾는 방법을 두 가지로 나눴다

같은 네트워크의 ONVIF 장비는 WS-Discovery 멀티캐스트로 찾고, 멀티캐스트가 막힌 환경은 IP 범위를 직접 스캔하도록 했다.

```go
type Target struct {
    IP    net.IP
    XAddr string
}
```

WS-Discovery 응답에서 device service 주소를 얻을 수 있으면 그 값을 사용했다. IP 범위 스캔에서는 후보 주소를 만들어 짧은 timeout으로 `GetDeviceInformation`을 호출했다.

스캔 대상이 많을 때 UI가 멈추지 않도록 worker 수를 제한한 goroutine pool로 처리했다.

```go
jobs := make(chan string)
results := make(chan CameraResult)

for i := 0; i < workerCount; i++ {
    go func() {
        for target := range jobs {
            results <- probe(target)
        }
    }()
}
```

무제한 goroutine을 만들지 않고 timeout과 동시 접속 수를 화면에서 조정할 수 있게 했다.

## 인증 방식은 정해 두지 않고 순서대로 확인했다

ONVIF SOAP 요청에서 쓰는 WS-Security와 HTTP 인증은 서로 다른 계층이다. 장비마다 지원 범위가 달라 다음 순서로 폴백했다.

```text
WS-Security PasswordDigest
→ WS-Security PasswordText
→ HTTP Digest
→ HTTP Basic
```

각 시도의 인증 방식, HTTP status, `WWW-Authenticate`, SOAP Fault를 별도 기록했다. 최종 성공만 남기면 다른 펌웨어에서 왜 실패했는지 다시 알 수 없기 때문이다.

```go
type AuthAttempt struct {
    Mode       AuthMode
    HTTPStatus int
    SOAPFault  string
    Err        error
}
```

HTTP Digest는 인증 없는 요청에서 challenge를 받은 뒤 `realm`, `nonce`, `qop`, 알고리즘을 파싱해 `Authorization` 헤더를 만들었다.

## `GetUsers`와 `SetUser`를 따로 봤다

ONVIF device service가 응답한다고 사용자 관리 기능까지 지원하는 것은 아니다.

```text
GetDeviceInformation 성공
GetUsers 미지원
SetUser 미지원
```

같은 조합도 가능했다. 그래서 다음 결과를 별도 필드로 저장했다.

- device service 접근 가능
- `GetUsers` 성공 여부와 사용자 목록
- `SetUser` 성공 여부
- `ActionNotSupported`, `NotAuthorized`, 비밀번호 정책 오류

## 진단용 SetUser는 현재 값으로 호출했다

지원 여부를 확인하려고 실제 새 비밀번호를 넣으면 진단 도구가 장비 상태를 바꾸게 된다. 그래서 조회한 사용자 등급과 현재 비밀번호를 사용해 같은 값으로 `SetUser`를 보내는 no-op 방식으로 구현했다.

```go
func probeSetUser(user User, currentPassword string) Attempt {
    request := SetUserRequest{
        Username:  user.Username,
        Password:  currentPassword,
        UserLevel: user.UserLevel,
    }
    return callSetUser(request)
}
```

다만 이 방식이 모든 장비에서 완전히 무해하다고 단정할 수는 없다.

- 같은 비밀번호 재설정을 거부하는 펌웨어가 있다.
- 비밀번호 변경 시 세션을 초기화하는 장비가 있을 수 있다.
- 현재 비밀번호가 새 정책을 만족하지 않으면 약한 비밀번호 오류가 날 수 있다.

그래서 도구의 결과는 “실제 새 비밀번호 변경 성공 보장”이 아니라 호환성 진단으로 표시했다.

## 판정을 두 가지로 나눈 이유

기존 애플리케이션에는 일부 비표준 카메라를 처리하기 위한 응답 보정 로직이 있었다. 예를 들어 gSOAP이 오류를 반환해도 원본 버퍼의 HTTP 상태를 보고 성공으로 취급하는 경로가 있었다.

이 로직만 그대로 따라가면 장비가 표준에 맞는지 알 수 없다. 반대로 표준만 보면 현재 서비스에서 실제로 어떻게 판정되는지 알 수 없다.

그래서 한 장비에 대해 두 결과를 같이 만들었다.

```go
type CameraResult struct {
    ExistingLogicVerdict Verdict
    ONVIFVerdict         Verdict
}
```

| 판정 | 의미 |
| --- | --- |
| 기존 로직 기준 | 현재 애플리케이션의 분기에서 어떤 결과가 날지 예상 |
| ONVIF 기준 | SOAP Fault와 응답 태그를 표준 의미로 해석 |

두 결과가 다르면 상세 화면에 원본 HTTP status, SOAP Fault, 응답 일부를 보여줬다. 이 차이가 장비 호환성 코드를 다시 볼 근거가 된다.

## 실패를 코드로 분류했다

“변경 불가” 한 종류로만 저장하지 않고 원인을 안정적인 코드로 나눴다.

```text
NO_ONVIF        device service에 도달하지 못함
AUTH_FAIL       모든 인증 방식 실패
NO_GETUSERS     GetUsers 미지원
NO_SETUSER      SetUser 미지원
USER_NOT_FOUND  대상 사용자를 찾지 못함
WEAK_PW         비밀번호 정책 거부
DEVICE_LOCKED   반복 실패로 장비 잠금
NETWORK_ERR     timeout 또는 연결 종료
UNKNOWN_FAULT   분류하지 못한 SOAP 오류
```

UI 문구가 바뀌어도 CSV의 코드로 같은 유형을 모을 수 있게 했다.

## 결과를 다시 분석할 수 있게 남겼다

Fyne 기반 Windows UI에는 다음 기능을 넣었다.

- WS-Discovery와 IP 범위 스캔
- 전체 공통 자격증명과 IP별 CSV 자격증명
- timeout과 worker 수 설정
- 장비별 인증 시도 내역
- `GetUsers`/`SetUser` 상세 결과
- 두 판정의 차이 표시
- CSV 내보내기

CSV에는 최종 판정뿐 아니라 이유 코드와 응답 힌트를 같이 저장했다. 스캔이 끝난 뒤 같은 실패 유형을 묶어 볼 수 있게 하기 위해서다.

## 구현하면서 남긴 한계

- no-op `SetUser` 결과와 실제 새 비밀번호 변경 결과가 다를 수 있다.
- IP 범위 스캔은 비표준 ONVIF 포트를 모두 찾지 못할 수 있다.
- 장비별 잠금 정책 때문에 인증 폴백 횟수를 무작정 늘리면 안 된다.
- 비표준 응답을 성공으로 보정하는 기존 로직은 false positive가 생길 수 있다.
- 진단 결과가 가능이어도 실제 적용 전에는 소수 장비로 상태 변경과 재로그인을 확인해야 한다.

## 정리

이 도구를 만들며 구현한 핵심은 다음과 같다.

```text
장비 탐색
→ 인증 방식 폴백
→ GetUsers/SetUser 기능별 확인
→ 상태를 바꾸지 않는 범위의 진단 호출
→ 기존 로직과 ONVIF 기준의 이중 판정
→ 실패 코드와 원본 근거 저장
```

카메라 모델명만 보고 지원 여부를 판단하지 않고, 실제 프로토콜 응답을 같은 형식으로 모아 비교할 수 있게 만든 진단 도구다. 실제 변경을 대신하지는 않지만, 어떤 장비를 먼저 확인해야 하는지와 어느 계층에서 실패하는지를 빠르게 좁힐 수 있게 했다.
