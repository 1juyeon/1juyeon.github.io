---
title: "[Error] Windows 11 24H2에서 원격 계정 비밀번호 변경이 거부된 문제"
date: 2025-06-20 20:00:00 +0900
categories: [error]
tags: [windows, netapi32, netuserchangepassword, netusersetinfo, jni]
layout: single
---

Java에서 JNI로 C++ DLL을 호출해 원격 Windows 계정의 비밀번호를 변경하는 기능이 있었다. 기존 환경에서는 동작했지만 Windows 11 24H2 대상에서 `NetUserChangePassword`가 `ERROR_ACCESS_DENIED(5)`를 반환했다.

```text
IPC connection: success
NetUserChangePassword: ERROR_ACCESS_DENIED (5)
```

IPC 인증이 성공했기 때문에 계정이나 네트워크만의 문제로 보기 어려웠다. 권한과 정책을 여러 방향으로 확인한 뒤, 기능의 의미에 맞는 Windows API로 변경했다.

## 기존 구현

기존 네이티브 코드는 관리자 자격증명으로 대상 PC의 `IPC$` 연결을 만든 뒤, 변경 대상 계정의 기존 비밀번호와 새 비밀번호를 `NetUserChangePassword`에 전달했다.

```cpp
DWORD result = NetUserChangePassword(
    remoteHost,
    targetUser,
    oldPassword,
    newPassword
);
```

이 함수는 사용자가 자신의 기존 비밀번호를 알고 새 비밀번호로 **변경(change)** 하는 의미에 가깝다.

하지만 내가 구현한 기능은 관리자가 원격 장비 계정의 비밀번호를 일괄 관리하는 것이었다. 이 경우에는 기존 비밀번호로 본인 변경을 요청하는 것보다 관리자 권한으로 값을 **재설정(reset)** 하는 동작이 더 맞았다.

## 먼저 확인한 범위

오류 5만 보고 바로 API를 바꾸지 않고 다음을 확인했다.

```text
1. 대상 IP와 445 포트 접근
2. 관리자 계정으로 IPC$ 연결 성공 여부
3. 대상 사용자 존재 여부와 계정 상태
4. 서비스 실행 계정과 수동 실행 계정 차이
5. Windows 보안 이벤트 로그
6. 비밀번호 정책 위반 여부
```

IPC 연결 단계와 비밀번호 변경 단계를 로그에서 분리한 것이 중요했다. 연결이 실패한 경우와 `NetUserChangePassword`가 권한을 거부한 경우는 대응이 다르기 때문이다.

## 확인 과정에서 제외한 것

SMB 1.0 활성화나 보안 정책 완화도 후보로 검토했지만 최종 해결책으로 사용하지 않았다. 오래된 프로토콜을 켜거나 UAC 원격 제한을 넓게 푸는 방식은 계정 변경 하나를 위해 시스템 보안을 낮출 수 있다.

또한 24H2의 어떤 내부 정책 변화가 직접 원인인지는 공개 문서와 코드만으로 확정하지 못했다. 따라서 “24H2가 이 API를 막았다”라고 단정하지 않고, 내가 재현한 환경에서 기존 호출이 오류 5를 반환했다는 사실까지만 근거로 삼았다.

## `NetUserSetInfo` level 1003으로 변경

최종적으로 관리자 자격증명으로 IPC 연결을 만든 뒤 `USER_INFO_1003` 구조체를 사용해 비밀번호를 설정하도록 바꿨다.

```cpp
USER_INFO_1003 info{};
info.usri1003_password = newPassword;

DWORD result = NetUserSetInfo(
    remoteHost,
    targetUser,
    1003,
    reinterpret_cast<LPBYTE>(&info),
    nullptr
);
```

API의 의미 차이는 다음과 같다.

| API | 필요한 정보 | 작업 성격 |
| --- | --- | --- |
| `NetUserChangePassword` | 기존 비밀번호와 새 비밀번호 | 사용자의 비밀번호 변경 |
| `NetUserSetInfo(1003)` | 관리자 권한과 새 비밀번호 | 관리자의 비밀번호 재설정 |

기존 Java/JNI 인터페이스에서도 더 이상 필요하지 않은 기존 비밀번호 인자를 제거하고, 관리자 계정과 대상 계정, 새 비밀번호만 전달하도록 맞췄다.

## 오류 코드도 다시 매핑했다

새 API에서 반환할 수 있는 값을 기능 결과로 나눴다.

```cpp
switch (result) {
case NERR_Success:
    return SUCCESS;
case ERROR_ACCESS_DENIED:
    return ACCESS_DENIED;
case NERR_UserNotFound:
    return USER_NOT_FOUND;
case NERR_PasswordTooShort:
case ERROR_INVALID_PARAMETER:
    return PASSWORD_POLICY_ERROR;
default:
    return UNKNOWN_ERROR;
}
```

사용자 없음, 권한 부족, 비밀번호 정책 실패를 같은 오류로 보여주지 않도록 구분했다.

## 비동기 IPC 연결 대기 시간도 보완했다

기존 코드는 IPC 연결을 별도 thread에서 수행하면서 약 0.5초만 기다렸다. 네트워크가 조금만 느려도 연결 실패처럼 처리될 수 있었다. 수정하면서 대기 시간을 늘리고, 다음 세 단계를 별도 로그로 남겼다.

```text
IPC 연결
→ NetUserSetInfo 호출
→ 새 비밀번호 확인
```

마지막에는 새 비밀번호로 IPC 연결이 되는지 다시 확인해 실제 계정 상태와 내부 저장값이 맞는지 검증했다.

## 정리

이 문제에서 한 일은 Windows 정책을 우회한 것이 아니다.

- IPC 연결 실패와 계정 변경 권한 실패를 분리했다.
- 관리자 일괄 변경이라는 기능 성격에 맞춰 change API를 reset API로 교체했다.
- JNI 인자와 네이티브 함수 시그니처를 함께 수정했다.
- 반환 코드를 권한, 사용자, 비밀번호 정책 오류로 구분했다.
- 새 비밀번호로 실제 접속되는지 사후 검증했다.

특정 OS 버전의 내부 원인을 추측해 보안 설정을 완화하는 것보다, 호출 목적과 Windows API의 권한 모델이 맞는지 다시 보는 것이 실제 해결에 더 가까웠다.
