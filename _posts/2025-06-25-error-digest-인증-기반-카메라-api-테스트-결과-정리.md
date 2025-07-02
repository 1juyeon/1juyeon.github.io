---
title: "[Error] Digest 인증 기반 카메라 API 테스트 결과 정리"
date: 2025-06-25 20:00:00 +0900
categories: [error]
layout: single
---

# 📋 Digest 인증 기반 카메라 API 테스트 결과 정리

## 🔧 테스트 환경
- **IP**: `192.168.40.157`
- **ID/PW**: `admin / 0000`
- **TLS 인증서 경고 무시 옵션**: `-k` 사용
- **Digest 인증 시도**: `--digest` 또는 `-digest` (※ `-digest`는 오타임)

---

## ✅ 테스트 결과 요약표

| 번호 | 명령어 설명 | 명령어 | 목적/특이사항 | 응답 요약 |
|------|-------------|--------|---------------|------------|
| 1 | 기존 Digest API 호출 (정상 문법) | `curl --digest -u admin:0000 -k "https://...deviceinfo&action=view"` | Digest 인증으로 장치 정보 조회 | ❌ `(94) 인증 함수 오류` |
| 2 | Digest API로 비밀번호 변경 시도 (오타 있음) | `curl -digest -u admin:0000 -k "https://...Password=0001@..."` | `-digest` → ❌ 잘못된 옵션 | ❌ `(94) 인증 함수 오류` |
| 3 | 인증서 검증 실패 테스트 | `curl -u admin:0000 "https://..."` | `-k` 미사용 → TLS 인증 실패 유도 | ❌ `(60) SEC_E_UNTRUSTED_ROOT` |
| 4 | 인증 누락 상태에서 호출 | `curl -u admin:0000 -k "https://.../stw-cgi"` | Digest 없이 Basic 인증 시도 | ❌ `401 Unauthorized` |
| 5 | Digest 인증 + 인증서 검증 실패 | `curl --digest -u admin:0000 "https://..."` | `-k` 빠짐 → TLS 오류 발생 | ❌ `(60) SEC_E_UNTRUSTED_ROOT` |
| 6 | 비밀번호 변경 (Digest 없음) | `curl -u admin:0000 -k "https://...Password=0001..."` | Digest 없이 호출 → 인증 실패 | ❌ `401 Unauthorized` |
| 8 | 인증 없이 호출 | `curl "https://..."` | 인증 없이 호출 | ❌ `(60) 인증서 검증 실패` |
| 9 | ID/PW에 쌍따옴표 사용 | `curl --digest -u "admin:0000" -k "https://..."` | 특수문자 대응 목적 | ❌ `(94) 인증 함수 오류` |
| 10 | Digest + -k 옵션으로 재시도 | `curl --digest -u "admin:0000" -k "https://..."` | 인증 정보, 인증서 무시 모두 포함 | ❌ `(94) SEC_E_QOP_NOT_SUPPORTED` |
| 11 | 인증 없이 -v 로 상세 정보 출력 | `curl -v -k "https://...?view"` | Digest 필요 여부 확인 | ❌ `401 Unauthorized` + Digest 정보 포함됨 |

---

## 🔍 테스트 목적과 해석

### 1. `--digest`
- Digest 인증 명시
- `curl --digest` 사용이 올바른 문법이며, `-digest`는 잘못된 옵션 (❌)

### 2. `-u user:pass`
- 사용자 인증 정보 입력
- 쌍따옴표(`"admin:0000"`)로 감싸는 것은 특수문자 포함 시 필수 (예: `!`, `@` 등)

### 3. `-k`
- 인증서 유효성 검사를 생략함
- 인증서가 사설(CA가 발급한 것이 아님)인 경우 필수

### 4. 응답 메시지 분석
- `curl: (94)` → 인증 함수 오류 (`SEC_E_QOP_NOT_SUPPORTED`: Windows SChannel에서 Digest QOP 미지원)
- `401 Unauthorized` → 인증 실패 (보통 ID/PW 또는 인증 방식 문제)
- `curl: (60)` → 인증서 체인 문제 (신뢰되지 않은 루트)

---

## 🧪 핵심 원인 분석

### 💡 문제 핵심
- Windows의 `schannel` 보안 패키지는 **Digest 인증의 `qop="auth"`** 방식에서 **SHA-256 또는 MD5 둘 중 어떤 것도 완전히 지원하지 않음** → `(94)` 오류 유발

### ✅ 해결 가능 방향
1. **OpenSSL 기반 curl 사용 (libcurl + OpenSSL)**  
   Windows 기본 curl은 `schannel` 기반이므로 Digest 지원이 제한됨  
   → OpenSSL 버전으로 교체 필요

2. **대안**
   - 리눅스 curl 또는 WSL curl 사용 권장
   - 또는 Postman / Python requests 등으로 Digest 인증 테스트

---

## 📘 참고: Digest 인증 응답 예시 헤더
```http
WWW-Authenticate: Digest realm="iPolis_...", charset="UTF-8", algorithm="SHA-256", qop="auth"
WWW-Authenticate: Digest realm="iPolis_...", charset="UTF-8", algorithm="MD5", qop="auth"
```
> 해당 장치는 SHA-256, MD5 두 알고리즘 모두 제공하고 있지만, Windows 기본 curl은 QOP 지원이 부족하여 실패함
