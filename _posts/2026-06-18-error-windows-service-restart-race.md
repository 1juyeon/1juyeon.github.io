---
title: "[Error] Windows 서비스 재시작이 가끔 실패할 때"
date: 2026-06-18 20:00:00 +0900
categories: [error]
tags: [windows, service, batch, retry, tomcat]
layout: single
review_required: true
---

# Windows 서비스 재시작이 가끔 실패할 때

## 증상

배치 파일에서 Windows 서비스를 멈춘 뒤 바로 다시 시작하는 구조가 있었다.

로그는 대략 이런 흐름이다.

```text
service stopped successfully
starting service ...
서비스를 시작하거나 멈추고 있습니다. 나중에 다시 하십시오.
start errorlevel=2
```

정지는 성공했다고 나오는데 시작이 실패한다. 이후 수동으로 다시 시작하면 정상적으로 올라온다.

## 원인

이런 경우는 정지와 시작 사이의 race condition인 경우가 많다.

서비스 정지 명령이 "정지 요청 성공" 또는 "SCM 상태상 STOPPED"를 보고 끝났더라도, 실제 프로세스와 포트, 파일 핸들, 내부 스레드가 완전히 정리되는 데는 시간이 더 걸릴 수 있다.

그 짧은 틈에 `net start`를 실행하면 Windows는 아직 서비스를 시작하거나 멈추는 중이라고 판단하고 오류를 반환한다.

## 기존 스크립트의 문제

문제가 되는 구조는 보통 이렇다.

```bat
net stop MyService
sc query MyService | findstr /i "STOPPED"
net start MyService
echo start errorlevel=%errorlevel%
```

`STOPPED`를 보자마자 바로 시작하고, 실패해도 한 번만 시도한 뒤 끝난다. 서비스가 실제로 조금 뒤에 시작 가능 상태가 되더라도 스크립트는 이미 실패로 종료된다.

## 개선 방향

서비스가 `STOPPED`로 보인 뒤 짧게 settle time을 둔다. 그리고 시작은 재시도한다.

```bat
timeout /t 3 /nobreak >nul

set _try=1
:start_retry
echo starting service try %_try%
net start MyService
set _err=%errorlevel%

sc query MyService | findstr /i "RUNNING" >nul
if not errorlevel 1 goto start_done

if %_try% geq 5 goto start_failed
set /a _try+=1
timeout /t 3 /nobreak >nul
goto start_retry

:start_done
echo service running
goto end

:start_failed
echo service start failed. errorlevel=%_err%

:end
```

중요한 점은 `errorlevel`을 바로 변수에 저장하는 것이다. 중간에 `echo`, `findstr`, `sc query` 같은 명령이 끼면 원래의 종료 코드를 잃을 수 있다.

## 성공 판정은 한 번 더 확인

`net start`가 0이 아닌 값을 반환해도 실제 서비스가 이미 `RUNNING`이 된 경우가 있다. 그래서 시작 명령의 종료 코드만 믿지 말고 `sc query`로 최종 상태를 한 번 더 확인하는 것이 좋다.

반대로 5번 모두 실패한다면 그때는 race condition이 아니라 항상적인 시작 실패일 수 있다.

- 포트 충돌
- JVM 옵션 오류
- 권한 문제
- 설정 파일 오류
- 서비스 계정 문제

이 경우에는 서비스 자체 로그를 확인해야 한다.

## 배포 시 주의할 점

이런 배치 파일은 저장소에 있는 원본과 실제 설치 경로에 배포된 파일이 다를 수 있다. 애플리케이션이 설치 경로의 배치 파일을 직접 실행한다면, 저장소 파일만 고쳐서는 문제가 해결되지 않는다.

확인할 것은 두 가지다.

- 실제 실행되는 배치 파일이 수정됐는가
- 파일이 없을 때 애플리케이션이 다시 생성하는 템플릿도 수정됐는가

정리하면, 서비스 재시작 실패가 가끔만 발생한다면 정지 직후 시작하는 race condition을 먼저 의심해볼 만하다. `STOPPED` 확인 후 대기, 시작 재시도, 최종 `RUNNING` 확인을 넣으면 운영 중 간헐 오류를 줄일 수 있다.
