---
title: "[Note] mTLS 인증 curl 테스트 가이드"
date: 2025-06-25 20:00:00 +0900
categories: [note]
layout: single
---

# mTLS 인증을 위한 curl 테스트 가이드

## 📌 목적
- HTTPS 장비 API와 안전하게 mTLS(mutual TLS) 통신을 테스트합니다.
- 클라이언트 인증서를 이용하여 `curl` 명령어로 API를 호출합니다.

---

## 1️⃣ 필요한 파일 준비
테스트를 위해 다음 인증서 파일이 필요합니다:

| 종류 | 설명 | 예시 파일명 |
|------|------|--------------|
| CA 인증서 | 서버의 신뢰를 보장하는 루트 인증서 | `ca.pem`, `rootca.crt` |
| 클라이언트 인증서 | 클라이언트의 신원을 증명 | `client.crt`, `client.pem` |
| 클라이언트 개인 키 | 인증서의 실제 소유권 증명 | `client.key` 또는 `client.pfx` |

### 🔍 파일 위치
보통 장비 웹 페이지의 '인증서 관리' > '클라이언트 인증서' 메뉴에서 다음을 시도합니다:
- 기존 인증서 **내보내기**: `.pfx` 또는 `.zip` 다운로드
- 새 인증서 **발급(Issue)**: `.pfx`, `.crt` + `.key` 생성

---

## 2️⃣ curl 명령어 구성

### A. PEM 형식 (client.crt + client.key + ca.pem)

```bash
curl --cacert "C:\\path\\to\\certs\ca.pem" ^
     --cert "C:\\path\\to\\certs\client.crt" ^
     --key "C:\\path\\to\\certs\client.key" ^
     --anyauth -u "<user>:<password>" ^
     "https://<device-host>/<api-path>"
```

### B. PFX 형식 (개인 키 포함)

```bash
curl --cacert "C:\\path\\to\\certs\ca.pem" ^
     --cert-type PFX ^
     --cert "C:\\path\\to\\certs\client.pfx" ^
     --pass "pfx-암호" ^
     --anyauth -u "<user>:<password>" ^
     "https://<device-host>/<api-path>"
```

> ⚠️ `-k` 옵션은 **보안 무시**이므로, `--cacert` 사용을 권장합니다.

---

## 3️⃣ 연결 확인 결과

### ✅ 성공:
- JSON 또는 XML 형식 응답 수신 (장비 정보 등)

### ❌ 실패:
| 에러 메시지 | 원인 |
|-------------|------|
| curl (35) | TLS 인증 실패 (파일 불일치, 서버 문제) |
| curl (58) | 인증서/키 불일치 |
| 401 Unauthorized | 인증은 성공, ID/비번 오류 가능성 |

---

## 🔑 개인 키가 없는 경우 대처법

| 시나리오 | 대처 방안 |
|----------|-----------|
| 인증서만 있고 .key 없음 | 장비 웹 UI에서 **새 인증서 발급** |
| PEM만 있고 PFX 없음 | `.key` 파일도 함께 받았는지 확인 또는 새로 발급 |
| 전달받은 .crt만 있음 | 발급자에게 `.key` 또는 `.pfx` 요청 |

---

## ✅ 권장 구성 요약

| 목적 | 필요한 파일 | 명령어 구성 |
|------|-------------|--------------|
| 통신만 성공 | client.crt + client.key | `-k` 옵션 필수 |
| **보안 포함 (권장)** | client.crt + client.key + ca.pem | `--cacert` 사용 |

---

## 📂 예시 파일 구조 (`C:\\path\\to\\certs\`)
```
C:\\path\\to\\certs\
├── ca.pem
├── client.crt
├── client.key
└── client.pfx
```

> 참고: PEM 형식은 메모장으로 열었을 때 `-----BEGIN CERTIFICATE-----` 등 텍스트가 보임
