---
title: "[Note] 배치 실행 로그에서 비밀번호가 찍히지 않게 하는 방법"
date: 2026-06-22 20:00:00 +0900
categories: [note]
tags: [security, logging, batch, masking, java]
layout: single
review_required: true
---


## 상황

Java에서 배치 파일을 실행하고, 그 표준 출력을 그대로 로그에 남기는 구조가 있었다.

배치 파일 안에는 DB 백업을 위해 환경변수를 설정하는 줄이 있었고, Windows batch의 기본 동작 때문에 실행 전에 명령 자체가 콘솔에 출력됐다.

```bat
set PGPASSWORD=<REDACTED>
```

이 출력이 Java 로그에 그대로 수집되면 비밀번호가 로그 파일에 남는다.

## 1차 방어: 배치 파일에 echo off

배치 파일 맨 위에 아래 줄을 넣으면 명령 복창이 꺼진다.

```bat
@echo off
```

그러면 `set PGPASSWORD=<REDACTED>` 같은 줄이 실행 전에 콘솔에 찍히지 않는다. 배치 파일을 직접 관리할 수 있다면 가장 간단한 개선이다.

하지만 이것만으로는 완전하지 않다.

- 운영 서버의 실제 배치 파일이 수정되지 않았을 수 있음
- 누군가 배치 파일을 다시 생성하거나 덮어쓸 수 있음
- 배치 파일 안에서 다른 명령이 값을 출력할 수 있음
- 배포본과 저장소 원본이 서로 다를 수 있음

그래서 로그를 남기는 애플리케이션 쪽에서도 한 번 더 막아야 한다.

## 2차 방어: 로그 출력 직전 마스킹

가장 견고한 위치는 "외부 명령의 출력을 로그에 쓰기 직전"이다.

```java
String output = runBatch();
log.info("batch output: {}", maskSensitive(output));
```

마스킹 함수는 필요한 패턴만 가리는 식으로 둔다.

```java
private String maskSensitive(String text) {
    if (text == null) {
        return null;
    }

    return text.replaceAll("(?i)(PGPASSWORD\\s*=\\s*)[^\\r\\n]+", "$1********");
}
```

이렇게 하면 배치 파일이 echo on 상태여도 로그에는 평문이 남지 않는다.

## 원본 출력은 바꾸지 않는 편이 안전하다

주의할 점은 실제 프로세스 출력값 전체를 바꿔서 후속 로직에 넘기지 않는 것이다.

후속 코드가 출력 내용을 파싱해서 성공/실패를 판단한다면, 원본을 변형하면 예기치 않은 문제가 생길 수 있다. 그래서 보통은 "로그에 찍는 문자열만 마스킹"하는 방식이 안전하다.

```java
String rawOutput = runBatch();
String logOutput = maskSensitive(rawOutput);

log.info("output: {}", logOutput);
return rawOutput;
```

## 둘 중 하나만 한다면

둘 다 하는 것이 가장 좋다.

- `@echo off`: 불필요한 명령 복창 자체를 없애 로그를 깔끔하게 함
- Java 마스킹: 배치 파일 상태와 무관하게 로그 출력 지점에서 비밀값을 차단함

굳이 우선순위를 정한다면 Java 마스킹이 더 중요하다. 운영 서버의 배치 파일은 사람이나 배포 과정에 따라 달라질 수 있지만, 로그 출력 지점은 애플리케이션 안에서 통제할 수 있기 때문이다.

## 정리

로그 보안은 한 곳에서만 막으면 쉽게 뚫린다. 특히 외부 명령을 실행하고 출력을 수집하는 구조에서는 실행 스크립트와 애플리케이션 로그 지점을 모두 확인해야 한다.

민감한 값은 "출력하지 않기"와 "출력 직전 마스킹하기"를 같이 적용하는 것이 안전하다.
