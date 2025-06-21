---
title: "250620 Windows Server 2022 설치 가이드"
date: 2025-06-20 10:00:00 +0900
categories: [note]
tags: [windows server, "21H2", "2022", 설치, Virual Machine, VM]
layout: single
---

# Windows Server 2022 설치 가이드

## 🔧 준비 단계
1. **설치 미디어 준비**
   - Windows Server 2022 ISO 파일 다운로드 (Microsoft 공식 사이트)
   - USB 메모리(최소 8GB)에 Rufus 등으로 부팅 가능한 설치 USB 생성

2. **하드웨어 요구 사항**
   - CPU: 최소 1.4 GHz, 64비트
   - RAM: 최소 512 MB (권장 2 GB 이상)
   - 디스크: 최소 32 GB 여유 공간
   - UEFI 지원, TPM 2.0 (Hyper-V, 보안 기능 이용 시)

---

## 🚀 설치 단계

### 1. 설치 미디어로 부팅
- USB 연결 후 서버 전원 켠 뒤 BIOS/UEFI에서 USB 부팅 설정

### 2. 언어 및 기타 설정
- **언어**, **시간/통화 형식**, **키보드 입력 방식** 선택  
  이후 **지금 설치(Install now)** 클릭

### 3. 라이선스 입력
- 제품 키 입력 또는 *나중에 입력(I don’t have a product key)* 클릭  
- 설치할 Windows Server 2022 에디션 선택 (Standard / Datacenter 등)

### 4. 설치 유형 선택
- **사용자 정의(Custom)** 선택  
  → 새 설치, 디스크 파티션 구성 가능

### 5. 디스크 파티션 설정
- 전체 디스크 사용하거나  
  → `새 파티션(New)` → 크기 지정 → **적용(Apply)** 클릭  
- **다음(Next)** 눌러 설치 진행

### 6. 설치 진행
- 파일 복사 → 기능 설치 → 구성 단계 자동 진행  
- 설치 완료 후 서버가 자동으로 재부팅됩니다.

### 7. 초기 설정
- **관리자 계정(Administrator)** 암호 설정  
- 로그인 후 필요한 **역할(Roles)** 및 **기능(Features)** 설치 가능 via 서버 관리자(Server Manager)

---

## ✅ 설치 종료 후 권장 작업

- **Windows 업데이트** 수행
- **네트워크 설정** (IP, DNS 등) 확인 및 구성
- **원격 데스크톱(RDP)** 설정 (필요 시)
- **백업 전략** 수립 및 스냅샷/복원 솔루션 구성

---

## 📝 요약
| 단계 | 설명 |
|------|------|
| 1. 준비 | ISO 다운로드 + 설치 USB 생성 |
| 2. 부팅 | BIOS에서 USB 부팅 |
| 3. 설정 | 언어 및 에디션 선택 |
| 4. 설치 | 디스크 분할 → 설치 완료 |
| 5. 설정 | 암호 설정 → 네트워크/업데이트 구성 |

---

---

---


# 🖥️ Oracle VirtualBox에 Windows Server 2022 설치 가이드

## 📌 개요
이 가이드는 VirtualBox를 이용해 Windows Server 2022를 가상 머신에 설치하는 과정을 누구나 이해할 수 있도록 단계별로 설명합니다.

---

## 1단계: 준비물

### 🔧 필수 준비물
- **Oracle VirtualBox**: https://www.virtualbox.org/
- **Windows Server 2022 ISO 파일**: [Microsoft Evaluation Center](https://www.microsoft.com/evalcenter/)

### 💡 팁
- ISO는 `영어(미국)` 버전으로 다운로드하면 설치가 수월합니다.
- 다운로드 용량은 약 5GB 이상이므로 미리 받아두세요.

---

## 2단계: 가상 머신 생성

### 📦 VirtualBox 설정
1. **VirtualBox 실행 → '새로 만들기' 클릭**
2. 이름: `Windows Server 2022`
3. 종류: `Microsoft Windows`
4. 버전: `Windows 2019 (64-bit)` 또는 `Other Windows (64-bit)`
5. 무인 설치 체크 해제 → 직접 설치 진행 추천
6. ISO 파일 선택 (다운로드한 ISO)
7. 메모리: **4GB 이상** 권장
8. CPU: **2개 이상**
9. 디스크 크기: **60GB 이상**

---

## 3단계: Windows Server 2022 설치

### 🧭 설치 절차
1. 가상 머신 시작 → ISO 자동 부팅
2. 설치 언어/키보드 설정 → `Next`
3. `Install Now` 클릭
4. **버전 선택 (중요!)**  
   `Standard Evaluation (Desktop Experience)` 또는  
   `Datacenter Evaluation (Desktop Experience)` 선택
5. 라이선스 동의 → `Next`
6. 설치 유형: `Custom`
7. 디스크 선택 → `Next`
8. 설치 진행 → 재부팅

---

## 4단계: 초기 설정 및 바탕화면

### 🔐 관리자 암호 설정
- 암호 입력 및 확인
- Ctrl+Alt+Del → 암호 입력 후 로그인

### ⚙️ 바탕화면이 안 나올 경우
- PowerShell 창에서 다음 명령 입력:
```cmd
C:\Windows\explorer.exe
```
- 오류 발생 시 → `Server Core` 버전 설치로 판단됨 → Desktop Experience 버전으로 재설치 필요

---

## 5단계: 게스트 확장 설치 (선택)

### 📀 게스트 확장 설치
1. 메뉴 → `장치` → `게스트 확장 CD 이미지 삽입`
2. 가상 머신 내에서 CD 열기 → `VBoxWindowsAdditions.exe` 실행
3. 설치 후 재부팅

---

## ✅ 설치 후 확인

| 항목 | 설명 |
|------|------|
| 바탕화면 표시 | `explorer.exe` 실행 필요 시 수동 실행 |
| 네트워크 설정 | IP, DNS 설정 가능 |
| 서버 관리자 | 자동 실행되며 역할 추가 가능 |
| 업데이트 | 최신 보안 패치 적용 권장 |

---

## 🔁 자주 묻는 질문

### ❓ "Press any key..."가 안 나와요
- 최신 VirtualBox는 자동으로 ISO를 부팅하므로 메시지가 안 보일 수 있습니다. 문제 아닙니다.

### ❓ 설치 화면이 영어로 나와요
- ISO가 영어 버전이면 기본적으로 설치도 영어입니다. 한글은 설치 후 언어팩으로 추가 가능.

### ❓ 로그인했더니 검은 화면만 나와요
- `Server Core` 버전 설치 시 바탕화면이 없습니다 → GUI가 포함된 `(Desktop Experience)` 버전으로 재설치하세요.

---

## 📝 마무리

Windows Server 2022 가상 머신 설치는 반복해 보면 훨씬 쉬워집니다. 만약 바탕화면이 보이지 않거나 explorer가 실행되지 않는다면, 설치한 에디션을 꼭 다시 확인해보세요.

