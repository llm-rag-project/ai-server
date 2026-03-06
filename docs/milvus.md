# Milvus 구축 및 Dify 연동

이 문서는 AI 서비스에서 사용하는 **Vector Database인 Milvus를 Docker Compose로 구축하고 Dify와 연동한 과정**을 정리합니다.

---

# 1. Milvus란 무엇인가

Milvus는 **Vector Database**입니다.

LLM 기반 서비스에서 문서를 임베딩 벡터로 저장하고,  
사용자의 질문과 **유사한 벡터를 빠르게 검색**하기 위해 사용됩니다.

본 프로젝트에서는 다음 구조로 사용됩니다.

```
사용자 질문
      ↓
Embedding 생성
      ↓
Milvus Vector Search
      ↓
관련 문서 Chunk 검색
      ↓
LLM 답변 생성
```

---

# 2. Milvus 배포 방식

Milvus는 단일 컨테이너가 아니라 다음 **3개의 서비스**로 구성됩니다.

```
Milvus Standalone Stack
 ├ etcd
 ├ minio
 └ milvus-standalone
```

각 서비스 역할

| 서비스 | 역할 |
|------|------|
| etcd | metadata 저장 |
| minio | object storage (vector 데이터 저장) |
| milvus-standalone | vector search engine |

---

# 3. Milvus Docker Compose 구성

Milvus는 Docker Compose로 실행됩니다.

실행 방식

```
docker compose up -d
```

실행되면 다음 컨테이너가 생성됩니다.

```
milvus-etcd
milvus-minio
milvus-standalone
```

---

# 4. Milvus Docker Compose 설정

Milvus는 다음 docker-compose.yml을 사용하여 실행됩니다.

```yaml
version: '3.5'

services:
  etcd:
    container_name: milvus-etcd
    image: quay.io/coreos/etcd:v3.5.25
    environment:
      - ETCD_AUTO_COMPACTION_MODE=revision
      - ETCD_AUTO_COMPACTION_RETENTION=1000
      - ETCD_QUOTA_BACKEND_BYTES=4294967296
      - ETCD_SNAPSHOT_COUNT=50000
    volumes:
      - ${DOCKER_VOLUME_DIRECTORY:-.}/volumes/etcd:/etcd
    command: etcd -advertise-client-urls=http://etcd:2379 -listen-client-urls http://0.0.0.0:2379 --data-dir /etcd

  minio:
    container_name: milvus-minio
    image: minio/minio:RELEASE.2024-12-18T13-15-44Z
    environment:
      MINIO_ACCESS_KEY: minioadmin
      MINIO_SECRET_KEY: minioadmin
    volumes:
      - ${DOCKER_VOLUME_DIRECTORY:-.}/volumes/minio:/minio_data
    command: minio server /minio_data --console-address ":9001"

  standalone:
    container_name: milvus-standalone
    image: milvusdb/milvus:v2.6.11
    command: ["milvus", "run", "standalone"]
    security_opt:
      - seccomp:unconfined
    environment:
      ETCD_ENDPOINTS: etcd:2379
      MINIO_ADDRESS: minio:9000
      MQ_TYPE: woodpecker
      common.security.authorizationEnabled: "true"
    volumes:
      - ${DOCKER_VOLUME_DIRECTORY:-.}/volumes/milvus:/var/lib/milvus
    depends_on:
      - "etcd"
      - "minio"
```

---

# 5. Docker Network 구성

Milvus는 Dify와 통신하기 위해 **같은 Docker Network에 연결됩니다.**

예시

```
Milvus Container
      ↓
docker_default network
      ↓
Dify Container
```

같은 Docker network에 있으면 컨테이너 이름으로 통신할 수 있습니다.

예시

```
milvus-standalone:19530
```

---

# 6. Milvus 실행 확인

Milvus 실행 후 다음 명령어로 컨테이너 상태를 확인합니다.

```
docker ps
```

정상 실행 시 다음 컨테이너가 표시됩니다.

```
milvus-etcd
milvus-minio
milvus-standalone
```

---

# 7. Dify와 Milvus 연동

Milvus 실행 후 Dify에서 Vector Database를 설정합니다.

경로

```
Dify Admin
 → Settings
 → Vector Database
```

Milvus 선택 후 다음 정보를 입력합니다.

```
Host: milvus-standalone
Port: 19530
```

같은 Docker network에 있으므로 **컨테이너 이름으로 접근이 가능합니다.**

---

# 8. 데이터 흐름

본 프로젝트에서 Milvus는 다음 흐름으로 사용됩니다.

```
크롤링 기사
      ↓
Main Server
      ↓
Dify Knowledge API
      ↓
Document 저장
      ↓
Chunk 생성
      ↓
Embedding 생성
      ↓
Milvus Vector 저장
```

질문이 들어오면 다음 과정이 실행됩니다.

```
사용자 질문
      ↓
Embedding 생성
      ↓
Milvus Vector Search
      ↓
관련 문서 Chunk Retrieval
      ↓
LLM 답변 생성
```

---

# 9. Milvus 스키마 관리

본 프로젝트에서는 **Milvus 스키마를 직접 설계하지 않습니다.**

Dify Knowledge Base 기능이 다음 작업을 자동으로 수행합니다.

```
Document Chunk 생성
Vector Embedding 생성
Milvus Collection 생성
Vector 저장
Metadata 저장
```

따라서 개발자는 **Milvus Collection을 직접 관리할 필요가 없습니다.**

---

# 10. 정리

Milvus 구축 과정 요약

```
1. Milvus docker-compose 실행
2. Milvus 컨테이너 실행 확인
3. Dify와 같은 Docker Network 연결
4. Dify Vector Database 설정
5. Knowledge Base를 통해 자동 벡터 저장
```

Milvus는 본 프로젝트에서 **RAG 기반 검색 시스템의 Vector Database 역할**을 수행합니다.