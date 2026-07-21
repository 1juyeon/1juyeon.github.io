---
title: "[Error] JNI 인자 하나가 빠져 URL과 계정 값이 한 칸씩 밀린 문제"
date: 2026-06-19 20:00:00 +0900
categories: [error]
tags: [jni, java, cpp, native, debugging]
layout: single
---

네이티브 DLL의 일부 변경만 이전 버전에 옮긴 뒤, 카메라 API 호출이 `CURL_FAIL`로 끝났다. 처음에는 URL이나 인증 방식 문제로 보였지만 로그의 인자 값이 서로 다른 자리에 찍히고 있었다.

```text
URL이 있어야 할 자리 -> 장비 ID
ID가 있어야 할 자리  -> 다른 문자열
비밀번호 자리          -> 앞 인자의 값
```

원인은 Java 선언과 C++ JNI 함수의 인자 개수가 달랐기 때문이다.

## 부분 반영 과정에서 계약이 달라졌다

Java 쪽 선언은 로그 추적용 장비 코드를 첫 번째 인자로 받도록 변경되어 있었다.

```java
native int checkPassword(
    String deviceCode,
    String url,
    String userId,
    String password
);
```

그런데 이전 버전용 DLL을 만들면서 C++ 함수에서는 이 인자를 제외했다.

```cpp
JNIEXPORT jint JNICALL Java_example_NativeApi_checkPassword(
    JNIEnv* env,
    jclass clazz,
    jstring url,
    jstring userId,
    jstring password
)
```

호출부는 네 개의 문자열을 넘기는데 네이티브 함수는 세 개로 해석한다. 결과적으로 `deviceCode`가 `url` 변수에 들어가고, 뒤의 인자도 차례대로 밀렸다.

## 왜 로딩 단계에서 바로 실패하지 않았는가

JNI는 오버로드 여부와 생성된 이름 규칙에 따라 함수를 연결한다. 이번 함수는 이름으로 연결은 되었기 때문에 `UnsatisfiedLinkError`가 먼저 발생하지 않았다. 대신 C++이 잘못된 위치의 `jstring`을 정상 문자열처럼 변환했고, 실제 HTTP 요청 단계에서 잘못된 URL이 사용됐다.

이 때문에 표면 증상은 JNI 오류가 아니라 curl 통신 실패였다.

```text
Java/C++ 시그니처 불일치
-> C++ 인자 해석 위치가 한 칸씩 이동
-> 잘못된 URL로 요청
-> curl 통신 실패
```

## 확인한 방법

함수 이름이나 변수명만 비교하지 않고 인자 계약을 세 층으로 나눠 확인했다.

1. Java `native` 선언의 타입과 순서
2. 생성된 JNI 헤더의 함수 시그니처
3. C++ 구현의 타입과 순서

그다음 각 인자를 변환한 직후의 로그를 마스킹된 형태로 비교했다.

```cpp
const char* deviceCodeValue = env->GetStringUTFChars(deviceCode, nullptr);
const char* urlValue = env->GetStringUTFChars(url, nullptr);
const char* userIdValue = env->GetStringUTFChars(userId, nullptr);
```

로그 값이 한 자리씩 이동한 형태였기 때문에 문자열 인코딩 문제보다 시그니처 불일치를 먼저 의심할 수 있었다.

## 수정 범위를 두 함수로 제한했다

Java 선언과 대조했을 때 장비 코드가 추가된 함수는 비밀번호 확인과 변경, 두 개뿐이었다. C++ 구현과 JNI 헤더에 첫 번째 인자를 다시 추가하고 내부 호출 순서를 맞췄다.

나머지 JNI 함수에는 같은 인자를 일괄 추가하지 않았다. 실제 Java 선언에 없는 인자를 넣으면 반대 방향의 불일치를 새로 만들기 때문이다.

## 부분 이식에서 배운 점

네이티브 변경을 다른 브랜치에 옮길 때는 관련 없어 보이는 인자를 함부로 제거하면 안 된다. 장비 코드는 외부 SDK의 새 기능이 아니라 Java 웹 애플리케이션과 DLL 사이의 호출 계약이었다.

확인 목록은 다음처럼 정리했다.

- Java 선언과 C++ 구현의 인자 수가 같은가
- `jstring`, `jint`, `jboolean` 순서가 같은가
- 생성된 JNI 헤더를 최신 선언으로 다시 만들었는가
- DLL만 교체한 경우 호출하는 Java 클래스도 같은 버전인가
- 로그는 URL·계정·비밀번호를 노출하지 않고도 인자 위치를 구분할 수 있는가

이번 오류는 curl에서 드러났지만 원인은 HTTP가 아니었다. JNI 경계를 넘는 값이 이상할 때는 가장 바깥의 실패 코드보다 Java와 C++이 공유하는 함수 계약부터 대조해야 한다.
