---
title: "250620 curl api auth error"
date: 2025-06-20 10:00:00 +0900
categories: [error]
tags: [curl, api, 인증, digest, basic]
layout: single
---

# 🔐 curl 인증 방식(Basic vs Digest) 및 사용 방법 정리

이 문서는 curl을 사용하여 인증이 필요한 장치(API 서버 등)에 요청할 때,  
**Basic 인증과 Digest 인증의 차이점**과  
**실제 명령어 사용법**, **디버깅 방법** 등을 정리한 실전 가이드입니다.

---

## 📌 인증 방식 개요

| 구분 | Basic 인증 | Digest 인증 |
|------|-------------|---------------|
| 전송 방식 | `username:password`를 Base64로 인코딩 | username, password, nonce 등을 이용해 해시값 생성 |
| 보안 수준 | 낮음 (HTTPS 필수) | 중간 (비밀번호 직접 노출은 없음) |
| 암호화 필요 | 반드시 HTTPS 필요 | HTTPS 없어도 해시 전송이므로 어느 정도 안전하지만 여전히 HTTPS 권장 |
| curl 사용법 | `-u "id:pw"` | `--digest -u "id:pw"` |
| 서버 요구 헤더 | `WWW-Authenticate: Basic ...` | `WWW-Authenticate: Digest ...` |
| 장점 | 구현이 간단함 | 비밀번호 평문 노출이 없음 |
| 단점 | 평문 인코딩(Base64), 쉽게 복호화 가능 | 복잡함, 일부 시스템 미지원 가능 |

---

## 🔍 서버가 사용하는 인증 방식 확인

### 명령어
```bash
curl -v -k "https://[서버주소]/api"
```

### 결과 예시

#### [1] Digest 인증을 사용하는 경우
```http
HTTP/1.1 401 Unauthorized
WWW-Authenticate: Digest realm="Camera", nonce="abc123", qop="auth"
```

#### [2] Basic 인증을 사용하는 경우
```http
HTTP/1.1 401 Unauthorized
WWW-Authenticate: Basic realm="Camera"
```

---

## 🧪 curl 명령어 비교

### ✅ 1. Basic 인증 방식 (서버가 Basic을 요구할 때)
```bash
curl -u "admin:1234" -k "https://192.168.0.100/stw-cgi/system.cgi?..."
```

### ✅ 2. Digest 인증 방식 (서버가 Digest를 요구할 때)
```bash
curl --digest -u "admin:1234" -k "https://192.168.0.100/stw-cgi/system.cgi?..."
```

### 🔍 3. 인증 방식 및 요청 흐름 로그 확인
```bash
curl -v -u "admin:1234" -k "https://..."
```

- `-v` : 요청 및 응답 헤더 전체 출력
- `-k` : 인증서 검증 생략 (사설 인증서 허용)
- `-u` : 사용자 인증 정보
- `--digest` : Digest 방식 명시 (필요한 경우만)

---

## 🧾 인증 실패 시 확인할 사항

| 체크 항목 | 설명 |
|-----------|------|
| 🔑 비밀번호에 특수문자 포함 여부 | `!`, `^`, `&` 등은 cmd에서 오동작 가능 → `"비밀번호"`로 감싸기 |
| 🧷 잘못된 인증 방식 사용 | Digest인데 Basic 쓰거나, 반대로 사용한 경우 |
| 🔁 서버 응답 로그 확인 | `curl -v`로 `WWW-Authenticate:` 헤더 분석 |
| 📶 비밀번호 또는 ID 오타 | 붙여넣기 대신 직접 타이핑 시도 |
| 💻 명령어 실행 환경 | CMD에서 특수문자 문제 시 PowerShell로 시도 |

---

## 🛡️ 보안 참고

- Basic 인증은 반드시 HTTPS와 함께 사용해야 함
- Digest도 완전한 보안은 아니며 중요한 API에는 JWT, OAuth 등을 고려해야 함
- `-k`는 **사설 인증서 테스트 용도로만 사용**하며, 운영 환경에서는 피할 것

---

## 📁 기타: 인증서 관련 질문에 대한 답변 요약

### 🔸 사설 인증서를 다른 카메라에 복사하면?
- 단순히 `.crt` 또는 `.pem`만 복사해도 **다른 장치의 내부 정보는 알 수 없음**
- 다만, 개인키까지 함께 복사하면 **장치 가장 가능성** 존재
- 일반적으로는 **장치의 인증 정보, 설정, 비밀번호 등은 포함되어 있지 않음**

---

## 📌 요약 정리표

| 목적 | 명령어 |
|------|--------|
| 인증 없이 서버 인증 방식 확인 | `curl -v -k "https://..."` |
| Basic 인증으로 요청 | `curl -u "admin:pw" -k "https://..."` |
| Digest 인증으로 요청 | `curl --digest -u "admin:pw" -k "https://..."` |
| 디버깅 및 인증 로그 확인 | `curl -v -u "admin:pw" -k "https://..."` |

---

## ✍️ 예시로 복습해보기

```bash
# 인증 없이 서버 인증 방식 확인
curl -v -k "https://192.168.0.100/stw-cgi/system.cgi?..."

# 서버가 Basic이라면 이렇게 요청
curl -u "admin:1234" -k "https://192.168.0.100/stw-cgi/system.cgi?..."

# 서버가 Digest라면 이렇게 요청
curl --digest -u "admin:1234" -k "https://192.168.0.100/stw-cgi/system.cgi?..."
```

---

📌 작성일: 2025-06-09  
🧠 작성자: ChatGPT + 사용자 질의 기반  
