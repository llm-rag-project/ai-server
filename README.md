# ai-server

AI 뉴스 분석 서비스의 **AI 서버 인프라 구성**을 정리한 저장소입니다.

AI 서버는 **Dify 기반 RAG 시스템**으로 구성되며, Vector Database로 **Milvus**를 사용합니다.

---

## Overview

이 프로젝트에서 AI 서버는 다음 역할을 담당합니다.

- Dify 기반 RAG 파이프라인 구성
- Milvus 기반 벡터 저장 및 유사도 검색
- Knowledge Base 기반 문서 검색
- LLM 응답 생성을 위한 AI 서버 환경 구성

---

## System Architecture

```text
Crawler
   ↓
Main Server
   ↓
Dify (AI Server)  ←  Workflows (RAG / 요약 / 중요도 점수 / 감성 분석)
   ↓
Milvus (Vector Database)
   ↓
LLM / Retrieval
```

---

## Repository Structure

```text
ai-server
├── docs
│   ├── dify-setup.md                    ← Dify 구축 과정
│   ├── milvus.md                        ← Milvus 구축 및 Dify 연동
│   └── workflow-design
│       ├── rag_chat.md                  ← rag_chat 워크플로우 설계 문서
│       ├── summary_ko.md                ← summary_ko 워크플로우 설계 문서
│       ├── importance_scoring.md        ← importance_scoring 워크플로우 설계 문서
│       └── articles_analysis.md         ← articles_analysis 워크플로우 설계 문서
│
├── infra
│   └── milvus
│       └── docker-compose.yml           ← Milvus Docker Compose 설정
│
├── workflows
│   ├── rag_chat.yml
│   ├── summary_ko.yml
│   ├── importance_scoring.yml
│   ├── articles_analysis.yml
│   └── screenshots
│       ├── rag_chat-whole-workflow.png
│       ├── summary_ko-whole-workflow.png
│       ├── importance_scoring-whole-workflow.png
│       └── articles_analysis-whole-workflow.png
│
├── .gitignore
├── README.md
└── TROUBLESHOOTING.md
```

---

## Components

### Dify

Dify는 AI 서버 플랫폼으로 사용됩니다. 현재 버전은 **v1.13.3**입니다.

주요 역할

- Knowledge Base 관리
- Retrieval 기반 문서 검색
- LLM Workflow 실행
- RAG 챗봇 구성

> 이 레포에는 Dify 전체 소스코드를 포함하지 않습니다.
> Dify는 공식 GitHub 저장소를 별도로 clone하여 사용합니다.

### Milvus

Milvus는 Vector Database로 사용됩니다. `infra/milvus/docker-compose.yml`을 통해 구성합니다.

주요 역할

- 문서 임베딩 벡터 저장
- 사용자 질문과 유사한 문서 검색
- RAG 검색 파이프라인 지원

구성 서비스

| 서비스 | 역할 |
|---|---|
| etcd | 메타데이터 저장 |
| minio | 오브젝트 스토리지 (벡터 데이터 저장) |
| milvus-standalone | 벡터 검색 엔진 |

---

## Workflows

Dify에서 실행되는 워크플로우 4개를 YAML로 관리합니다.
각 워크플로우의 설계 의도와 프롬프트는 `docs/workflow-design/`에 정리되어 있습니다.

### rag_chat

사용자의 질문을 분류하여 경로별로 분기해 뉴스 기반 답변을 생성하는 RAG 챗봇입니다.

```text
시작 (user_id, article_id)
  → IF/ELSE (article_id 있음?)
      → [True]  쿼리재작성 → 지식검색 (score_threshold: 0.5) → LLM → 답변
      → [False] 질문 분류기
                  → [뉴스 질문] 쿼리재작성 → 지식검색 (score_threshold: 0.5) → LLM → 답변
                  → [일반 대화] LLM → 답변
```

| 항목 | 값 |
|---|---|
| 모델 | gemini-2.5-flash |
| Embedding | text-embedding-3-small |
| 대화 이력 | 활성화 (window: 6턴) |

### summary_ko

뉴스 기사 1건을 입력받아 한국어 요약문을 생성합니다.

```text
시작 (article_id, title, content) → 코드(언어 판별 / 길이 전처리) → LLM → 출력
```

| 항목 | 값 |
|---|---|
| 모델 | gemini-2.5-pro |
| temperature | 0.3 |
| 출력 | 한국어 요약문 2~3문장 |

### importance_scoring

기사 목록을 입력받아 각 기사의 중요도 점수(1~100)와 한국어 이유를 반환합니다.

```text
시작 (articles JSON) → LLM-분석 (CoT) → LLM-점수산출 (JSON 전용) → 코드(JSON 파싱) → 출력
```

| 항목 | 값 |
|---|---|
| 모델 | gemini-2.5-pro |
| temperature | 분석 0.3 / 점수산출 0.1 |
| 출력 | `[{article_id, score, reason}]` 배열 |

### articles_analysis

기사 목록을 입력받아 각 기사의 감성(긍정/부정/중립)과 홍보성 여부를 반환합니다.
크롤링 배치 완료 후 메인 서버가 매일 오전 8시에 자동 호출합니다.

```text
시작 (articles JSON)
  → 코드1(입력검증)
  → LLM(감성/분류 분석)
  → 코드2(JSON 파싱)
  → 출력
```

| 항목 | 값 |
|---|---|
| 모델 | gemini-2.5-pro |
| temperature | 0.1 |
| 최대 입력 건수 | 30건 |
| 출력 | `[{article_id, sentiment, is_promotion}]` 배열 |

**출력 필드**

| 필드 | 타입 | 설명 |
|---|---|---|
| `article_id` | number | 기사 ID |
| `sentiment` | string | `긍정` / `부정` / `중립` / `분석실패` |
| `is_promotion` | boolean \| null | 홍보성 여부 (`null`이면 분석 실패) |

---

## Documents

| 문서 | 내용 |
|---|---|
| `docs/dify-setup.md` | Dify 구축 과정 (버전 체크아웃, Docker Compose 실행) |
| `docs/milvus.md` | Milvus 구축 및 Dify 연동 |
| `docs/workflow-design/rag_chat.md` | rag_chat 워크플로우 설계 문서 |
| `docs/workflow-design/summary_ko.md` | summary_ko 워크플로우 설계 문서 |
| `docs/workflow-design/importance_scoring.md` | importance_scoring 워크플로우 설계 문서 |
| `docs/workflow-design/articles_analysis.md` | articles_analysis 워크플로우 설계 문서 |
| `TROUBLESHOOTING.md` | 주요 장애 대응 가이드 |
