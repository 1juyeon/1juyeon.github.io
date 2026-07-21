---
title: "[Error] Windows 서비스가 정지된 직후 다시 시작되지 않은 문제"
date: 2026-06-18 20:00:00 +0900
categories: [error]
tags: [windows, service, batch, retry, deployment]
layout: single
---

데이터베이스 설정을 바꾼 뒤 웹 서비스를 재시작하는 배치에서 간헐적으로 시작이 실패했다. 정지 도구는 성공을 반환했고 서비스 조회 결과도 `STOPPED`였지만, 바로 실행한 시작 명령은 실패했다.

```text
service stopped successfully
starting service ...
서비스를 시작하거나 멈추고 있습니다. 나중에 다시 하십시오.
start errorlevel=2
```

같은 서버에서 잠시 뒤 수동으로 시작하면 정상 동작했다. 설정 자체가 항상 잘못됐다기보다 정지와 시작 사이의 타이밍을 먼저 의심했다.

## STOPPED만 보고 바로 시작하고 있었다

기존 흐름은 정지 상태가 확인되면 시작을 한 번 호출하고 끝났다.

```bat
:wait_stop
sc query MyWebService | find "STOPPED" >nul
if errorlevel 1 goto wait_stop

net start MyWebService
if errorlevel 1 goto start_failed
```

Windows 서비스 제어 관리자(SCM)의 상태가 `STOPPED`로 바뀌었더라도 프로세스의 포트와 파일 핸들이 완전히 정리되기까지 시간이 더 걸릴 수 있다. 이 짧은 구간에서 다시 시작하면 SCM은 아직 전환 중인 서비스로 판단할 수 있다.

여기서 race condition은 두 작업의 실행 순서와 속도에 따라 결과가 달라지는 상태를 뜻한다. 항상 실패하지 않고 가끔 실패한 이유도 이 타이밍에 있었다.

## 시작을 최대 다섯 번 재시도했다

한 번 실패했다고 전체 복원 작업을 실패시키지 않고, 대기 후 시작 상태를 다시 확인하도록 바꿨다.

```bat
set START_RETRY=0

:start_service
set /a START_RETRY+=1
net start MyWebService
set START_RESULT=%errorlevel%

sc query MyWebService | find "RUNNING" >nul
if not errorlevel 1 goto start_ok

if %START_RETRY% GEQ 5 goto start_failed
timeout /t 3 /nobreak >nul
goto start_service
```

`net start` 직후 `errorlevel`을 저장한 이유는 그 뒤의 `sc`, `find`, `echo` 명령이 종료 코드를 덮을 수 있기 때문이다. 시작 명령의 반환값과 실제 `RUNNING` 상태도 따로 확인했다.

이 수정의 목적은 모든 시작 실패를 숨기는 것이 아니다. 서비스 전환 중 발생한 일시적인 실패만 흡수하고, 다섯 번 뒤에도 올라오지 않으면 설정·권한·포트 문제로 남기는 것이다.

## 저장소의 배치 파일만 고쳐서는 부족했다

코드를 따라가 보니 애플리케이션은 설치 경로에 있는 배치 파일을 실행했다. 그 파일이 없으면 Java 서비스 코드가 문자열 템플릿으로 배치를 다시 생성하는 구조였다.

따라서 같은 로직이 두 곳에 존재했다.

```text
저장소의 배치 원본
실제 설치 경로에 배포된 배치
Java 코드 안의 배치 생성 템플릿
```

원본만 수정하면 이미 설치된 환경은 예전 파일을 계속 실행한다. 배포 파일만 수정하면 다음에 파일이 재생성될 때 예전 로직으로 돌아간다. 배치와 Java 템플릿을 함께 맞추고, 실제 실행 경로의 파일이 교체됐는지 확인했다.

## 검증 기준

- 서비스가 빠르게 정지되는 경우에도 한 번에 다시 시작되는가
- 첫 시작이 실패했을 때 대기 후 재시도하는가
- 이미 `RUNNING`이면 성공으로 판정하는가
- 다섯 번 실패하면 오류 코드와 시도 횟수가 로그에 남는가
- 배치 파일을 삭제해 다시 생성해도 같은 재시도 로직이 들어가는가

이번 문제는 `timeout` 한 줄을 넣는 것보다 실행 파일의 원본이 어디인지 찾는 데 시간이 더 들었다. 운영 자동화에서는 코드 저장소의 파일, 설치된 파일, 런타임 생성 템플릿이 서로 다른 원본이 될 수 있다는 점까지 확인해야 수정이 유지된다.
