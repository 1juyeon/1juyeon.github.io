---
title: "[Note] 1분 주기 스케줄이 겹쳐 실행되는지 코드로 확인한 기록"
date: 2026-06-11 20:00:00 +0900
categories: [note]
tags: [spring, scheduler, concurrency, batch]
layout: single
---

1분마다 실행되는 비밀번호 변경 스케줄이 있었다. 한 스케줄의 장비 처리가 길어질 때 다른 스케줄이 동시에 시작할 수 있는지 확인해야 했다.

확인할 범위는 두 가지였다.

```text
같은 1분에 대상이 된 여러 스케줄이 병렬로 실행되는가?
한 번의 실행이 1분을 넘으면 다음 tick과 겹치는가?
```

## 실제 코드에서 확인한 흐름

스케줄 메서드는 1분마다 실행되고, 실행 대상 목록을 `for` 루프로 처리했다.

```java
@Scheduled(cron = "0 0/1 * * * *")
public void checkRunTask() {
    List<Schedule> schedules = findSchedulesToRun();

    for (Schedule schedule : schedules) {
        runPasswordSchedule(schedule);
    }
}
```

`runPasswordSchedule()`을 별도 executor에 넘기거나 `@Async`로 호출하는 코드도 없었다. 따라서 같은 tick에서 스케줄 A가 끝나야 스케줄 B가 시작한다.

```text
A 시작 → A 완료 → B 시작 → B 완료
```

## 다음 tick도 현재 구성에서는 겹치지 않았다

스케줄러 설정과 실행 thread를 확인했을 때 현재 배포 구조는 한 worker에서 실행됐다. 이전 실행이 끝나지 않으면 다음 1분 호출은 기다리므로 두 번의 `checkRunTask()`가 동시에 돌지 않았다.

결론은 다음과 같았다.

```text
현재 단일 인스턴스/단일 worker 구성에서는 동시 실행되지 않는다.
```

## 동시 실행이 아니라 지연이 문제였다

예를 들어 A가 4분 걸리고 B가 같은 시각에 실행 대상이라면 B는 병렬로 시작하지 않는다. A가 끝난 뒤 시작하므로 예정 시각보다 늦어진다.

```text
10:00 A 시작
10:04 A 완료
10:04 B 시작
```

사용자는 B가 늦게 실행된 것을 보고 스케줄러가 누락됐다고 느낄 수 있다. 하지만 실제 원인은 중복 실행이 아니라 앞 작업의 처리 시간이다.

그래서 시작 시각과 종료 시각, 대상 수, 경과 시간을 같이 남겨야 했다.

```java
long startedAt = System.currentTimeMillis();
try {
    runPasswordSchedule(schedule);
} finally {
    log.info("schedule finished. id={}, elapsedMs={}",
        schedule.getId(),
        System.currentTimeMillis() - startedAt);
}
```

## 결론이 바뀔 수 있는 조건

이번 결론은 현재 코드와 배포 구성에 대한 것이다. 아래 변경이 생기면 다시 확인해야 한다.

- scheduler pool size를 2 이상으로 늘림
- 내부 작업에 `@Async`, `ExecutorService`, `CompletableFuture` 사용
- 애플리케이션 인스턴스를 여러 대 실행
- 별도 프로세스가 같은 스케줄 테이블을 처리

특히 다중 인스턴스에서는 Java 메모리 lock만으로 중복 실행을 막을 수 없다. DB 상태 전이, row lock 또는 분산 lock이 필요하다.

## 정리

처음 확인하려던 것은 “다른 스케줄이 시작될 수 있는가”였고, 실제 코드 기준 결론은 **현재는 시작할 수 없다**였다.

- 같은 tick의 대상은 `for` 루프로 순차 처리한다.
- 현재 scheduler worker도 한 개라 다음 tick과 겹치지 않는다.
- 대신 앞 작업이 길면 뒤 작업이 늦어진다.

일반적인 가능성을 나열하는 것보다 실제 실행 경로, 비동기 호출, scheduler 설정, 배포 인스턴스 수를 확인한 뒤 결론을 내리는 것이 중요했다.
