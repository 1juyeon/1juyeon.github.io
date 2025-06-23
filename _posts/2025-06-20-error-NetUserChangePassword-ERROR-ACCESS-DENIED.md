---
title: "250620 NetUserChangePassword ERROR ACCESS DENIED"
date: 2025-06-20 10:00:00 +0900
categories: [error]
tags: [windows, NetUserChangePassword, api, "24H2", windows server, "21H2", NetUserSetInfo]
layout: single
---

# Windows 11 비밀번호 변경 (`NetUserChangePassword`) 관련 권한 및 정책 정리

## 📌 1. 개요

Windows 11 24H2 환경에서 `NetUserChangePassword` 함수를 사용하여 원격 PC 또는 로컬 PC의 사용자 비밀번호를 변경하고자 할 때, **`ERROR_ACCESS_DENIED (5)`** 오류가 발생하는 현상에 대해 테스트를 진행하였다.

이 문서는 JNI + DLL을 통해 해당 함수를 호출하는 환경에서 어떤 보안 정책과 권한이 영향을 미치는지를 실험하고, Windows 11 24H2 버전에서 발생하는 제한 사항을 정리한 것이다.

---

## 🧪 2. 테스트 환경

| 항목 | 값 |
|------|----|
| 클라이언트 PC | Windows 11 24H2 |
| 서버/대상 PC | Windows 11 24H2 / Windows Server 2022 21H2 |
| 호출 방식 | Java → JNI → DLL(C++) |
| 사용 함수 | `NetUserChangePassword` |
| IPC 연결 | `WNetAddConnection2` or `net use` |
| 실행 환경 | 톰캣 서비스 (서비스 계정 실행) |

---

## 🔍 3. 주요 확인 항목 및 명령어 정리

### ✅ 3.1 사용자 계정 조건 확인

```bash
# 계정 정보 확인
net user <계정명>
```

**확인 포인트:**
- `사용자가 암호를 바꿀 수도 있음 : 예` → ✅
- `암호 필요 : 예` → ✅
- 계정이 `Administrators` 그룹에 포함되어 있는지 확인

```bash
# 관리자 그룹 구성원 확인
net localgroup administrators
```

---

### ✅ 3.2 IPC 연결 확인 (WNet / net use)

**C++ 내에서는** `WNetAddConnection2` 함수를 사용해 IPC 연결을 시도

```cpp
NETRESOURCE nr = {0};
nr.dwType = RESOURCETYPE_ANY;
nr.lpRemoteName = L"\\192.168.40.90\IPC$";
WNetAddConnection2(&nr, L"관리자비밀번호", L"관리자계정", 0);
```

**CMD에서도 직접 확인 가능:**

```cmd
net use \192.168.40.90\ipc$ /user:관리자계정 관리자비밀번호
```

**성공 시:** `명령을 잘 실행했습니다.`  
**실패 시:** 오류 코드 1326 (인증 실패) 또는 5 (액세스 거부)

---

### ✅ 3.3 톰캣 서비스 실행 계정 확인

- `LocalSystem`으로 실행될 경우 권한 문제 발생 가능
- 톰캣 서비스를 **명시적인 로컬 관리자 계정**으로 실행하도록 설정

```powershell
# 서비스 실행 계정 확인 (PowerShell)
Get-WmiObject win32_service | Where-Object { $_.Name -eq 'Tomcat9' } | Select-Object Name, StartName
```

**서비스 로그온 계정 변경 방법:**
1. `services.msc` 실행
2. Tomcat 서비스 → 속성 → [로그온] 탭
3. "다음 계정" → 로컬 관리자 계정 입력

---

### ✅ 3.4 보안 정책 (`secpol.msc`) 확인

> ⚠️ Windows 11 Home에서는 `secpol.msc` 사용 불가

실행:
```cmd
secpol.msc
```

확인할 항목들:
- **보안 설정 > 로컬 정책 > 사용자 권한 할당**
  - `로컬에서 로그온 허용`
  - `네트워크에서 이 컴퓨터에 액세스 허용`
- **보안 설정 > 보안 옵션**
  - `계정: 로컬 계정의 네트워크 로그온 사용 제한` → **사용 안 함**

---

### ✅ 3.5 로컬 그룹 정책 편집기 (`gpedit.msc`) - 일부 Pro 버전 이상만 가능

실행:
```cmd
gpedit.msc
```

경로:
```
컴퓨터 구성 > Windows 설정 > 보안 설정 > 로컬 정책 > 보안 옵션
```

중요 항목:
- `계정: 로컬 계정의 공유 및 보안 모델`
  - 클래식 - 로컬 사용자 인증만 허용 (✅)
- `계정: 관리자 계정 이름 바꾸기`
- `계정: 게스트 계정 상태`

---

### ✅ 3.6 이벤트 로그 분석 (비밀번호 변경 실패 로그 추적)

실행:
```cmd
eventvwr.msc
```

경로:
```
Windows 로그 > 보안
```

- **Event ID 4648**: 명시적 자격 증명을 사용한 로그온 시도
- **오류 예시:** `Status: 0xC0000022 (STATUS_ACCESS_DENIED)`

---

## ⚠️ 4. 발견된 문제 및 분석

- `NetUserChangePassword`는 **충분한 권한, IPC 연결, 서비스 계정 설정이 완료되어 있어도 Windows 11 24H2에서 실패**하는 사례가 지속됨
- Windows 11 23H2 이하에서는 동일 코드가 동작하였다는 사용자 사례 있음 (직접 실험 제외)

---

## ⚠️ 4-1. 추가 확인해본 항목

- IPC 연결 확인
  ```cmd
  net use \\192.168.40.90\ipc$ /user:administrator core2580@!
  ``` 
- UAC 설정 확인
  - HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System `LocalAccountTokenFilterPolicy (DWORD)` 생성 또는 수정 → **값 1로 설정**
- SMB 설정 확인 
    - 윈도우 최신 버전부터 아래와 같은 기본 설정이 비활성화 되어 나온다는 내용 확인을 위해
    - 제어판 - 프로그램 및 기능 - Windows 기능 켜기/끄기 - SMB 1.0/CIFS 파일 공유 지원 체크 ( 재부팅 필요 )
  
- 인바운드 445 포트 관련 규칙 확인
- 445 포트 열려있는지 확인
  ```cmd
  telnet 192.168.40.90 445
  ``` 
- 계정 확인을 위한 명령어들
   ```cmd
  net user 계정이름 패스워드 ---> 패스워드변경
  net user ----> pc 유저들
  net user 계정이름 ---> 계정에 대한 정보들
  net localgroup ----> 어떤 그룹들 있느지
  net localgroup 특정 그룹 ---> 그룹에 누구 있는지
  net user 계정이름 /active:yes 활성화 no 는 비활성환데 비활성화 잘 안 되는듯
  ```  
---

## 🔐 5. 결론

### ❗ 결론적으로

> **Windows 11 24H2에서는 보안 정책이 강화되어, `NetUserChangePassword` 함수 사용 시 `ERROR_ACCESS_DENIED`가 발생할 수 있음.**

이는 내부적으로 MS에서 명시하지 않았지만, 시스템 커널 보안 또는 자격 증명 위임 모델이 변경되었을 가능성이 있으며, 기존 IPC 인증 후의 자격 토큰만으로는 비밀번호 변경이 거부되는 사례로 추정됨.

- 관련 참고 링크 및 마이크로소프트 문의 링크
  ##### * 24H2 오류 관련 사례
  >  https://answers.microsoft.com/en-us/windows/forum/all/windows-11-version-24h2-update-issue-with/07366110-8ee9-422c-815a-f6909414900b?utm_source=chatgpt.com

    ##### * 마이크로소프트 직접 문의 
  >  https://answers.microsoft.com/en-us/windows/forum/windows_11-other_settings/netuserchangepassword-returns-erroraccessdenied-on/de3b0bf0-1855-486d-98d6-f954c8f677b6?messageId=b0770387-0256-4d25-ade4-de785803cfac
  
  

---

## 💡 6. 대안 및 권장 방법

| 방법 | 설명 |
|------|------|
| `NetUserSetInfo` | 권한이 충분한 관리자 계정을 통해 직접 사용자 비밀번호 업데이트 |
| PowerShell 원격 스크립트 | `Set-LocalUser -Name "user" -Password` 등 |
| WMI or PsExec 활용 | 원격 명령 실행을 통한 변경 |
| 서비스 실행 권한 강화를 위한 Group Policy | 관리자 계정으로 서비스 실행 + 권한 위임 |

---

## 📝 7. 블로그/보고서용 요약 문구

> Windows 11 24H2에서는 `NetUserChangePassword` 함수가 내부 보안 정책 강화로 인해 관리자 권한 및 IPC 인증을 모두 충족하더라도 `ERROR_ACCESS_DENIED` 오류가 발생할 수 있으며, 이는 MS 내부 정책 변경에 기인한 것으로 추정된다.

## 8. 결론
> NetChangeUserPassword 함수에서 NetUserSetInfo (USER_INFO_1003) 함수로 변경

| NetUserChnagePassword | NetUserSetInfo |
|------|----|
| 비밀번호 변경 (Change) | 비밀번호 재설정 (Reset) |
| "기존 열쇠를 새 열쇠로 교체" | "마스터키로 새 잠금장치를 설치" |
| 기존 비밀번호를 알아야함 | 관리자 권한만 있으면 됨 |
---