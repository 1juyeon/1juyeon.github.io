---
title: "Work Log"
layout: single
permalink: /work-log/
author_profile: true
---

## 문제 해결과 상태 일관성

- [SOAP 응답은 실패했지만 카메라 비밀번호는 바뀐 문제]({% post_url 2026-06-06-error-onvif-password-changed-soap-result-mismatch %})
- [카메라 비밀번호 변경 API가 curl (94)로 실패한 문제]({% post_url 2025-06-25-error-digest-인증-실패-원인-분석-및-해결-흐름 %})
- [세션 값을 복사했을 때 다른 계정으로 접근되던 문제]({% post_url 2024-05-22-error-session-cookie-copy-reuse %})
- [Windows 11 24H2에서 원격 계정 비밀번호 변경이 거부된 문제]({% post_url 2025-06-20-error-NetUserChangePassword-ERROR-ACCESS-DENIED %})
- [네이티브 연동에서 긴 문자열 뒤에 쓰레기값이 붙던 문제]({% post_url 2025-04-08-error-native-string-buffer-garbage %})
- [스케줄이 1분 일찍 실행되던 시간 정밀도 문제]({% post_url 2024-11-04-error-schedule-next-run-time-precision %})
- [DB 복원 중 서버 재시작과 Ajax 응답 타이밍을 정리한 기록]({% post_url 2025-05-27-error-ajax-async-option %})

## 보안과 계정 관리

- [장기 미접속 계정 비활성화 정책을 다듬으며 확인한 것]({% post_url 2024-01-08-note-inactive-account-policy-admin-exception %})
- [비밀번호 규칙 관리와 검증 전용 기능을 만들며 정리한 것]({% post_url 2024-01-19-note-password-policy-validation-screen %})
- [무결성 검증 기능과 이력 화면을 추가하며 나눈 역할]({% post_url 2024-03-08-note-integrity-check-history-ui %})
- [로그인 실패 잠금 계정이 자동 로그인에서 다른 상태로 바뀌던 문제]({% post_url 2024-07-12-error-login-lock-autologin-state %})
- [비밀번호 랜덤 생성이 재섞기 과정에서 무한 반복되던 문제]({% post_url 2024-07-17-error-password-generator-reshuffle-loop %})
- [랜덤 비밀번호 생성에 DRBG를 적용하며 확인한 것]({% post_url 2025-04-07-note-drbg-password-generation %})
- [네이티브 DRBG 초기화 오류를 thread-local로 정리한 기록]({% post_url 2025-05-21-error-native-drbg-thread-local-init %})
- [IP 접근 제어 설정 화면을 만들며 정리한 검증 흐름]({% post_url 2025-04-22-note-ip-filtering-admin-ui %})
- [런타임 마스터 비밀번호 기반 KEK/DEK 구조로 바꾸며 정리한 것]({% post_url 2026-03-30-note-kek-dek-runtime-master-password %})
- [기존 암호문을 새 복호화 로직과 같이 살리며 배운 점]({% post_url 2026-04-06-note-legacy-ciphertext-migration %})
- [배치 실행 로그에서 비밀번호가 찍히지 않게 하는 방법]({% post_url 2026-06-22-note-log-masking-defense-in-depth %})

## 이력과 운영 기능

- [실패한 운영 작업도 감사 이력에 남기도록 고친 기록]({% post_url 2024-07-11-error-password-change-failure-history %})
- [운영 이력 데이터가 많아졌을 때 조회 성능을 정리한 기록]({% post_url 2024-11-01-note-password-history-retention-performance %})
- [백업 경로 설정에서 불필요한 드라이브 의존성을 줄인 기록]({% post_url 2025-03-27-note-backup-path-optional-drive %})
- [SSH 기반 서버 계정 변경 기능을 붙이며 나눈 처리 기준]({% post_url 2025-11-11-note-linux-server-password-change %})
- [민감 값 보기/숨기기와 복사 버튼을 넣으며 신경 쓴 것]({% post_url 2025-11-14-note-password-visibility-copy-ui %})
- [스케줄 활성화 토글에서 옵션 상태가 초기화되던 문제]({% post_url 2025-11-17-error-schedule-option-reset %})
- [운영 설정 변경을 DB와 백업까지 동기화한 기록]({% post_url 2026-06-16-note-db-password-change-consistency %})
- [민감 설정 변경 이력 화면을 정리하며 결정한 것]({% post_url 2026-06-17-note-password-change-history-schema %})
- [1분 주기 스케줄이 겹쳐 실행되는지 코드로 확인한 기록]({% post_url 2026-06-11-note-schedule-concurrency %})

## 외부 시스템과 네트워크 장비 연동

- [외부 데이터의 줄바꿈 때문에 목록 팝업이 깨지던 문제]({% post_url 2024-08-12-error-device-description-newline-popup %})
- [외부 연동 TLS 전환에서 인터페이스와 설정을 같이 맞춘 기록]({% post_url 2025-04-04-note-external-integration-tls-config %})
- [SSPI를 끈 Windows용 libcurl을 직접 빌드한 기록]({% post_url 2025-06-30-note-curl-openssl-직접-빌드-정리 %})
- [카메라 API 문제를 풀기 위해 정리한 HTTP Digest 인증 흐름]({% post_url 2026-07-21-note-http-digest-camera-api-communication %})
- [HTTP 200인데 gSOAP SOAP_NO_TAG가 발생한 응답 분석]({% post_url 2026-07-21-error-http-200-gsoap-soap-no-tag %})
- [std::string::find 조건을 반대로 써 장비 잠금으로 오판한 문제]({% post_url 2026-05-20-error-string-find-device-lock-condition %})
- [JNI 인자 하나가 빠져 URL과 계정 값이 한 칸씩 밀린 문제]({% post_url 2026-06-19-note-jni-argument-flow %})
- [외부 동기화에서 기존 값이 덮어써지는 문제]({% post_url 2026-06-08-error-device-import-overwrite-guard %})
- [네이티브 결과 코드 1010을 Java enum이 해석하지 못한 문제]({% post_url 2026-06-04-error-missing-error-code-mapping %})
- [간헐적인 외부 SDK 로그인 실패에서 가설을 배제한 과정]({% post_url 2026-06-29-error-intermittent-external-sdk-login %})
- [SSH 장비 제어를 JSON 명령 템플릿으로 확장한 기록]({% post_url 2026-07-02-note-network-device-json-template %})
- [장비용 JSON 템플릿 로드 실패가 통신 오류로 보인 문제]({% post_url 2026-07-08-error-json-template-loading-null %})
- [네트워크 진단이 ARP 캐시 때문에 실패처럼 보일 때]({% post_url 2026-07-09-error-device-ping-arp-retry %})

## 업그레이드와 배포 안정성

- [OpenSSL 업그레이드 후 장비의 TLS 종료를 오류로 처리한 문제]({% post_url 2026-05-20-error-openssl-upgrade-unexpected-eof %})
- [오래된 Spring 웹 프로젝트를 Jakarta 기준으로 옮기며 확인한 범위]({% post_url 2026-05-20-note-javax-jakarta-upgrade-decision %})
- [Windows 서비스가 정지된 직후 다시 시작되지 않은 문제]({% post_url 2026-06-18-error-windows-service-restart-race %})
- [오래된 라이브러리 업그레이드를 단계별로 나눠 진행한 기록]({% post_url 2026-07-10-note-library-upgrade-staging %})
- [DB 엔진 업그레이드에서 백업 스크립트까지 같이 본 이유]({% post_url 2026-07-14-note-database-upgrade-backup-script %})

## 개인 도구와 실험

- [Qt/C++로 MPEG-TS 암호화 상태 진단 도구를 만든 기록]({% post_url 2025-03-11-note-qt-mpeg-ts-scrambling-analyzer %})
- [Streamlit AI 트러블슈팅 앱에서 벡터 검색 방식을 바꾼 기록]({% post_url 2025-09-19-note-streamlit-vector-search-faiss %})
- [안전 가이드 문서를 검색해 행동 권고와 리포트를 만드는 RAG 실험]({% post_url 2025-12-17-note-safety-guide-rag-report-prototype %})
- [ONVIF 카메라 비밀번호 변경 가능성을 미리 진단하는 도구를 만든 기록]({% post_url 2026-05-14-note-onvif-camera-compatibility-probe %})
- [암호화된 설정값 복구 도구를 Go 단일 exe로 만들며 겪은 GUI 멈춤]({% post_url 2026-06-23-error-decrypt-tool-gui-freeze %})
- [로컬 AI 세션 아카이버에 Codex 로그와 이미지 보존을 추가한 기록]({% post_url 2026-07-16-note-local-ai-session-archive-codex-images %})
- [ChatGPT 데이터 내보내기를 세션 아카이브에 연결한 기록]({% post_url 2026-07-18-note-chatgpt-export-session-archive %})
