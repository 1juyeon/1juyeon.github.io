---
title: "[Error] go.mod에 DTD 오류가 뜰 때"
date: 2026-06-25 20:00:00 +0900
categories: [error]
tags: [go, eclipse, dtd, validation, gomod]
layout: single
review_required: true
---

# go.mod에 DTD 오류가 뜰 때

## 증상

Go 프로젝트의 `go.mod` 파일에 이런 오류가 표시될 수 있다.

```text
문서 유형 선언을 포함하거나 문서 유형 선언이 가리키는 마크업 선언은 올바른 형식이어야 합니다.
DTD Problem
```

처음 보면 `go.mod` 문법이 잘못된 것처럼 보인다. 하지만 실제로는 Go 문제가 아닐 수 있다.

## 원인

Eclipse나 일부 IDE가 `go.mod`를 XML 파일처럼 오해하고 XML Validator를 돌리면 이런 오류가 난다.

`go.mod`는 XML이 아니다.

```go
module example

go 1.22
```

이런 일반 텍스트를 XML 파서가 읽으면 DTD나 문서 유형 선언이 잘못됐다는 오류를 낼 수 있다. 즉 코드 문제가 아니라 도구의 validation 설정 문제다.

## 먼저 확인할 것

명령줄에서 Go 자체가 정상적으로 인식하는지 확인한다.

```bash
go env GOMOD
go list ./...
```

Go 명령은 정상인데 IDE에서만 DTD 오류가 뜬다면, 빌드 오류가 아니라 IDE validator의 false positive로 보면 된다.

## 해결 방법 1: XML Validator에서 제외

Eclipse 기준으로는 프로젝트 속성에서 validation 설정을 열고 XML Validator의 exclude pattern에 `go.mod`를 추가한다.

예시는 다음과 같다.

```text
**/go.mod
```

Go 코드가 Java 웹 프로젝트 안에 함께 들어있을 때 이런 일이 자주 생긴다. Java IDE가 워크스페이스 전체를 XML 검증 대상으로 보면서 Go 파일까지 건드리는 것이다.

## 해결 방법 2: .mod 연결 해제

Content Types 설정에서 `.mod` 확장자가 XML에 연결되어 있다면 제거한다.

```text
Window
-> Preferences
-> General
-> Content Types
-> Text
-> XML
```

여기에서 `.mod`가 XML 확장자로 들어가 있다면 빼준다.

## 해결 방법 3: 프로젝트 구조 분리

Go 도구를 Java 프로젝트 하위에 넣어야 한다면, IDE가 그 폴더를 빌드/검증 대상으로 보지 않게 표시하는 것이 좋다.

- resource filter로 제외
- derived 폴더로 표시
- Go 코드는 VS Code, GoLand 같은 Go 친화 에디터에서 관리

## 정리

`go.mod`의 DTD 오류는 파일 내용 문제가 아니라 IDE가 파일 타입을 잘못 판단해서 생기는 경우가 많다.

이럴 때는 바로 `go.mod`를 고치지 말고, 먼저 "누가 이 오류를 내는지" 확인해야 한다. Go 컴파일러가 아니라 XML Validator가 낸 오류라면, 코드 수정이 아니라 validation 제외 설정이 정답이다.
