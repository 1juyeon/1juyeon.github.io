---
title: "[Note] HTTPS 인증 방식별 통신 전략 및 구현 요약"
date: 2025-06-25 20:00:00 +0900
categories: [note]
layout: single
---


# 📘 HTTPS 인증 방식별 통신 및 구현 전략 정리 (Basic/Digest vs mTLS)

이 문서는 HTTPS 통신을 사용하는 카메라 API 환경에서 인증 방식에 따라 어떤 설정이 필요한지를 정리한 문서입니다. Basic, Digest 인증과 mTLS(클라이언트 인증서 기반)의 차이점, C++ 코드 변경 포인트, curl 명령어 예시 등을 구체적으로 설명합니다.

---

## ✅ 시나리오 1: HTTPS + Basic/Digest 인증 (ID/비밀번호 인증 방식)

### 🔍 개요
- 가장 일반적인 인증 방식
- 카메라 API가 사용자 ID와 비밀번호를 요구하는 방식
- 인증 방식은 서버에 따라 Basic 또는 Digest일 수 있음
- HTTPS 기반이면 Basic 인증도 안전하게 사용 가능 (TLS로 암호화됨)

### ⚙️ 인증 방식 처리 방법
- libcurl에서 `CURLAUTH_ANYSAFE`를 사용하여 Digest, Basic, NTLM 등의 인증 방식 중 가장 안전한 방식 자동 선택
- Digest 우선, Basic은 HTTPS에서만 허용

### 🧠 CURLAUTH_ANYSAFE 설명
| 옵션 | 설명 |
|------|------|
| `CURLAUTH_ANY` | 서버가 제시하는 인증 방식 중 가장 보안이 강한 방식 선택 (Basic 포함) |
| `CURLAUTH_ANYSAFE` | Basic 인증을 제외한 안전한 인증만 허용 (단, HTTPS인 경우 Basic도 허용) |

### 🛠 C++ 코드 예시
```cpp
void CSamsungCamCurlMgr::MakeCurlHeader(curl_slist ** header_list, CURL * curl, int nProtocol)
{
    curl_easy_setopt(curl, CURLOPT_HTTPAUTH, CURLAUTH_ANYSAFE);

    char userpwd[URL_MAX_LEN] = { 0, };
    sprintf_s(userpwd, "%s:%s", m_strID.c_str(), m_strOldPW.c_str());
    curl_easy_setopt(curl, CURLOPT_USERPWD, userpwd);

    curl_easy_setopt(curl, CURLOPT_HEADER, 0);
    curl_easy_setopt(curl, CURLOPT_FOLLOWLOCATION, 1);
    curl_easy_setopt(curl, CURLOPT_WRITEFUNCTION, WriteCallBack);
    curl_easy_setopt(curl, CURLOPT_WRITEDATA, &m_WriteCallbackParam);
}
```

### 🧪 CMD 테스트 명령어
```bash
# HTTPS + Basic 인증 테스트
curl --anyauth -u "admin:password123" "https://<카메라_IP>/stw-cgi/system.cgi?msubmenu=deviceinfo&action=view"

# HTTPBin을 활용한 테스트
curl --anyauth -u "user:pass" "https://httpbin.org/basic-auth/user/pass"
curl --anyauth -u "user:pass" "https://httpbin.org/digest-auth/qop/user/passwd"
```

---

## ✅ 시나리오 2: HTTPS + mTLS 인증 (클라이언트 인증서 기반 인증 방식)

### 🔍 개요
- 일부 카메라에서는 클라이언트의 신원을 인증서 기반으로 검증 (Mutual TLS)
- TLS 핸드셰이크 과정에서 클라이언트 인증서를 요구함
- 인증서 없으면 TLS 연결 자체가 실패함

### 🧾 필요한 인증 파일
| 파일 | 설명 |
|------|------|
| `ca.pem` | 서버 인증서의 CA 루트 인증서 (서버 신뢰용) |
| `client.crt` | 클라이언트 인증서 |
| `client.key` | 클라이언트 개인 키 |
| (선택) `client.pfx` | 인증서와 키가 통합된 PFX 형식 |

### 🛠 C++ 코드 예시
```cpp
void CSamsungCamCurlMgr::MakeCurlHeader(curl_slist ** header_list, CURL * curl, int nProtocol)
{
    // 서버 인증서 검증
    curl_easy_setopt(curl, CURLOPT_CAINFO, "C:\certs\ca.pem");

    // 클라이언트 인증서 및 개인키 설정
    curl_easy_setopt(curl, CURLOPT_SSLCERT, "C:\certs\client.crt");
    curl_easy_setopt(curl, CURLOPT_SSLKEY, "C:\certs\client.key");
    // 필요 시 개인키 비밀번호 설정
    // curl_easy_setopt(curl, CURLOPT_KEYPASSWD, "password");

    // HTTP 인증
    curl_easy_setopt(curl, CURLOPT_HTTPAUTH, CURLAUTH_ANYSAFE);
    char userpwd[URL_MAX_LEN] = { 0, };
    sprintf_s(userpwd, "%s:%s", m_strID.c_str(), m_strOldPW.c_str());
    curl_easy_setopt(curl, CURLOPT_USERPWD, userpwd);

    curl_easy_setopt(curl, CURLOPT_HEADER, 0);
    curl_easy_setopt(curl, CURLOPT_FOLLOWLOCATION, 1);
    curl_easy_setopt(curl, CURLOPT_WRITEFUNCTION, WriteCallBack);
    curl_easy_setopt(curl, CURLOPT_WRITEDATA, &m_WriteCallbackParam);
}
```

### 🧪 CMD 테스트 명령어
```bash
# 클라이언트 인증서, 개인 키, CA 포함
curl --cacert ca.pem --cert client.crt --key client.key --anyauth -u "admin:password" "https://<카메라_IP>/stw-cgi/system.cgi?msubmenu=deviceinfo&action=view"

# PFX 파일 사용 시
curl --cacert ca.pem --cert-type P12 --cert client.pfx --pass "your_pfx_password" --anyauth -u "admin:password" "https://<카메라_IP>/stw-cgi/system.cgi?msubmenu=deviceinfo&action=view"
```

---

## 📌 FAQ

### Q. Basic 인증은 https에서 안전한가요?
A. 예. HTTPS는 전송 데이터를 TLS로 암호화하므로 Basic 인증도 안전합니다. 하지만 HTTP에서는 절대 사용하면 안 됩니다.

### Q. 인증서 없이 mTLS 통신이 가능한가요?
A. 불가능합니다. 클라이언트 인증서(.crt)와 개인 키(.key 또는 .pfx)가 반드시 필요합니다.

### Q. 인증서를 파일 없이 메모리에서 직접 전달할 수 있나요?
A. 가능합니다. libcurl의 `CURLOPT_SSLCERT_BLOB`, `CURLOPT_SSLKEY_BLOB`을 사용하면 메모리 데이터로 인증서/키를 전달할 수 있습니다.

---

## 🧾 최종 요약 비교표

| 항목 | Basic/Digest (HTTPS) | mTLS 인증 (클라이언트 인증서) |
|------|----------------------|-------------------------------|
| 인증 방식 | ID + PW | 인증서 + ID + PW |
| 보안 수준 | 중간 (HTTPS 전제 시 안전) | 매우 높음 |
| TLS 핸드셰이크 영향 | 없음 | 클라이언트 인증서 없으면 실패 |
| 필요한 파일 | 없음 | `ca.pem`, `client.crt`, `client.key` |
| CURL 인증 옵션 | `--anyauth` | `--cacert`, `--cert`, `--key`, `--anyauth` |
| C++ 옵션 | `CURLOPT_HTTPAUTH`, `CURLOPT_USERPWD` | 위 + `CURLOPT_SSLCERT`, `CURLOPT_SSLKEY`, `CURLOPT_CAINFO` |

---

## ✅ 권장 구현 전략

- 인증 방식이 확실하지 않은 경우 → `CURLAUTH_ANYSAFE` 사용
- 서버에서 클라이언트 인증서를 요구하는 경우 → 인증서 및 키 파일 제공
- 테스트 시 항상 `curl` 명령어로 먼저 시도하여 인증 구조를 확인

---

## 🔄 추가 정리: 인증 방식별 C++ 및 curl 테스트 최종 정리

### ✅ 시나리오 1: HTTPS + Basic/Digest 인증 (ID/비밀번호만 사용)

#### C++ 코드 예시
```cpp
void CSamsungCamCurlMgr::MakeCurlHeader(curl_slist ** header_list, CURL * curl, int nProtocol)
{
    curl_easy_setopt(curl, CURLOPT_HTTPAUTH, CURLAUTH_ANYSAFE);
    char userpwd[URL_MAX_LEN] = { 0, };
    sprintf_s(userpwd, "%s:%s", m_strID.c_str(), m_strOldPW.c_str());
    curl_easy_setopt(curl, CURLOPT_USERPWD, userpwd);
    curl_easy_setopt(curl, CURLOPT_HEADER, 0);
    curl_easy_setopt(curl, CURLOPT_FOLLOWLOCATION, 1);
}
```

#### CMD 테스트 예시
```bash
curl --anyauth -u "admin:your_password" "https://<카메라_IP>/..."
curl --anyauth -u "user:pass" "https://httpbin.org/basic-auth/user/pass"
```

---

### ✅ 시나리오 2: HTTPS + mTLS 인증 (클라이언트 인증서 + ID/비밀번호)

#### C++ 코드 예시
```cpp
void CSamsungCamCurlMgr::MakeCurlHeader(curl_slist ** header_list, CURL * curl, int nProtocol)
{
    curl_easy_setopt(curl, CURLOPT_CAINFO, "C:\certs\ca.pem");
    curl_easy_setopt(curl, CURLOPT_SSLCERT, "C:\certs\client.crt");
    curl_easy_setopt(curl, CURLOPT_SSLKEY, "C:\certs\client.key");
    curl_easy_setopt(curl, CURLOPT_HTTPAUTH, CURLAUTH_ANYSAFE);
    char userpwd[URL_MAX_LEN] = { 0, };
    sprintf_s(userpwd, "%s:%s", m_strID.c_str(), m_strOldPW.c_str());
    curl_easy_setopt(curl, CURLOPT_USERPWD, userpwd);
}
```

#### CMD 테스트 예시
```bash
curl --cacert ca.pem --cert client.crt --key client.key --anyauth -u "admin:your_password" "https://<카메라_IP>/..."

curl --cacert C:\certs\ca.pem --cert C:\certs\client.crt --key C:\certs\client.key --anyauth -u "admin:SF6!DKEur" "https://10.10.240.71/stw-cgi/system.cgi?msubmenu=deviceinfo&action=view"
```

---

## 🔍 추가 개념 정리

### ✅ CURLAUTH_ANY vs CURLAUTH_ANYSAFE
| 항목 | CURLAUTH_ANY | CURLAUTH_ANYSAFE |
|------|---------------|------------------|
| Basic 인증 (HTTP) | 허용 (위험) | 차단 |
| Basic 인증 (HTTPS) | 허용 | 허용 |
| Digest, NTLM, Negotiate | 모두 허용 | 모두 허용 |
| 추천 여부 | ❌ 보안에 민감한 환경에서는 권장 안 함 | ✅ 안전한 기본값으로 추천 |

---

## 🛠 TLS 인증 실패 시 curl -v 결과 비교

| 구분 | HTTPS + Digest/Basic | HTTPS + mTLS 인증 실패 |
|------|----------------------|--------------------------|
| TLS 연결 | 성공 | 실패 (Handshake 실패) |
| 인증 요청 | `WWW-Authenticate` 헤더 제공 | 없음 (TLS에서 종료됨) |
| curl 오류 코드 | 없음 또는 401 표시 | (35), (94), handshake failure |
| 확인 방법 | `curl -v -k` 로 요청하여 HTTP 헤더 확인 | `curl -v -k` 사용 시 TLS 단계에서 실패 메시지 확인 |

---

## 📂 인증서 파일은 어디에?

| 위치 | 설명 |
|------|------|
| 카메라 웹 UI | `인증서 관리` 메뉴에서 다운로드 |
| VMS 시스템 | SSM, WAVE 같은 관리 시스템의 설정 메뉴 |
| SI 업체/기술 지원 | 시스템 구축/설치 담당자 또는 제조사 문의 |

---

## 🧪 mTLS 테스트용 curl 명령어 요약

### PEM 형식
```bash
curl --cacert ca.pem --cert client.crt --key client.key --anyauth -u "admin:password" "https://<카메라_IP>/..."
```

### PFX 형식
```bash
curl --cacert ca.pem --cert-type P12 --cert client.pfx --pass "pfx_password" --anyauth -u "admin:password" "https://<카메라_IP>/..."
```

---

## 🔐 클라이언트 인증서와 개인 키만으로 충분한가?

정답은: **"기능적으로는 가능하나, 보안상 충분하지 않습니다."**

---

### 🎯 목적 1: 통신 성공만을 위한 최소 구성 (인증서 2개)

#### 필요한 파일
- 클라이언트 인증서 (.crt 또는 .pem)
- 클라이언트 개인 키 (.key)

#### 동작 원리
- 서버가 클라이언트 인증서를 요구하는 경우, 이 두 파일만으로 TLS Handshake를 완료할 수 있습니다.
- 다만, 서버의 신뢰성을 검증하지 않기 때문에 `-k` 또는 `--insecure` 옵션이 **반드시 필요**합니다.

#### curl 예시
```bash
curl --cert client.crt --key client.key -k --anyauth -u "admin:SF6!DKEur" "https://10.10.240.71/..."
```

> 📌 `-k` 옵션은 "서버 인증서의 유효성 검사를 건너뛰겠다"는 의미입니다. 즉, 중간자 공격(MITM)에 취약합니다.

---

### ✅ 목적 2: 안전하고 올바른 상호 인증 (인증서 3개 - 권장)

#### 필요한 파일
- 클라이언트 인증서 (.crt 또는 .pem)
- 클라이언트 개인 키 (.key)
- CA 인증서 (.pem)

#### 동작 원리
- 클라이언트는 인증서/개인키로 자신을 증명하고,
- 서버는 CA 인증서로 검증되어야만 통신을 허용합니다.
- `-k` 없이 `--cacert` 옵션을 사용하여 신뢰된 서버와만 통신하게 됩니다.

#### curl 예시
```bash
curl --cacert ca.pem --cert client.crt --key client.key --anyauth -u "admin:SF6!DKEur" "https://10.10.240.71/..."
```

---

### 🧾 최종 정리표

| 구성 | 기능적 연결 | 보안적 신뢰 | curl 옵션 |
|------|--------------|----------------|------------|
| 인증서 + 키 (2개) | ✅ 연결됨 | ❌ 서버 위조 가능성 | `-k` 필요 |
| 인증서 + 키 + CA (3개) | ✅ 연결됨 | ✅ 안전하게 서버 검증 | `--cacert` 사용 |

> ✅ **보안이 중요한 환경이라면 반드시 세 가지 파일을 모두 사용하세요!**
