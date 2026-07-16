---
title: "[Error] 완료 팝업이 계속 반복될 때 원인 분리하기"
date: 2026-06-08 20:00:00 +0900
categories: [error]
tags: [javascript, polling, toast, cookie, ajax]
layout: single
review_required: true
---

# 완료 팝업이 계속 반복될 때 원인 분리하기

## 상황

웹 화면 우측 하단에 "작업 완료" 같은 초록색 팝업이 한 번만 떠야 하는데, 같은 메시지가 계속 반복해서 뜨는 경우가 있다.

이때 원인은 크게 두 가지로 나뉜다.

1. 서버에서 작업이 실제로 여러 번 완료되고 있다.
2. 작업은 한 번만 완료됐지만 브라우저의 중복 방지 로직이 실패하고 있다.

둘을 먼저 분리해야 한다. 서버 문제인지 프론트 문제인지 섞어 보면 수정 위치가 계속 흔들린다.

## 1단계: 서버가 실제로 반복 실행되는지 확인

서버 로그에서 같은 작업 완료 로그가 짧은 시간 안에 여러 번 찍히는지 확인한다.

- 완료 로그가 여러 번 찍힘: 서버 스케줄 또는 상태 전환 로직 확인
- 완료 로그는 한 번뿐임: 클라이언트 polling, cookie, localStorage 중복 방지 확인

브라우저에서 1초마다 polling을 하고 있다면, 완료 상태가 유지되는 동안 매번 같은 응답을 받을 수 있다. 이때 "이미 보여준 완료인지"를 안정적으로 판별해야 한다.

## 2단계: 중복 방지 키가 안정적인지 확인

중복 방지 키로 작업 이름을 그대로 쓰면 문제가 생길 수 있다.

- 공백
- 한글
- 특수문자
- 화면 표시용 문구 변경
- 같은 이름을 가진 다른 작업

이런 값은 cookie key로 쓰기에 불안정하다. 가능하면 숫자 ID나 서버가 내려주는 고유 코드처럼 변하지 않는 값을 써야 한다.

```javascript
const key = `task-done:${taskCode}`;
const currentVersion = String(resultUpdatedAt || resultCode);

if (localStorage.getItem(key) !== currentVersion) {
  showToast(message);
  localStorage.setItem(key, currentVersion);
}
```

`taskName`처럼 사람이 읽는 값은 메시지에는 좋지만 상태 저장 키로는 부적합하다.

## 3단계: 팝업 라이브러리 중복 옵션 확인

toastr 같은 라이브러리를 쓴다면 동일 메시지 중복 방지 옵션도 같이 켠다.

```javascript
toastr.options = {
  preventDuplicates: true,
  timeOut: 10000
};
```

다만 이 옵션은 보조 장치다. 근본적으로는 polling 로직에서 "이번 완료 알림을 이미 보여줬는지"를 판단해야 한다.

## 4단계: polling 내부의 동기 AJAX 제거

`setInterval` 안에서 `async: false` AJAX를 여러 번 호출하면 브라우저가 멈추거나 응답 순서가 꼬일 수 있다.

```javascript
setInterval(function () {
  // 동기 AJAX를 여러 번 호출하는 구조는 피한다.
}, 1000);
```

polling은 비동기로 처리하고, 이전 요청이 아직 끝나지 않았다면 다음 요청을 건너뛰는 방식이 더 안전하다.

```javascript
let polling = false;

setInterval(async function () {
  if (polling) return;
  polling = true;

  try {
    await checkTaskStatus();
  } finally {
    polling = false;
  }
}, 1000);
```

## 정리

반복 팝업은 "서버가 반복 실행됨"과 "클라이언트 중복 방지가 실패함" 중 하나다.

서버 로그가 한 번만 찍혔다면 브라우저 쪽에서 중복 키를 의심해야 한다. 특히 작업 이름처럼 화면용 문자열을 cookie key로 쓰고 있다면, 숫자 ID나 고유 코드 기반으로 바꾸는 것이 안전하다.
