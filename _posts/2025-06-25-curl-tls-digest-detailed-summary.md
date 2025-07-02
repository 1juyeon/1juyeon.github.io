---
title: "curl Digest 인증과 TLS 백엔드 개념 정리"
date: 2025-06-25 20:00:00 +0900
categories: [note]
layout: single
---


# 📚 curl Digest 인증, TLS 백엔드, Schannel, SSPI 관련 개념 및 실습 정리

(작성일: 최신 기준)

---

## ✅ 1. TLS와 Digest 인증은 다른가요?

### ✔️ 결론: 네, 완전히 다릅니다. 서로 다른 계층, 다른 역할을 수행합니다.

### 🔐 TLS (Transport Layer Security)
- TLS는 인터넷에서 데이터를 **안전하게 주고받기 위해 전체 연결을 암호화**하는 보안 프로토콜입니다.
- HTTP, SMTP, FTP 등 다양한 프로토콜과 함께 사용될 수 있지만 가장 널리 쓰이는 형태는 HTTPS입니다.
- TLS는 **전송 계층(OSI 4계층)**에 속하며, 클라이언트-서버 간 데이터를 암호화하여 **도청, 변조, 위조 방지**를 목표로 합니다.
- TLS는 인증서 기반으로 동작하며, 보통 공개키(Public Key)와 개인키(Private Key)를 이용해 초기 핸드셰이크 이후 대칭키를 설정합니다.

### 👤 Digest 인증
- HTTP 프로토콜에서 사용되는 사용자 인증 방식 중 하나입니다.
- 사용자 ID와 비밀번호를 직접 전송하지 않고, 서버에서 보내온 `nonce` 값과 함께 해시(SHA-1 또는 SHA-256 등) 계산을 수행해 전송합니다.
- 즉, **로그인 과정만 암호화**된 것입니다.
- Digest 인증은 **응용 계층(OSI 7계층)**에 속하며, `Authorization` 헤더에 관련 정보가 포함됩니다.
- `TLS` 없이 `Digest`만 사용하는 경우에도 로그인 정보는 해시로 보호되지만, **데이터 내용 전체는 평문으로 전송될 수 있어 위험**합니다.

---

## ✅ 2. TLS 백엔드란?

### ✔️ 정의
curl이 TLS/SSL 보안 연결을 수행할 때 실제로 암호화와 인증서를 처리하는 하위 라이브러리입니다.

### 📚 주요 백엔드 설명

#### 🔒 OpenSSL
- 가장 널리 사용되는 암호화 라이브러리
- TLS 1.3, 다양한 해시 알고리즘, RSA/ECDSA 등 최신 보안 기술을 빠르게 지원
- Windows에서는 별도 설치가 필요함
- 강력하지만 설정과 사용이 다소 복잡할 수 있음

#### 🧬 LibreSSL
- OpenSSL을 포크(fork)하여 만든 프로젝트 (OpenBSD 팀에서 개발)
- 보안성과 코드 안정성을 강조
- OpenSSL보다 가볍고 덜 복잡한 구조

#### 🧩 Schannel (Secure Channel)
- Windows 운영체제에 기본 내장된 TLS/SSL 백엔드
- Internet Explorer, Microsoft Edge 등에서도 사용됨
- Windows의 인증서 저장소를 그대로 사용할 수 있음 (CA 관리 간소화)
- 성능 최적화가 잘 되어 있어 Windows 환경에서는 매우 효율적

---

## ✅ 3. Schannel을 보조 백엔드로 사용하는 이유

- 보조 백엔드는 기본 백엔드(OpenSSL 등)로 작동 중, 특정 상황에서 보완적 역할을 수행
- 예: 인증서 신뢰 오류 발생 시 Schannel을 통해 Windows 시스템 인증서 저장소를 이용하여 문제 해결 가능
- Windows 시스템과의 통합성이 높아 조직 내 보안 체계(예: 사내 루트 인증서 등)와 호환성이 좋음
- 인증서 관리, 성능, 호환성 측면에서 실용적 대안이 됨

---

## ✅ 4. SSPI란?

### ✔️ 정의
SSPI(Security Support Provider Interface)는 Windows에서 제공하는 **통합 인증 시스템**입니다.

- Microsoft의 NTLM, Kerberos, Negotiate 등 다양한 인증 프로토콜을 하나의 인터페이스로 제공
- 클라이언트 인증 시 사용자 입력 없이 현재 로그인한 Windows 계정 정보를 사용 가능
- curl에서 SSPI가 활성화된 상태라면, Digest 인증 등도 이 API를 통해 처리하게 됩니다
- 다만, SSPI 기반 Digest 인증은 SHA-256 같은 최신 알고리즘은 curl에서 미지원입니다

---

## ✅ 5. Schannel과 SSPI가 충돌하는 이유

- Schannel도 내부적으로 SSPI를 사용하는 경우가 있어, 둘을 동시에 사용할 경우 **함수 중복, 심볼 충돌, 초기화 순서 오류** 등이 발생할 수 있습니다.
- 특히 mingw-w64 환경에서 curl을 빌드할 때 이런 충돌이 잦았습니다.
- 결과적으로 curl 8.6.0_3에서는 충돌을 회피하기 위해 SSPI를 끄고 Schannel만 활성화한 버전이 배포되었습니다.

---

## ✅ 6. curl 8.6.0_3 버전 특징

| 항목 | 상태 |
|------|------|
| Schannel | 활성화됨 |
| SSPI | 비활성화됨 |
| Digest SHA-256 | curl 내부 처리기 사용 시 가능 |
| LibreSSL | 이 버전은 LibreSSL을 기본 TLS 백엔드로 사용 |

즉, Digest 인증은 SSPI가 꺼진 상태이므로 curl이 자체적으로 처리하며, 이 경우 SHA-256 해시 알고리즘도 지원됩니다.

---

## ✅ 7. CMake 패치로 충돌 해결

- 충돌의 원인은 빌드 시 `USE_SSPI`, `USE_SCHANNEL` 같은 플래그들이 정리되지 않아서 발생
- 패치 이후:
  - Schannel과 SSPI를 동시에 사용할 수 있도록 구성 분리
  - 링크 충돌 방지 설정 추가
  - 초기화 루틴 정비

---

## ✅ 8. Digest SHA-256 테스트 방법

### 명령어
```bash
curl --digest -u admin:1234 -v https://your-server.com/protected
```

### 전제 조건
- 서버가 다음과 같은 헤더를 응답해야 함:
```
WWW-Authenticate: Digest realm="example", algorithm=SHA-256, nonce="..."
```

### SHA-256 사용 여부 확인
- `-v` 옵션 로그에서 Authorization 헤더의 `algorithm=SHA-256` 포함 여부로 확인

---

## ✅ 9. Schannel은 Digest SHA256과 관계 있을까?

- ❌ 직접적인 관계 없음
- Digest 인증은 HTTP 응용 계층에서 처리되며, Schannel은 TLS 전송 계층 보안에만 관여
- 따라서 Digest 인증이 SHA-256으로 동작하는지 여부는 **SSPI 또는 curl 내부 처리기**에 달려 있습니다

---

## ✅ 10. Windows에서 OpenSSL 기반 curl 사용 가능 여부

- 기본 Windows curl은 Schannel 또는 LibreSSL 기반이 많음
- 그러나 OpenSSL 기반 curl도 공식적으로 제공됨 (https://curl.se/windows/)
- 설치 후 `curl --version`에서 `OpenSSL`이 출력되면 OpenSSL 기반임

---

## ✅ 최종 요약

| 항목 | 내용 |
|------|------|
| TLS vs Digest 인증 | TLS는 통신 암호화 (전송 계층), Digest는 로그인 인증 (응용 계층) |
| TLS 백엔드 | OpenSSL, LibreSSL, Schannel 중 선택 가능 |
| Schannel 보조 사용 이유 | 인증서 저장소, 성능, 호환성 측면에서 장점 |
| SSPI 역할 | Windows 로그인 정보를 인증에 활용 |
| curl 8.6.0_3 | Schannel ON + SSPI OFF + LibreSSL, Digest SHA-256 가능 |
| Digest SHA-256 확인 방법 | curl -v 로 Authorization 헤더에 algorithm 확인 |
| Schannel과 Digest 관계 | 직접 관계 없음 (계층 분리) |
| Windows에서 OpenSSL curl 사용 | 가능, 별도 다운로드 필요 |

---
