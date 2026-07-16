---
title: "[Note] Spring과 Tomcat 업그레이드에서 javax와 jakarta를 먼저 봐야 하는 이유"
date: 2026-05-20 20:00:00 +0900
categories: [note]
tags: [spring, tomcat, jakarta, javax, migration]
layout: single
review_required: true
---

# Spring과 Tomcat 업그레이드에서 javax와 jakarta를 먼저 봐야 하는 이유

## 핵심

Java 웹 프로젝트에서 Spring과 Tomcat을 업그레이드할 때는 버전 숫자보다 먼저 `javax.*`와 `jakarta.*` 기준선을 봐야 한다.

이 기준을 잘못 잡으면 "JDK는 올라갔는데 WAS와 Spring이 서로 맞지 않는" 조합이 된다.

## 왜 문제가 되는가

오래된 Java 웹 프로젝트는 보통 아래 패키지를 사용한다.

```java
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import javax.servlet.Filter;
```

하지만 Jakarta EE 기반으로 넘어가면 패키지가 바뀐다.

```java
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import jakarta.servlet.Filter;
```

이건 단순 import 변경이 아니다. WAS, Spring, Spring Security, JSP, taglib, servlet-api, 테스트 코드까지 영향을 받는다.

## 조합을 고를 때 보는 기준

업그레이드 후보를 볼 때는 다음 질문을 먼저 던진다.

- 현재 코드는 `javax.*` 기반인가?
- 목표 Spring 버전은 Jakarta 기반인가?
- 목표 Tomcat 버전은 어떤 Servlet 스펙을 지원하는가?
- JSP와 taglib가 Jakarta 호환 버전인가?
- 서드파티 라이브러리가 Jakarta 네임스페이스를 지원하는가?

이 질문에 답하지 않고 "JDK 21이면 되겠지", "Tomcat 최신이면 되겠지"라고 고르면 중간에 막힌다.

## 최소 변경 경로와 전면 마이그레이션 경로

보통 선택지는 두 가지다.

| 경로 | 특징 |
| --- | --- |
| `javax.*` 유지 경로 | 코드 변경이 적고 회귀 위험이 낮다. 단, 지원 기간을 반드시 확인해야 한다. |
| `jakarta.*` 전환 경로 | 장기 유지보수에는 유리하지만 전면 마이그레이션이 필요하다. |

보안 심사 대응처럼 일정이 정해진 작업에서는 "지원 중인 버전"과 "마이그레이션 리스크"를 같이 봐야 한다. 지원 종료된 버전을 피하려다 전면 마이그레이션을 무리하게 넣으면 일정 리스크가 더 커질 수 있다.

## 실제로 확인할 파일들

영향 범위를 빨리 보려면 아래부터 검색한다.

```text
javax.servlet
javax.annotation
javax.validation
javax.xml.bind
javax.persistence
```

웹 프로젝트라면 다음 파일도 같이 본다.

- `web.xml`
- Spring context XML
- Spring Security XML 또는 Java config
- JSP taglib 선언
- `lib/`에 직접 들어간 `servlet-api.jar`
- Eclipse facet, classpath, build path 설정

특히 `servlet-api.jar`를 프로젝트 `lib/`에 직접 넣어둔 오래된 구조라면 WAS가 제공하는 API와 충돌할 수 있다.

## 마이그레이션을 시작하기 전 체크

전면 전환을 하기로 했다면 바로 import만 바꾸지 말고 먼저 빌드 기준선을 만든다.

1. 현재 브랜치에서 전체 빌드가 되는지 확인한다.
2. 테스트 가능한 핵심 시나리오를 정한다.
3. JDK와 WAS를 먼저 고정한다.
4. 프레임워크와 보안 라이브러리 버전을 맞춘다.
5. `javax` 검색 결과를 모듈별로 나눈다.
6. 한 번에 전체를 고치지 말고 웹 계층, 보안 계층, JSP 계층을 나눠서 본다.

## 정리

Spring/Tomcat 업그레이드는 "어느 버전이 더 최신인가"보다 "같은 Servlet/Jakarta 계열인가"가 먼저다.

`javax.*`를 유지할지, `jakarta.*`로 전환할지를 먼저 결정하면 가능한 조합과 작업 범위가 훨씬 선명해진다.
