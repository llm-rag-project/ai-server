# importance_scoring 워크플로우 설계

> 각 기사의 중요도 점수와 이유를 배열로 반환하는 워크플로우
> 

---

## 개요

| 항목 | 내용 |
| --- | --- |
| 워크플로우 이름 | importance_scoring |
| 역할 | 기사 목록 → 중요도 점수(1~100) + 한국어 이유 배열 반환 |
| 트리거 방식 | 메인 서버 API 호출 |
| 모델 | gemini-2.5-pro |

---

## 구조

```
[변경 전]
시작 → LLM → 코드(JSON 파싱) → 출력

[변경 후]
시작 → LLM-분석(CoT) → LLM-점수산출(JSON 전용) → 코드(JSON 파싱) → 출력
```

---

## 시작 노드 입력변수

| 변수명 | 타입 | 필수 여부 | 설명 |
| --- | --- | --- | --- |
| user_id | string | 필수 | 사용자 식별자 |
| articles | string | 필수 | 기사를 담은 JSON 문자열 |

> **articles 설계 의도**
현재는 기사 1건씩 버튼 트리거 방식으로 호출되며, 1건짜리 배열로 전달된다. 향후 다건 처리 확장을 고려해 배열 구조로 설계되었다.
> 
> 
> Dify 시작 노드가 Array[Object]를 지원하지 않아 json.dumps()로 직렬화된 JSON 문자열로 전달한다.
> 

---

## LLM-분석 노드 (CoT)

### 역할

각 기사의 영향 범위, 시급성, 파급력을 자유 텍스트로 분석한다. JSON 출력 없음.

### 왜 분리했나

CoT(Chain-of-Thought) 분석과 JSON 출력을 한 LLM 노드에 동시에 요구하면 구조적 충돌이 발생한다. Gemini 계열 모델은 "내부적으로만 분석하고 JSON만 출력해"라는 지시를 잘 따르지 않고, 분석 내용을 JSON 앞뒤에 자연어로 섞어 출력하는 경향이 있다. 이렇게 되면 코드 노드의 JSON 파싱이 실패한다.

역할을 물리적으로 분리해서:

- LLM-분석이 자유롭게 분석 텍스트를 생성하고
- LLM-점수산출은 오직 JSON만 출력하도록 책임을 나눈다.

이렇게 하면 CoT의 품질 향상 효과는 살리면서 JSON 파싱 안정성도 확보된다.

### 설정값

| 항목 | 값 |
| --- | --- |
| 모델 | gemini-2.5-pro |
| temperature | 0.3 |

### 시스템 프롬프트

```
당신은 뉴스 중요도 분석 전문가입니다.
입력된 기사 목록에 대해 각 기사별로 아래 3가지 항목을 분석하세요.

[분석 항목]
1. 영향 범위: 이 기사가 영향을 미치는 대상의 규모 (개인 / 지역 / 국가 / 글로벌)
2. 시급성: 현재 진행 중인 사안인지, 즉각적 대응이 필요한지 여부
3. 파급력: 경제·정치·사회적으로 연쇄 반응이나 실질적 결과(정책 변화, 시장 영향, 인명 피해 등)가 수반되는지

[출력 형식]
각 기사를 아래 형식으로 분석하세요. JSON 없이 텍스트로만 출력합니다.

article_id: {id}
- 영향 범위: ...
- 시급성: ...
- 파급력: ...
- 종합 판단: ... (1~2문장으로 이 기사의 중요도를 종합적으로 평가)

article_id: {id}
- 영향 범위: ...
...
```

### 유저 프롬프트

```
[기사 목록 JSON]
{{articles}}
```

---

## LLM-점수산출 노드 (JSON 전용)

### 역할

LLM-분석 노드의 분석 결과를 참고해 점수를 산정하고 JSON만 반환한다.

### 설정값

| 항목 | 값 |
| --- | --- |
| 모델 | gemini-2.5-pro |
| temperature | 0.1 |

**temperature를 0.1로 낮춘 이유:**
점수 산정은 일관성이 핵심이다. 같은 기사를 두 번 넣었을 때 점수가 크게 달라지면 서비스 신뢰도가 떨어진다. JSON 구조 출력은 창의성이 전혀 필요 없으므로 최저값에 가깝게 설정했다.

### 입력변수

| 변수명 | 연결 |
| --- | --- |
| articles | 시작 노드 → articles |
| llm_analysis_output | LLM-분석 노드 → 출력 텍스트 |

### 시스템 프롬프트

```
당신은 뉴스 기사 중요도 점수 산정 전문가입니다.
제공된 [기사 분석 결과]를 참고하여 각 기사의 중요도를 1~100점으로 평가하고 JSON만 반환합니다.

---

[카테고리별 점수 기준]

■ 80~100점 (매우 중요)
- 경제: 기준금리 결정, 글로벌 금융위기, 대형 기업 파산·합병, 환율 급변
- 정치: 선거 결과, 정권 교체, 주요 입법 통과, 국가 간 외교 분쟁 격화
- 사회: 대규모 재난·사고(다수 사망), 전국 단위 파업·시위
- 국제: 전쟁·무력 충돌 발생, 국제 제재, 글로벌 공급망 붕괴

■ 50~79점 (중요)
- 경제: 주요 기업 실적 발표, 산업 규제 변화, 부동산 정책 발표
- 정치: 장관급 인사, 여야 협상 타결, 지방선거 결과
- 사회: 사회적 논란이 되는 사건, 수도권 영향을 미치는 이슈
- 국제: 주요국 정상회담, 무역 협정 체결, 지역 분쟁 발생

■ 20~49점 (보통)
- 경제: 중소기업 동향, 소비 트렌드, 특정 업계 단신
- 정치: 당내 논의, 지방 정책 변화, 위원회 구성
- 사회: 지역 사건·사고, 문화·스포츠 이슈
- 국제: 해외 지역 뉴스, 국제기구 일반 발표

■ 1~19점 (낮음)
- 홍보성 기사, 특정 개인·소집단에만 해당하는 내용
- 반복 보도, 정보 가치가 낮은 단신

---

[점수 분산 규칙]
- 기사가 2건 이상이면, 가장 높은 점수와 가장 낮은 점수의 차이가 반드시 30점 이상이어야 합니다.
- 모든 기사에 유사한 점수(±10점 이내)를 부여하지 마세요.
- 기사가 1건일 경우 이 규칙은 적용하지 않습니다.

---

[출력 규칙]
- 반드시 아래 JSON 구조만 출력하세요.
- 마크다운, 코드블록(```), 설명 문장, 줄바꿈 전후 텍스트 일절 금지.
- article_id는 입력값 그대로 유지하세요.
- score는 1~100 사이의 정수입니다.
- reason은 한국어 1~2문장으로, 영향 범위·시급성·파급력 중 점수에 결정적 영향을 준 요소를 구체적으로 포함하세요.
- title, content 등 다른 필드는 절대 포함하지 마세요.

출력 형식:
{"items": [{"article_id": 101, "score": 85, "reason": "한국어 이유"}, {"article_id": 102, "score": 42, "reason": "한국어 이유"}]}
```

### 유저 프롬프트

```
[기사 분석 결과]
{{text}}

[기사 목록 JSON]
{{articles}}

위 분석 결과를 참고하여 각 기사의 점수를 산정하고 JSON만 반환하세요.

[출력]
```

---

## 코드 노드 (JSON 파싱)

### 역할

LLM-점수산출 노드의 출력을 파싱해 구조화된 배열로 변환한다.

### 방어 처리 목록

| 처리 항목 | 내용 |
| --- | --- |
| 마크다운 코드블록 제거 | ````json` ````` 제거 후 파싱 |
| 불완전 JSON 복구 | 마지막 콤마 제거 후 재파싱 시도 |
| items 키 유연 탐색 | `items` → `articles` → `results` 순으로 시도 |
| article_id 타입 보정 | 문자열/정수/null 모두 처리 |
| reason 빈값 처리 | 빈 reason은 "이유 없음"으로 대체 |
| 배열 타입 검증 | items가 리스트가 아닐 경우 빈 배열 반환 |

### 코드

```python
def main(llm_output: str) -> dict:
    import json, re

    if not llm_output or not llm_output.strip():
        return {"items": []}

    # 마크다운 코드블록 및 앞뒤 텍스트 제거
    text = re.sub(r'```json|```', '', llm_output).strip()

    # JSON 객체 추출 (가장 바깥쪽 중괄호 기준)
    match = re.search(r'\{[\s\S]*\}', text)
    if not match:
        return {"items": []}

    try:
        parsed = json.loads(match.group())
    except json.JSONDecodeError:
        # 마지막 수단: 불완전 JSON 복구 시도
        try:
            partial = match.group().rstrip(',') + '}'
            parsed = json.loads(partial)
        except Exception:
            return {"items": []}

    # items 키 유연하게 탐색
    items = parsed.get("items") or parsed.get("articles") or parsed.get("results") or []
    if not isinstance(items, list):
        return {"items": []}

    importance_map = {"high": 80, "medium": 50, "low": 20}
    result = []

    for item in items:
        if not isinstance(item, dict):
            continue

        # score 처리
        score = item.get("score", 50)
        try:
            score = int(float(score))
        except (ValueError, TypeError):
            score = importance_map.get(str(item.get("importance", "")).lower(), 50)

        # article_id 타입 보정
        article_id = item.get("article_id")
        try:
            article_id = int(article_id) if article_id is not None else None
        except (ValueError, TypeError):
            article_id = str(article_id) if article_id is not None else None

        # reason 처리
        reason = str(item.get("reason", "")).strip()
        if not reason:
            reason = "이유 없음"

        result.append({
            "article_id": article_id,
            "score": max(1, min(100, score)),
            "reason": reason
        })

    return {"items": result}
```

### 입력변수

| 변수명 | 연결 |
| --- | --- |
| llm_output | LLM-점수산출 노드 → 출력 텍스트 |

### 출력변수

| 변수명 | 타입 | 설명 |
| --- | --- | --- |
| items | array | article_id, score, reason 배열 |

---

## 출력

```json
{
  "items": [
    {"article_id": 101, "score": 85, "reason": "한국어 이유"},
    {"article_id": 102, "score": 42, "reason": "한국어 이유"}
  ]
}
```

---

## 변수 흐름 요약

```
시작 노드
  ├─ articles ─────────────→ LLM-분석 유저 프롬프트
  │                                  ↓ (분석 텍스트 출력)
  ├─ articles ─────────────→ LLM-점수산출 유저 프롬프트
  │   LLM-분석 출력 ────────→ LLM-점수산출 유저 프롬프트
  │                                  ↓ (JSON 출력)
  │                          코드 노드 (JSON 파싱)
  │                                  ↓
  │                               출력 (items 배열)
```

---

## 변경 전/후 비교

| 항목 | 변경 전 | 변경 후 |
| --- | --- | --- |
| temperature | 0.7 | 분석 0.3 / 점수산출 0.1 |
| LLM 구조 | 단일 노드 (CoT + JSON 동시) | 2단계 분리 (분석 → JSON 전용) |
| 프롬프트 언어 | 영어 | 한국어 |
| 점수 기준 | 추상적 4단계 | 카테고리별 구체 이벤트 예시 |
| 점수 분산 규칙 | 없음 | 최고·최저 30점 이상 차이 강제 |
| reason 품질 | "1문장으로" 추상 지시 | 영향 범위·시급성·파급력 근거 포함 지시 |
| 코드 노드 방어 처리 | 기본 수준 | 불완전 JSON 복구, 타입 보정, 빈값 처리 추가 |
