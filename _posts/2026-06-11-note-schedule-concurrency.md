---
title: "[Note] 스케줄러가 겹쳐 실행되는지 확인하는 방법"
date: 2026-06-11 20:00:00 +0900
categories: [note]
tags: [spring, scheduler, concurrency, batch, locking]
layout: single
review_required: true
---

# 스케줄러가 겹쳐 실행되는지 확인하는 방법

## 궁금했던 점

1분마다 실행되는 스케줄러가 있고, 그 안에서 여러 작업을 처리한다면 이런 의문이 생긴다.

- 앞 작업이 끝나기 전에 다음 작업이 시작될까?
- 이번 분의 실행이 길어지면 다음 분 실행과 겹칠까?
- 여러 스케줄 항목이 같은 시각에 도래하면 병렬로 돌까?

이건 스케줄러 설정과 코드 구조를 같이 봐야 한다.

## for 루프면 같은 tick 안에서는 순차 처리

스케줄 메서드 안에서 작업 목록을 가져와 `for` 루프로 처리한다면, 같은 실행 안에서는 기본적으로 순차 처리된다.

```java
@Scheduled(cron = "0 0/1 * * * *")
public void runSchedules() {
    List<Job> jobs = findRunnableJobs();

    for (Job job : jobs) {
        runOneJob(job);
    }
}
```

이 구조에서는 A 작업이 끝나야 B 작업이 시작된다. 같은 시각에 여러 작업이 대상이 되더라도 동시에 실행되는 것이 아니라 뒤 작업이 지연된다.

## 다음 tick과 겹치는지는 스레드풀 설정을 봐야 한다

Spring 스케줄러가 단일 스레드로 동작한다면, 이전 실행이 끝나기 전에는 다음 실행이 기다린다. 그래서 한 번에 하나만 돈다.

하지만 아래 조건이 있으면 겹칠 수 있다.

- 스케줄러 스레드풀이 2개 이상임
- 메서드 내부에서 `@Async` 또는 별도 thread를 사용함
- 작업별로 executor에 던지고 바로 반환함
- 여러 서버 인스턴스가 같은 DB를 보고 스케줄을 실행함

그래서 단순히 `@Scheduled`만 보고 판단하면 안 된다. 실행 스레드 수, 비동기 호출 여부, 배포 인스턴스 수를 같이 확인해야 한다.

## 확인 체크리스트

- `TaskScheduler` 또는 `ScheduledExecutorService` pool size가 몇 개인가
- 스케줄 메서드에 `@Async`가 붙어 있는가
- 내부에서 `CompletableFuture`, `ExecutorService`, `new Thread`를 쓰는가
- 작업 시작 전에 DB 상태를 `RUNNING`으로 바꾸는가
- 작업 종료 또는 예외 시 상태를 반드시 해제하는가
- 같은 애플리케이션이 여러 대 떠 있는가

여러 대가 떠 있다면 애플리케이션 메모리 lock만으로는 부족하다. DB row lock, unique constraint, distributed lock 같은 공유 lock이 필요하다.

## 지연과 중복은 다른 문제

순차 처리 구조에서는 동시 실행은 막히지만 지연은 생길 수 있다.

예를 들어 1분마다 실행되는데 첫 번째 작업이 5분 걸리면, 그 뒤 작업은 계속 밀린다. 동시 실행은 아니지만 사용자는 "스케줄이 제시간에 안 돈다"고 느낄 수 있다.

이 경우에는 병렬화보다 먼저 작업 단위를 줄이거나, 실행 시간을 기록해서 지연을 관찰할 수 있게 만드는 편이 좋다.

```java
long started = System.currentTimeMillis();
try {
    runOneJob(job);
} finally {
    log.info("job finished. id={}, elapsedMs={}", job.id(), System.currentTimeMillis() - started);
}
```

## 예방 설계

스케줄 작업은 재실행과 중복 실행을 전제로 설계하는 것이 안전하다.

- 작업 시작 시 `RUNNING` 상태를 기록한다.
- 이미 `RUNNING`이면 건너뛰거나 timeout 기준으로 복구한다.
- 성공/실패와 관계없이 finally에서 상태를 정리한다.
- 외부 장비나 API 호출은 idempotent하게 만든다.
- 실행 이력을 남겨 지연과 실패를 구분한다.

정리하면, `for` 루프 자체는 순차 실행을 보장하지만 전체 시스템의 중복 실행까지 보장하지는 않는다. 스레드풀, 비동기 호출, 다중 인스턴스, DB 상태 전환까지 같이 확인해야 한다.
