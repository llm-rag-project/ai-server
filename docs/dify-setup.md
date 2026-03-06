# Dify 구축 과정

이 문서는 개발 서버에서 **Dify를 로컬 환경에 구축한 과정**을 정리한 문서입니다.

---

# 1. Dify 다운로드

Dify 공식 GitHub 저장소를 클론합니다.

```bash
git clone https://github.com/langgenius/dify.git
```

---

# 2. Dify 디렉토리 이동

클론한 Dify 저장소의 docker 폴더로 이동합니다.

```bash
cd dify/docker
```

---

# 3. 환경 변수 파일 준비

`.env.example` 파일을 `.env`로 복사합니다.

```bash
cp .env.example .env
```

이후 필요에 따라 `.env` 파일의 설정 값을 확인하거나 수정합니다.

예시

```env
EXPOSE_NGINX_PORT=80
EXPOSE_NGINX_SSL_PORT=443
```

이 설정은 Dify가 외부에 노출되는 포트를 의미합니다.

---

# 4. Docker Compose 실행

다음 명령어로 Dify 서비스를 실행합니다.

```bash
docker compose up -d
```

이 명령을 실행하면 Dify 관련 컨테이너들이 백그라운드에서 실행됩니다.

예시 구성 요소

- nginx
- api
- worker
- web
- postgres
- redis

---

# 5. 실행 확인

컨테이너가 정상적으로 실행되었는지 확인합니다.

```bash
docker ps
```

정상적으로 실행되었다면 Dify 관련 컨테이너들이 표시됩니다.

---

# 6. Dify 접속

브라우저에서 다음 주소로 접속합니다.

```text
http://localhost
```

또는 개발 서버 환경에 따라

```text
http://개발서버IP
```

로 접속할 수 있습니다.

`.env` 파일에서 포트를 변경했다면 해당 포트 번호로 접속해야 합니다.

예를 들어

```env
EXPOSE_NGINX_PORT=8001
```

로 설정했다면 접속 주소는 다음과 같습니다.

```text
http://localhost:8001
```

---

# 7. 정리

Dify 구축 과정 요약

```text
1. Dify GitHub 저장소 clone
2. dify/docker 디렉토리 이동
3. .env.example → .env 복사
4. docker compose up -d 실행
5. docker ps로 실행 상태 확인
6. 브라우저에서 localhost 접속
```

이 과정을 통해 개발 서버에 **Self-hosted Dify 환경**을 구축할 수 있습니다.