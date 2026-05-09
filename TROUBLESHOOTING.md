# Troubleshooting

## 1. 지식검색 result가 항상 []로 나올 때

**증상**
- Dify Chatflow에서 지식검색 노드 실행 결과가 `"result": []`
- LLM이 "관련 정보를 찾을 수 없습니다" 응답

**원인 체크리스트**
1. Knowledge 인덱스 방법이 "경제적"으로 설정된 경우 → metadata filtering 및 벡터 검색 불가
2. 지식검색 노드의 query 변수가 잘못 연결된 경우
3. Milvus가 꺼져있는 경우 (가장 흔한 원인)

**해결**
1. Knowledge > 설정 > 인덱스 방법을 "고품질"로 변경, 검색 설정을 "벡터 검색"으로 변경
2. 지식검색 노드 > 질의 텍스트 변수를 `시작.query`로 정확히 연결
3. Milvus 상태 확인 및 재시작 (아래 2번 참고)

---

## 2. Milvus 연결 오류

**증상**
```
MilvusException: code=2, message=Fail connecting to server on milvus-standalone:19530
```

**원인**
Milvus standalone 컨테이너가 꺼져있거나 etcd에 연결하지 못해 죽은 상태

**해결**
```bash
cd C:\Users\lab\milvus
docker compose ps        # 상태 확인
docker compose down
docker compose up -d
docker compose ps        # 2분 후 세 컨테이너 모두 healthy 확인
```

healthy 확인 후 Dify Knowledge에서 재인덱싱 시도

---

## 3. milvus-standalone만 계속 죽을 때

**증상**
- `docker compose ps` 결과에 milvus-standalone이 없음
- etcd, minio만 healthy 상태

**원인**
docker-compose.yml에서 etcd, minio 서비스에 `milvus` 네트워크가 설정되지 않아
standalone이 `etcd:2379`로 DNS 조회 실패

**에러 로그**
```
dial tcp: lookup etcd on 127.0.0.11:53: no such host
```

**해결**
`docker-compose.yml`의 etcd, minio 서비스에 networks 추가
```yaml
etcd:
  ...
  networks:
    - milvus

minio:
  ...
  networks:
    - milvus
```

---

## 4. docker compose up 시 네트워크 충돌 오류

**증상**
```
network milvus was found but has incorrect label com.docker.compose.network
```

**원인**
이전에 비정상 종료되면서 milvus 네트워크가 깨진 채로 남아있는 상태

**해결**
```bash
docker compose down
docker network prune -f
docker compose up -d
```

**예방**
컴퓨터 끄기 전에 항상 `docker compose down` 실행

---

## 5. Knowledge 문서 인덱싱 오류

**증상**
- 문서 상태가 "오류" (빨간불)
- "1 문서 인덱스에 실패했습니다" 메시지

**원인**
Milvus가 꺼진 상태에서 인덱싱 시도

**해결**
1. Milvus 정상 기동 확인 (위 2번 참고)
2. Dify > Knowledge > 문서 > "새시도" 버튼 클릭
3. 새시도 실패 시 문서 삭제 후 재업로드

---

## 6. Dify metadata filter variable 버그

**증상**
- 지식검색 노드에서 metadata filter를 variable로 설정해도 `result: []`로 반환
- metadata filter를 고정값으로 설정하면 정상 작동

**원인**
Dify v1.13.0 버그

**해결**
v1.13.3로 업그레이드 후 `.env`에 `REDIS_MAX_CONNECTIONS=10` 추가

```bash
cd dify
git fetch --tags
git checkout 1.13.3
cd docker
docker compose down
docker compose up -d
```

`.env`에 아래 항목 추가:

```env
REDIS_MAX_CONNECTIONS=10
```

## 7. 지식검색 노드 무한 대기 문제 (Milvus 미실행)

**증상**

- rag_chat 워크플로우에서 지식검색 노드에서 멈춤
- 트레이싱 탭에서 지식검색 노드가 `...` 상태로 응답 없음
- Dify 컨테이너는 모두 `Up` 상태이고 Redis, Weaviate도 정상
- Docker 재시작 또는 서버 재부팅 후 발생하는 경우가 많음

**원인**

Milvus가 Dify와 별도 docker-compose로 구성되어 있고 `restart: always` 옵션이 없어서, Docker 재시작 또는 서버 재부팅 시 자동으로 올라오지 않는다.

Dify의 지식검색 노드는 Milvus에 벡터 검색 요청을 보내는데, Milvus가 내려가 있으면 응답을 받지 못하고 무한 대기 상태가 된다.

**진단 방법**

```bash
cd C:\Users\lab\milvus
docker compose ps
```

아래처럼 컨테이너 목록이 비어있으면 Milvus가 내려간 상태다.

```
NAME      IMAGE     COMMAND   SERVICE   CREATED   STATUS    PORTS
(없음)
```

정상 상태라면 `milvus-etcd`, `milvus-minio`, `milvus-standalone` 3개가 `Up` 상태로 보여야 한다.

**즉시 해결 방법**

```bash
cd C:\Users\lab\milvus
docker compose up -d
```

Milvus가 완전히 올라오는 데 약 1~2분 소요된다. `milvus-standalone`의 healthcheck가 통과될 때까지 기다린 후 Dify에서 다시 테스트한다.

---

**영구 해결 방법 (restart: always 설정)**

`C:\Users\lab\milvus\docker-compose.yml`에서 각 서비스에 `restart: always`를 추가한다.

```yaml
services:
  etcd:
    restart: always
    ...
  minio:
    restart: always
    ...
  standalone:
    restart: always
    ...
```

저장 후 재적용:

```bash
cd C:\Users\lab\milvus
docker compose up -d
```

이후 Docker 재시작이나 서버 재부팅 시에도 Milvus가 자동으로 올라온다.

---

**참고**

- Dify의 `docker-compose.yml`에는 이미 `restart: always`가 적용되어 있어서 Dify 컨테이너는 자동 재시작된다. Milvus만 별도로 설정이 필요하다.
- Milvus는 etcd → minio → standalone 순서로 의존성이 있어서 올라오는 데 시간이 걸린다. `docker compose up -d` 직후 바로 테스트하면 아직 준비가 안 된 상태일 수 있다.
