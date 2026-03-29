# ai-server

이 레포는 AI 뉴스 분석 서비스의 **AI 서버 인프라 구성**을 정리한 저장소입니다.

AI 서버는 **Dify 기반 RAG 시스템**으로 구성되며,  
Vector Database로 **Milvus**를 사용합니다.

---

# Overview

이 프로젝트에서 AI 서버는 다음 역할을 담당합니다.

- Dify 기반 RAG 파이프라인 구성
- Milvus 기반 벡터 저장 및 유사도 검색
- Knowledge Base 기반 문서 검색
- LLM 응답 생성을 위한 AI 서버 환경 구성

---

# System Architecture

전체 서비스 구조는 다음과 같습니다.

```text
Crawler
   ↓
Main Server
   ↓
Dify (AI Server)
   ↓
Milvus (Vector Database)
   ↓
LLM / Retrieval
```

---

# Repository Structure

```text
ai-server
│
├ docs
│   ├ dify-setup.md
│   ├ milvus.md
│   └ architecture.md
│
├ infra
│   └ milvus
│       └ docker-compose.yml
│
├ .gitignore
├ README.md
└ TROUBLESHOOTING.md
```

---

# Components

## Dify

Dify는 AI 서버 플랫폼으로 사용됩니다.

주요 역할

- Knowledge Base 관리
- Retrieval 기반 문서 검색
- LLM Workflow 실행
- RAG 챗봇 구성

## Milvus

Milvus는 Vector Database로 사용됩니다.

주요 역할

- 문서 임베딩 벡터 저장
- 사용자 질문과 유사한 문서 검색
- RAG 검색 파이프라인 지원

---

# Documents

자세한 내용은 아래 문서를 참고합니다.

- `docs/dify-setup.md` : Dify 구축 과정
- `docs/milvus.md` : Milvus 구축 및 Dify 연동

---

# Notes

이 레포에는 Dify 전체 소스코드를 포함하지 않습니다.  
Dify는 공식 GitHub 저장소를 별도로 clone하여 사용합니다.

Milvus는 `infra/milvus/docker-compose.yml`을 통해 구성합니다.
