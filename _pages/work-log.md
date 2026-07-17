---
title: "Work Log"
layout: single
permalink: /work-log/
author_profile: true
---

# Work Log

회사 내부 정보와 제품명은 제외하고, 실제 업무 중 다룬 구현/장애 대응/운영 개선 내용을 공개 가능한 수준으로 정리한 글입니다.

## 보안과 인증 대응

- [장기 미접속 계정 비활성화 정책을 다듬으며 확인한 것]({% post_url 2024-01-08-note-inactive-account-policy-admin-exception %})
- [비밀번호 규칙 관리와 검증 전용 기능을 만들며 정리한 것]({% post_url 2024-01-19-note-password-policy-validation-screen %})
- [무결성 검증 기능과 이력 화면을 추가하며 나눈 역할]({% post_url 2024-03-08-note-integrity-check-history-ui %})
- [세션 값을 복사했을 때 다른 계정으로 접근되던 문제]({% post_url 2024-05-22-error-session-cookie-copy-reuse %})
- [로그인 실패 잠금 계정이 자동 로그인에서 다른 상태로 바뀌던 문제]({% post_url 2024-07-12-error-login-lock-autologin-state %})
- [랜덤 비밀번호 생성에 DRBG를 적용하며 확인한 것]({% post_url 2025-04-07-note-drbg-password-generation %})
- [IP 접근 제어 설정 화면을 만들며 정리한 검증 흐름]({% post_url 2025-04-22-note-ip-filtering-admin-ui %})
- [런타임 마스터 비밀번호 기반 KEK/DEK 구조로 바꾸며 정리한 것]({% post_url 2026-03-30-note-kek-dek-runtime-master-password %})
- [설정 암호화 알고리즘을 확인하고 복구 도구까지 맞춘 기록]({% post_url 2026-06-18-note-jasypt-enc-runtime-key %})

## 계정과 비밀번호 운영 기능

- [비밀번호 변경 실패 이력이 남지 않던 문제]({% post_url 2024-07-11-error-password-change-failure-history %})
- [비밀번호 랜덤 생성이 재섞기 과정에서 무한 반복되던 문제]({% post_url 2024-07-17-error-password-generator-reshuffle-loop %})
- [리눅스 서버 비밀번호 변경 기능을 붙이며 나눈 처리 기준]({% post_url 2025-11-11-note-linux-server-password-change %})
- [비밀번호 보기/숨기기와 복사 버튼을 넣으며 신경 쓴 것]({% post_url 2025-11-14-note-password-visibility-copy-ui %})
- [스케줄 활성화 토글에서 옵션 상태가 초기화되던 문제]({% post_url 2025-11-17-error-schedule-option-reset %})
- [DB 비밀번호 변경 화면을 만들며 같이 동기화한 것들]({% post_url 2026-06-16-note-db-password-change-consistency %})

## 외부 연동과 장비 자동화

- [장치 설명의 줄바꿈 때문에 목록 팝업이 깨지던 문제]({% post_url 2024-08-12-error-device-description-newline-popup %})
- [외부 연동 TLS 전환에서 인터페이스와 설정을 같이 맞춘 기록]({% post_url 2025-04-04-note-external-integration-tls-config %})
- [장치 정보를 다시 불러올 때 기존 값이 덮어써지는 문제]({% post_url 2026-06-08-error-device-import-overwrite-guard %})
- [네트워크 장비 비밀번호 변경을 JSON 템플릿으로 확장한 기록]({% post_url 2026-07-02-note-network-device-json-template %})
- [장비 ping 테스트가 ARP 캐시 때문에 실패처럼 보일 때]({% post_url 2026-07-09-error-device-ping-arp-retry %})

## 업그레이드와 운영 안정성

- [비밀번호 변경 이력이 많아졌을 때 조회 성능을 정리한 기록]({% post_url 2024-11-01-note-password-history-retention-performance %})
- [스케줄이 1분 일찍 실행되던 시간 정밀도 문제]({% post_url 2024-11-04-error-schedule-next-run-time-precision %})
- [백업 경로 설정에서 불필요한 드라이브 의존성을 줄인 기록]({% post_url 2025-03-27-note-backup-path-optional-drive %})
- [오래된 라이브러리 업그레이드를 단계별로 나눠 진행한 기록]({% post_url 2026-07-10-note-library-upgrade-staging %})
- [DB 엔진 업그레이드에서 백업 스크립트까지 같이 본 이유]({% post_url 2026-07-14-note-database-upgrade-backup-script %})
- [Spring과 Tomcat 업그레이드에서 javax와 jakarta를 먼저 봐야 하는 이유]({% post_url 2026-05-20-note-javax-jakarta-upgrade-decision %})
