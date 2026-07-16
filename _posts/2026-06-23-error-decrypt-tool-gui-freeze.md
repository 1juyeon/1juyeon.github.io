---
title: "[Error] Go로 만든 Win32 GUI가 버튼 연타 시 멈출 때"
date: 2026-06-23 20:00:00 +0900
categories: [error]
tags: [go, win32, gui, decryptor, testing]
layout: single
review_required: true
---

# Go로 만든 Win32 GUI가 버튼 연타 시 멈출 때

## 상황

간단한 복구 도구를 Go로 만들고 Win32 API로 GUI를 붙였다.

처음 실행은 잘 되는데, 버튼을 여러 번 누르면 프로그램이 "응답 없음" 상태가 되는 문제가 있었다.

복호화 로직 자체는 테스트를 통과했는데 UI만 멈춘다면, 암호화 코드보다 Win32 메시지 루프를 먼저 봐야 한다.

## 원인

Go 런타임은 goroutine을 여러 OS thread 위에서 스케줄링한다.

그런데 Win32 GUI는 창을 만든 thread와 메시지 루프를 도는 thread가 매우 중요하다. 메시지 루프가 OS thread에 고정되어 있지 않으면, 콜백과 메시지 처리가 꼬여 버튼을 반복해서 누를 때 멈출 수 있다.

이럴 때 필요한 것이 `runtime.LockOSThread()`다.

```go
func runGUI() {
    runtime.LockOSThread()
    defer runtime.UnlockOSThread()

    // create window
    // message loop
}
```

GUI 메인 루프는 한 OS thread에 고정하는 편이 안전하다.

## 버튼 핸들러 방어

복구 도구처럼 사용자가 입력을 붙여넣고 버튼을 누르는 프로그램은 입력 오류가 많다.

버튼 핸들러에는 최소한의 방어가 있어야 한다.

```go
func onDecryptClick() {
    defer func() {
        if r := recover(); r != nil {
            setStatus("내부 오류가 발생했습니다.")
        }
    }()

    // parse input
    // decrypt
    // update result
}
```

잘못된 Base64, 빈 값, 예상하지 못한 포맷이 들어와도 프로그램이 죽지 않고 상태 메시지를 보여줘야 한다.

## 검증 방법

GUI 결과를 자동화로 읽으려 할 때도 주의가 필요하다. Windows에서는 다른 프로세스의 자식 컨트롤 텍스트를 단순 `GetWindowText`로 안정적으로 읽지 못하는 경우가 있다.

그래서 검증을 둘로 나누는 편이 좋다.

- 복호화 함수 정확성: 단위 테스트로 검증
- GUI 안정성: 버튼 연타, 복사 버튼, 잘못된 입력 스트레스 테스트

예를 들면 다음을 확인한다.

- 복호화 버튼 50회 이상 연타해도 멈추지 않음
- 잘못된 입력에서도 프로세스가 살아 있음
- 결과 복사 실패 시 상태 메시지를 보여줌
- CLI 모드가 있다면 같은 복호화 함수를 사용함

## 정리

Go로 Win32 GUI를 직접 만들 때 버튼 연타로 멈춘다면, 복호화 로직보다 OS thread와 메시지 루프를 먼저 의심해야 한다.

GUI thread는 `runtime.LockOSThread()`로 고정하고, 버튼 핸들러에는 `recover()`와 입력 검증을 넣어야 사용자가 실수해도 도구가 죽지 않는다.
