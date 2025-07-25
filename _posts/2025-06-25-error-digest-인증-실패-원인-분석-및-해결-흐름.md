---
title: "[Error] Digest 인증 실패 원인 분석 및 해결 흐름"
date: 2025-06-25 20:00:00 +0900
categories: [error]
layout: single
---


# 📘 Digest 인증 테스트 중 발생 가능한 문제 분석 및 확인 흐름

## 🧭 문제 상황의 시작

Hanwha Vision 카메라 장치에 HTTPS 기반의 Digest 인증 API 요청을 보내는 과정에서 아래와 같은 명령어를 사용했습니다:

```bash
curl --digest -u "admin:0000" -k "https://192.168.40.157:443/stw-cgi/system.cgi?msubmenu=deviceinfo&action=view"
```

하지만 응답 결과로 다음과 같은 오류가 발생했습니다:

```
schannel : InitializeSecurityContext failed: SEC_E_QOP_NOT_SUPPORTED (0x8009030A)
curl: (94) An authentication function returned an error
```

### 💡 의문 제기
- ID/PW는 브라우저에서 정상 로그인 가능하므로 틀릴 가능성은 낮음
- 명령어 형식 및 인증 옵션도 정확히 입력됨
- 서버는 Digest 인증을 `SHA-256`과 `MD5` 두 방식 모두 지원한다고 명시함
- 이미 LibreSSL 기반 curl을 사용하고 있음에도 동일한 오류 발생

---

## 🔎 이런 조건이 문제가 될 수 있습니다

### 1. `Digest 인증 + SHA-256 알고리즘` + Windows 환경

이런 조합에서 문제가 발생할 수 있다는 점이 관찰되었습니다. 특히 curl이 인증을 내부 처리하지 않고, Windows 보안 기능인 **SSPI(Security Support Provider Interface)**에 위임하는 경우, 해당 기능이 최신 Digest 인증 알고리즘(SHA-256)을 지원하지 못할 수 있습니다.

즉, 서버는 SHA-256 또는 MD5 중 선택하라고 하였지만, 클라이언트가 SHA-256을 우선적으로 선택했고, 이를 처리하는 Windows 인증 모듈이 거부했을 가능성이 있습니다.

```
→ SSPI: Digest 인증을 처리하는 wdigest.dll이 SHA-256을 지원하지 않을 수 있음
```

---

## 🧪 테스트로 확인할 수 있는 것들

### ✅ 테스트 1: Postman Digest Auth 요청
Postman은 Windows의 SSPI를 사용하지 않고 자체 인증 처리 로직을 사용하기 때문에, 동일한 Digest 인증 요청이 정상 동작하는지 테스트할 수 있습니다.

- 성공 시: curl이 아닌 Windows 환경 문제로 추정 가능
- 실패 시: 인증 정보 또는 서버 응답 자체 문제일 수 있음

### ✅ 테스트 2: curl을 리눅스 환경(WSL 등)에서 실행
리눅스 환경의 curl은 OpenSSL 기반으로 Digest 인증을 자체적으로 처리하므로, Windows SSPI 영향 없이 테스트 가능합니다.

```bash
curl --digest -u "admin:0000" -k "https://..."
```

### ✅ 테스트 3: 서버가 SHA-256이 아닌 MD5만 제공하는 경우
과거 테스트에서는 같은 curl 환경에서 Digest 인증이 성공했던 적이 있을 수 있습니다. 이는 해당 서버가 SHA-256 없이 MD5만 제공한 상황이었을 가능성도 있으며, 이 경우 Windows SSPI도 인증을 정상적으로 처리할 수 있습니다.

---

## 🔄 지금까지의 흐름 정리

1. curl 명령어로 Digest 인증 요청 → 오류 발생
2. LibreSSL 기반 curl로도 동일 오류 → 환경 자체의 인증 처리 문제 의심
3. 오류 메시지에 여전히 `schannel`이 등장 → 내부적으로 Windows SSPI가 동작 중
4. Postman이나 리눅스에서는 인증이 가능하다면 → curl + Windows 조합의 제한

---

## 🔁 대응 가능 방법들

| 방법 | 설명 | 특이사항 |
|------|------|----------|
| ✅ Postman 사용 | 인증 처리에 Windows 기능 사용 안함 | GUI 기반 |
| ✅ WSL 환경 curl | Linux 기반 curl에서 Digest 인증 가능 | Schannel 완전 배제 |
| ⛔ SSPI 없는 curl 빌드 | 이론적으론 가능하나 거의 없음 | 현실적이지 않음 |
| ⚠️ C++ 연동 | curl 호출 대신, cpp-httplib 등으로 인증 로직 자체 구현 필요 | 권장 |

---

## 🧵 최종 제안

- 지금의 문제는 특정 조합(SHA-256 Digest + Windows curl/SSPI)에서 발생하는 인증 호환성 문제일 가능성이 있습니다.
- Postman, WSL, 또는 C++ 내 Digest 인증 구현을 통해 우회할 수 있으며,
- 정확한 원인 판단을 위해서는 반드시 Postman에서의 테스트 성공 여부를 확인하는 것이 중요합니다.
