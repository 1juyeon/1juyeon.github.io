---
title: "[Note] JNI 함수 인자 누락처럼 보일 때 추적하는 방법"
date: 2026-06-19 20:00:00 +0900
categories: [note]
tags: [jni, java, cpp, sdk, debugging]
layout: single
review_required: true
---

# JNI 함수 인자 누락처럼 보일 때 추적하는 방법

## 상황

Java에서 JNI 함수를 호출하는 코드를 보다가 "새 비밀번호가 네이티브 함수로 안 넘어가는 것 아닌가?"라는 의심이 생겼다.

네이티브 함수의 파라미터 이름은 `pw`처럼 일반적인 이름이고, Java 쪽 변수명은 `newPw`라면 더 헷갈릴 수 있다.

하지만 이름만 보고 판단하면 안 된다. 실제 호출 흐름을 끝까지 따라가야 한다.

## 추적 순서

먼저 Java 호출부를 본다.

```java
int result = NativeApi.changePassword(deviceId, userId, newPw);
```

다음으로 JNI 시그니처를 본다.

```cpp
JNIEXPORT jint JNICALL Java_package_NativeApi_changePassword(
    JNIEnv* env,
    jobject obj,
    jstring deviceId,
    jstring userId,
    jstring pw
) {
    // ...
}
```

여기서 `pw`라는 이름만 보면 현재 비밀번호처럼 보일 수 있다. 하지만 Java에서 세 번째 인자로 `newPw`를 넘겼다면, JNI의 `pw`는 새 비밀번호다.

중간에 변환 변수가 있으면 더 명확히 표시해두는 것이 좋다.

```cpp
const char* newPasswordValue = env->GetStringUTFChars(pw, nullptr);
```

이렇게 바꾸면 나중에 읽는 사람이 "이 pw가 무엇인지" 다시 추적하지 않아도 된다.

## 기존 비밀번호가 없는 구조도 있다

비밀번호 변경 함수라고 해서 항상 old password와 new password가 모두 필요한 것은 아니다.

이미 인증된 관리 세션을 통해 변경하는 구조라면 외부 API는 새 비밀번호만 받을 수 있다.

```text
관리 세션 로그인
-> 대상 장비 선택
-> 새 비밀번호 전달
-> 관리 서버가 변경 수행
```

이 경우 old password는 함수 인자로 필요하지 않다. 인증은 이미 앞 단계의 세션이 담당한다.

## 진짜 위험한 부분은 버전 호환성

JNI에서 더 조심해야 하는 것은 인자 이름보다 Java wrapper와 네이티브 DLL, 외부 SDK의 버전이 맞는지다.

예를 들어 어느 시점부터 외부 SDK 함수가 `newId` 같은 인자를 추가로 받도록 바뀌었다면, wrapper와 DLL이 서로 다른 버전일 때 문제가 난다.

확인할 것은 다음과 같다.

- Java 선언부의 인자 개수
- JNI 함수 시그니처
- 네이티브 내부에서 호출하는 SDK 함수 시그니처
- 배포된 DLL 버전
- 런타임 로그에 찍히는 wrapper 또는 SDK 버전

## 기록해두면 좋은 것

JNI 코드는 Java와 C/C++ 사이에서 이름과 타입이 쉽게 어긋난다. 그래서 다음 정보를 코드 근처나 운영 문서에 남겨두면 도움이 된다.

- Java 메서드 인자 순서
- JNI 변환 후 변수명
- 외부 SDK 함수 인자 순서
- old password가 필요한 구조인지, 세션 인증으로 대체되는 구조인지
- DLL 교체 시 같이 배포해야 하는 파일 목록

정리하면, JNI에서 인자가 빠진 것처럼 보일 때는 이름보다 호출 순서와 API 계약을 기준으로 확인해야 한다. 변수명이 애매하다면 코드를 바꾸는 것보다 먼저 흐름을 명확히 적어두는 것이 좋다.
