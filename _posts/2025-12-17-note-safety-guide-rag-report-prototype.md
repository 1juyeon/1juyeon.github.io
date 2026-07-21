---
title: "[Note] 안전 가이드 문서를 검색해 행동 권고와 리포트를 만드는 RAG 실험"
date: 2025-12-17 20:00:00 +0900
categories: [note]
tags: [rag, chromadb, python, ollama, document-search]
layout: single
---

CCTV 이상행동 이벤트가 들어왔을 때 안전 가이드 문서에서 관련 절차를 찾고, 상황별 행동 권고와 리포트 초안을 만드는 RAG 프로토타입을 구현했다.

단순히 LLM에 “낙상 사고 대응 방법을 알려줘”라고 묻는 것이 아니라, 로컬에 준비한 PDF/DOCX/Markdown 문서를 근거로 검색하고 그 결과를 생성 입력에 포함하는 것이 목표였다.

## 전체 흐름

```text
PDF/DOCX/MD 문서
→ 텍스트 추출
→ 절차 단위 청킹
→ 이벤트/시설/대응 단계 메타데이터 태깅
→ ChromaDB 인덱싱
→ 상황 입력
→ 메타데이터 필터 + 벡터 검색
→ 행동 권고 생성
→ 6가지 형식의 리포트 생성
```

프로토타입은 Python CLI로 만들었고, LLM 없이도 검증할 수 있는 규칙 기반 생성기와 로컬 Ollama 연동을 모두 지원하도록 나눴다.

## 문서 형식별 로더를 분리했다

문서 폴더를 재귀 탐색해 확장자에 맞는 로더를 사용했다.

```python
def load_document(path: Path) -> str:
    suffix = path.suffix.lower()
    if suffix == ".pdf":
        return load_pdf(path)
    if suffix == ".docx":
        return load_docx(path)
    if suffix == ".md":
        return path.read_text(encoding="utf-8")
    raise ValueError(f"unsupported document: {suffix}")
```

PDF는 페이지 번호를 같이 보존하고, DOCX는 문단 텍스트를 추출했다. 스캔 이미지 PDF는 일반 텍스트 추출로 읽히지 않으므로 OCR이 별도로 필요하다는 제한도 두었다.

## 고정 길이만으로 자르지 않았다

문서를 일정 글자 수로만 자르면 “신고 → 응급조치 → 사후 기록” 같은 하나의 절차가 중간에서 끊길 수 있다.

그래서 제목과 문장 경계를 우선하고, 번호가 붙은 절차는 가능한 한 같은 청크에 남겼다. 각 청크에는 검색에 사용할 메타데이터를 붙였다.

```python
class KBChunk(BaseModel):
    text: str
    source_file: str
    page_number: int | None
    event_types: list[EventType]
    facility_types: list[FacilityType]
    phases: list[ScenarioPhase]
    risk_levels: list[RiskLevel]
```

이 구조로 검색 결과가 어느 파일과 페이지에서 왔는지 생성 결과와 함께 남길 수 있었다.

## 벡터 유사도만 사용하지 않은 이유

“사람이 쓰러졌다”는 문장은 요양시설, 어린이집, 공공장소 문서에서 모두 비슷하게 검색될 수 있다. 하지만 시설과 대응 단계가 다르면 필요한 절차도 다르다.

그래서 먼저 이벤트 종류와 시설 종류를 메타데이터 필터로 제한하고, 그 안에서 의미가 가까운 청크를 벡터 검색했다.

```python
filters = {
    "event_types": [situation.event_type],
    "facility_types": [situation.facility_type],
    "phases": [
        ScenarioPhase.IMMEDIATE_RESPONSE,
        ScenarioPhase.REPORTING,
    ],
}

chunks = vector_store.search(
    query=situation.description,
    filters=filters,
    top_k=5,
)
```

필터가 너무 엄격해 결과가 없으면 대응 단계 같은 일부 조건을 완화해 한 번 더 검색했다. 처음부터 필터를 모두 없애지는 않았다. 검색 결과가 있다는 이유로 다른 시설의 절차를 가져오는 것을 줄이기 위해서다.

## 생성 백엔드를 인터페이스로 분리했다

처음부터 로컬 LLM에만 의존하면 모델 서버가 없을 때 검색과 리포트 로직을 테스트하기 어렵다.

```python
class LLMClient(Protocol):
    def generate(self, prompt: str) -> str:
        ...

class RuleBasedLLMClient:
    ...

class OllamaLLMClient:
    ...
```

규칙 기반 구현은 검색 결과와 상황 정보를 정해진 템플릿으로 조합했다. Ollama 구현은 로컬 HTTP API를 호출하되, 연결 실패나 timeout이 나면 기본 리포트로 돌아가도록 했다.

이 분리 덕분에 다음을 따로 검증할 수 있었다.

- 문서 로딩과 청킹
- 메타데이터 필터
- 벡터 검색 결과
- 생성 프롬프트
- LLM 응답 파싱
- fallback 리포트

## 6가지 출력 형식을 하나의 결과로 묶었다

같은 사건이라도 전달 대상에 따라 필요한 길이와 내용이 달라진다.

```python
class Report6Types(BaseModel):
    immediate_report: str
    daily_report: str
    weekly_report: str
    monthly_report: str
    sms_text: str
    kakao_text: str
    reference_sources: list[str]
```

- 즉시 보고: 현재 위험과 즉시 조치
- 일간 보고: 사고 개요, 원인, 조치, 후속 관리
- 주간/월간 보고: 동일 유형의 경향과 개선 항목
- SMS/메신저: 길이를 줄인 현장 전달용 문장

LLM 응답은 JSON으로 받도록 요청하고, 필드 누락이나 JSON 파싱 실패 시 fallback 생성기로 전환했다.

## 테스트에서 확인한 것

리포트 생성 테스트에서는 최소한 다음 계약을 확인했다.

```text
모든 출력 필드가 비어 있지 않은가
SMS처럼 짧은 형식이 생성되는가
검색 출처가 결과에 남는가
LLM 호출 실패 시 fallback이 동작하는가
```

생성 문장이 자연스럽다는 것만 테스트한 것은 아니다. 파이프라인의 각 단계가 실패해도 결과 구조가 깨지지 않는지를 우선 확인했다.

## 프로토타입의 한계

- 키워드 기반 자동 태깅은 문맥을 완전히 이해하지 못한다.
- 표와 이미지 중심 PDF는 텍스트 추출 품질이 낮을 수 있다.
- 검색 결과가 맞아도 생성 문장이 원문보다 과장될 수 있다.
- 주간/월간 보고서는 실제 누적 통계가 아니라 단일 이벤트 기반 초안이다.
- 안전 대응 문서는 최신 지침과 담당자 검토가 필요하다.

따라서 이 코드는 현장 판단을 대신하는 시스템이 아니라, 문서 검색과 보고서 초안 자동화 가능성을 검증한 실험으로 두었다.

## 정리

이 프로젝트에서 구현한 것은 LLM 호출 한 번이 아니었다.

- PDF/DOCX/Markdown 로더와 문서 청킹
- 이벤트, 시설, 대응 단계 메타데이터 설계
- ChromaDB 필터와 벡터 검색
- 검색 결과가 없을 때의 단계적 필터 완화
- 규칙 기반/로컬 LLM 백엔드 분리
- 6종 리포트 구조와 fallback
- 출처와 페이지 정보 보존

RAG에서 중요한 것은 모델보다 검색할 문서를 어떤 단위로 나누고, 어떤 조건으로 좁히며, 실패했을 때 어디까지 되돌아갈지 정하는 일이라는 것을 확인한 프로젝트였다.
