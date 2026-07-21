---
title: "[Error] 장비용 JSON 템플릿 로드 실패가 통신 오류로 보인 문제"
date: 2026-07-08 20:00:00 +0900
categories: [error]
tags: [json, java, null, routing, network]
layout: single
---

SSH 장비의 비밀번호를 변경하는 기능에서 화면에는 일반적인 통신 실패만 표시됐다. 네트워크보다 먼저 객체 생성 경로를 확인하니, 장비용 JSON 템플릿을 읽지 못한 객체가 다음 단계로 넘어가며 `NullPointerException`이 발생할 수 있는 구조였다.

문제는 "JSON 파일이 있느냐" 하나로 끝나지 않았다. DB에서 어떤 실행 방식을 선택했는지, 템플릿 이름이 전달됐는지, 해당 템플릿을 읽는 클래스가 맞는지까지 연결되어 있었다.

## 전용 구현과 공통 템플릿 구현이 같이 있었다

기존 코드는 제조사·모델 문자열에 따라 전용 클래스를 먼저 선택했다. 새 장비는 같은 제조사 계열이더라도 공통 JSON 템플릿 엔진으로 실행해야 했다.

```text
장비 정보 조회
-> 전용 클래스 또는 공통 템플릿 엔진 선택
-> JSON 파일 로드
-> SSH 연결
-> send/expect 단계 실행
```

제조사 이름만 보면 기존 전용 클래스로 들어가기 때문에, DB에 `use_generic` 값을 추가해 공통 엔진을 명시적으로 선택하도록 했다.

```java
if (device.isUseGeneric()
        && hasText(device.getTemplateName())) {
    GenericSshSwitch candidate =
        new GenericSshSwitch(ip, adminId, device.getTemplateName());

    if (candidate.isTemplateLoaded()) {
        result = candidate;
    }
} else if (isKnownVendor(device)) {
    result = new VendorSpecificSwitch(...);
}
```

이렇게 하면 제조사 문자열이 기존 분기와 겹쳐도 DB 설정이 `use_generic=true`인 장비는 JSON 엔진으로 들어간다.

## 로드 실패 객체를 반환하지 않게 했다

전용 클래스 중 하나는 생성자에서 JSON 파싱 예외를 잡고 로그만 남겼다. 상위 코드는 객체가 생성됐다는 사실만 보고 다음 메서드를 호출할 수 있었다.

```java
try {
    json = load(path);
    templateLoaded = true;
} catch (Exception e) {
    templateLoaded = false;
    log.error("template load failed: {}", path, e);
}
```

팩토리에서는 `templateLoaded`를 확인해 실패한 객체를 반환하지 않도록 가드를 추가했다.

```java
VendorSpecificSwitch candidate = new VendorSpecificSwitch(...);
if (candidate.isTemplateLoaded()) {
    result = candidate;
}
```

공통 템플릿 엔진도 실행 전에 같은 상태를 확인하고, 로드하지 못했다면 SSH 연결을 시도하지 않은 채 설정 오류 코드를 반환하도록 했다. 핵심은 `json == null`인 상태를 정상 객체처럼 다음 계층으로 보내지 않는 것이다.

## 실제로 확인한 조건

로드 실패 원인을 다음 순서로 나눠 확인했다.

1. 장비의 제조사·모델 정보가 어떤 분기를 선택하는지
2. DB의 `use_generic`과 템플릿 이름이 객체에 채워지는지
3. 설치 경로에 JSON 파일이 배포됐는지
4. JSON 문법과 필수 `info`, `change_pw`, `check_pw` 항목이 맞는지
5. 기존 전용 JSON과 공통 템플릿 JSON을 서로 다른 엔진이 읽고 있지 않은지

같은 `.json`이어도 스키마는 같지 않았다. 기존 전용 클래스가 읽는 키와 새 공통 엔진이 읽는 키가 달랐기 때문에 파일 확장자만 보고 호환된다고 판단할 수 없었다.

## 테스트하며 남긴 한계

코드 컴파일과 JSON 파싱, 수동 SSH 명령 흐름은 확인했지만 이 시점에 웹 화면부터 실제 장비까지 이어지는 전체 경로를 모두 검증한 것은 아니었다. 그래서 글의 결론도 "현장 검증 완료"가 아니라 다음 회귀 테스트가 남았다고 구분했다.

- 웹에서 변경·확인 버튼을 눌렀을 때 공통 엔진으로 들어가는지
- DB에 `use_generic`과 템플릿 이름이 정확히 반영됐는지
- 틀린 비밀번호와 장비 연결 실패가 서로 다른 오류로 표시되는지
- 기존 전용 구현을 쓰는 장비가 이전 분기를 그대로 타는지

이번 오류에서 중요한 수정은 NPE 한 곳을 막는 데 그치지 않았다. DB 설정, 객체 팩토리, JSON 로더, 실제 배포 파일이 하나의 실행 경로를 이룬다는 점을 확인하고, 초기화에 실패한 객체가 통신 단계까지 가지 않도록 경계를 만든 작업이었다.
