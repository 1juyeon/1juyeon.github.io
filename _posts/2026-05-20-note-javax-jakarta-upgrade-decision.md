---
title: "[Note] 오래된 Spring 웹 프로젝트를 Jakarta 기준으로 옮기며 확인한 범위"
date: 2026-05-20 20:00:00 +0900
categories: [note]
tags: [spring, tomcat, jakarta, javax, migration]
layout: single
---

오래된 Java 웹 프로젝트의 JDK, Tomcat, Spring을 지원 중인 버전으로 올리는 작업을 시작했다. 처음에는 라이브러리 JAR을 교체하면 될 것 같았지만, 실제 변경은 Java import부터 XML 보안 설정, JSP 라이브러리, DB 연결 풀까지 이어졌다.

첫 마이그레이션 커밋만 100개가 넘는 파일에 영향을 줬다. 이 작업을 하며 버전 숫자보다 먼저 `javax.*`와 `jakarta.*` 기준선을 정해야 한다는 점을 확인했다.

## 패키지명만 바꾸는 작업이 아니었다

Servlet API가 Jakarta로 옮겨가면서 다음 import를 전환했다.

```java
// before
import javax.servlet.http.HttpServletRequest;

// after
import jakarta.servlet.http.HttpServletRequest;
```

컨트롤러뿐 아니라 필터, 세션 리스너, 요청 래퍼, 파일 다운로드의 `ServletOutputStream`, 리플렉션으로 조회하던 `ServletContext` 타입까지 찾아야 했다.

`javax.servlet` 검색 결과만 일괄 치환하면 끝나는 것도 아니었다. 프로젝트에 직접 넣어둔 `servlet-api.jar`, JSP/JSTL, 메일·activation API처럼 Java EE 계열 의존성도 같은 기준으로 맞춰야 했다. WAS가 제공하는 API와 프로젝트 `lib`의 API가 섞이면 컴파일은 되더라도 런타임에 클래스 충돌이 날 수 있다.

## Spring Security API도 같이 바뀌었다

Spring Security 3.x에서 사용하던 클래스와 설정 일부는 새 버전에 없거나 생성 방식이 바뀌었다.

비밀번호 인코더 import는 오래된 패키지에서 현재 패키지로 옮겼다.

```java
// before
org.springframework.security.authentication.encoding.PasswordEncoder

// after
org.springframework.security.crypto.password.PasswordEncoder
```

권한 객체도 제거된 구현 대신 `SimpleGrantedAuthority`로 바꿨다.

```java
authorities.add(new SimpleGrantedAuthority("ROLE_USER"));
```

Remember-me 인증 제공자는 setter 주입이 더 이상 맞지 않아 생성자 인자로 키를 넘기도록 XML을 수정했다.

```xml
<bean class="org.springframework.security.authentication.RememberMeAuthenticationProvider">
    <constructor-arg value="example-key" />
</bean>
```

보안 URL 규칙도 새 버전에서 명시적으로 평가되도록 로그인 경로와 나머지 인증 경로를 순서대로 선언했다. 이런 설정은 애플리케이션이 뜨는지만 봐서는 알기 어렵고 로그인, 자동 로그인, 권한별 접근을 실제로 확인해야 한다.

## 의존성을 묶음으로 봤다

한 라이브러리만 새 버전으로 바꾸면 연쇄적으로 필요한 JAR이 생겼다.

```text
JDK / 컴파일 레벨
-> Tomcat / Servlet·JSP 스펙
-> Spring Framework
-> Spring Security
-> MyBatis·DBCP·JDBC
-> Jakarta Mail·Activation·JSTL
```

예를 들어 Spring Framework가 요구하는 보조 라이브러리가 빠지면 기동 중 `ClassNotFoundException`이 발생했고, MyBatis-Spring은 Spring과 MyBatis 양쪽 버전에 맞는 조합을 골라야 했다. 그래서 필요한 JAR을 무작정 더하는 대신 기존 구버전 JAR을 제거했는지도 함께 확인했다.

## 실제로 나눈 검증 단계

1. 현재 브랜치가 기존 환경에서 빌드되는 기준점을 남겼다.
2. Servlet/Jakarta import와 웹 XML을 먼저 맞췄다.
3. Spring과 Spring Security의 제거된 API를 컴파일 오류 단위로 수정했다.
4. 애플리케이션 기동과 Spring context 생성 로그를 확인했다.
5. 로그인, 로그아웃, 세션, 자동 로그인, 권한 차단을 별도 시나리오로 확인했다.
6. DB 연결, 파일 다운로드, JSP 렌더링처럼 Servlet API를 직접 쓰는 경로를 확인했다.

## 처음 시도와 실제 적용을 구분했다

초기 마이그레이션은 영향 범위를 파악하고 빌드 오류를 줄이는 데 의미가 있었지만, 그 자체를 운영 적용 완료로 보지는 않았다. 이후 로깅, JDBC, OpenSSL 로더, DB, JDK·Tomcat·Spring 전환을 단계별 커밋으로 다시 나눴다.

큰 버전 이동에서 한 번에 모든 JAR을 교체하면 어떤 변경이 회귀를 만들었는지 찾기 어렵다. `javax`에서 `jakarta`로 넘어가는 기준을 먼저 잡고, 보안·DB·웹 계층을 각각 검증 가능한 단계로 분리한 것이 실제 적용 과정에서 더 중요했다.
