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
v1.13.1로 업그레이드 후 `.env`에 `REDIS_MAX_CONNECTIONS=10` 추가

```bash
cd dify
git fetch --tags
git checkout 1.13.1
cd docker
docker compose down
docker compose up -d
```

`.env`에 아래 항목 추가:

```env
REDIS_MAX_CONNECTIONS=10
```
