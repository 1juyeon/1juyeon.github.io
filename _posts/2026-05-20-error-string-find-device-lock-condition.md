---
title: "[Error] std::string::find 조건을 반대로 써 장비 잠금으로 오판한 문제"
date: 2026-05-20 21:00:00 +0900
categories: [error]
tags: [cpp, string, condition, onvif, debugging]
layout: single
---

카메라 인증 실패 응답을 문자열로 분류하는 C++ 코드에서, 잠금 메시지가 없는 경우까지 장비 잠금 오류로 처리될 수 있는 조건을 발견했다.

문제가 된 코드는 다음 형태였다.

```cpp
if (response.find("The device is locked") == std::string::npos) {
    result = DEVICE_LOCKED;
}
```

코드만 빠르게 읽으면 문장이 자연스러워 보이지만 조건은 반대다.

## find의 반환값을 다시 확인했다

`std::string::find()`는 문자열을 찾으면 발견한 시작 위치를 반환하고, 찾지 못하면 `std::string::npos`를 반환한다.

```text
응답: "The device is locked"
find 결과: 0 또는 다른 위치 값

응답: "Unauthorized"
find 결과: std::string::npos
```

따라서 `== std::string::npos`는 "잠금 문구가 없다"는 뜻이다. 기존 코드는 잠금 문구가 없을 때 `DEVICE_LOCKED`를 반환하고 있었다.

수정은 비교 연산자 하나였지만 의미는 완전히 달랐다.

```cpp
if (response.find("The device is locked") != std::string::npos) {
    result = DEVICE_LOCKED;
}
```

## 왜 단순 오타가 오래 남을 수 있었나

이 코드는 여러 오류 문자열을 `else if`로 분류하는 중간에 있었다. 앞 조건에서 인증 오류나 HTTP 오류가 먼저 걸리면 잠금 분기까지 내려오지 않기 때문에, 모든 실패가 잠금으로 보이는 것은 아니었다.

또한 실제 장비는 여러 번 인증에 실패해야 잠금 응답을 보내므로 정상·인증 실패 테스트만으로는 이 분기의 참과 거짓을 모두 확인하기 어려웠다.

## 수정 후 확인한 경우

문자열 포함 여부는 최소 두 방향을 함께 확인해야 한다.

```cpp
assert(classify("The device is locked") == DEVICE_LOCKED);
assert(classify("Unauthorized") != DEVICE_LOCKED);
```

대소문자나 장비별 문구 차이가 있다면 별도 정규화가 필요하지만, 이번 수정에서는 기존에 수집한 정확한 문구와 분기 의미를 먼저 맞췄다. 범위를 넓히겠다고 근거 없는 키워드를 추가하면 다른 응답을 다시 오분류할 수 있기 때문이다.

## 남긴 기준

- `find(...) == npos`: 포함되지 않음
- `find(...) != npos`: 포함됨
- 오류 분류 테스트에는 일치하는 입력과 일치하지 않는 입력을 함께 둔다.
- 장비 상태처럼 의미가 큰 오류는 문자열 분기 뒤 실제 재시도 결과도 함께 본다.

한 글자 수정이었지만 사용자에게는 "계정이 잠겼다"와 "다른 인증 오류가 났다"의 차이다. 문자열 비교 코드는 반환값의 의미를 조건문과 말로 각각 읽어보는 습관이 필요했다.
