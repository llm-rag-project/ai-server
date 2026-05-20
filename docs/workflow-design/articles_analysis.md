# article_analysis 워크플로우 설계

> 기사 목록의 감성(긍정/부정/중립)과 홍보성 여부를 판단하는 워크플로우

---

## 개요

| 항목 | 내용 |
| --- | --- |
| 워크플로우 이름 | article_analysis |
| 역할 | 기사 목록 → 감성(긍정/부정/중립) + 홍보성 여부 배열 반환 |
| 트리거 방식 | 메인 서버 배치 자동 호출 (매일 오전 8시) |
| 모델 | gemini-2.5-pro |

---

## 구조

```
[변경 전]
시작 → LLM → 코드(JSON 파싱) → 출력

[변경 후]
시작 → 코드1(입력검증) → LLM(감성/분류 분석) → 코드2(JSON 파싱) → 출력
```

---

## 시작 노드 입력변수

| 변수명 | 타입 | 필수 여부 | 설명 |
| --- | --- | --- | --- |
| articles | paragraph | 필수 | 기사를 담은 JSON 문자열, 최대 30건 |

> **articles 설계 의도**
> 크롤링 배치 완료 후 메인 서버가 자동 호출하며, 수집된 기사 전체를 단일 호출로 처리하여 API 호출 횟수를 최소화한다.
> Dify 시작 노드가 Array[Object]를 지원하지 않아 json.dumps()로 직렬화된 JSON 문자열로 전달한다.

**articles 원소 구조**

```json
[
  {
    "article_id": 101,
    "title": "기사 제목",
    "content": "기사 본문 전체 텍스트"
  }
]
```

---

## 코드1 노드 (입력검증)

### 역할

시작 노드에서 받은 `articles`를 검증하여 LLM 노드에 안전하게 전달한다. 빈 값, JSON 파싱 불가, 빈 배열, 30건 초과 시 즉시 예외를 발생시켜 LLM 호출을 방지한다.

### 입력변수

| 변수명 | 연결 |
| --- | --- |
| articles | 시작 노드 → articles |

### 출력변수

| 변수명 | 타입 | 설명 |
| --- | --- | --- |
| articles_validated | string | 검증 완료된 articles JSON 문자열 |

### 처리 내용

| 처리 항목 | 내용 |
| --- | --- |
| 빈 값 처리 | articles가 비어있으면 ValueError 발생 |
| JSON 파싱 실패 처리 | 파싱 오류 시 ValueError 발생 |
| 빈 배열 처리 | 배열이 비어있으면 ValueError 발생 |
| 최대 건수 제한 | 30건 초과 시 ValueError 발생 |

### 코드

```python
import json

def main(articles: str) -> dict:
    if not articles or articles.strip() == "":
        raise ValueError("articles가 비어있습니다.")
    try:
        parsed = json.loads(articles)
        if not isinstance(parsed, list) or len(parsed) == 0:
            raise ValueError("articles가 빈 배열입니다.")
        if len(parsed) > 30:
            raise ValueError("articles 최대 30건 초과")
    except json.JSONDecodeError:
        raise ValueError("articles JSON 파싱 실패")

    return {"articles_validated": articles}
```

---

## LLM 노드 (감성/분류 분석)

### 역할

각 기사의 감성(긍정/부정/중립)과 홍보성 여부를 분석하여 JSON만 반환한다.

### 왜 CoT 분리 없이 단일 LLM으로 설계했나

importance_scoring 워크플로우와 달리 감성 분류와 홍보성 판단은 복잡한 수치 산정이 아닌 **단순 레이블 분류 작업**이다. 분석 근거 텍스트를 별도로 생성할 필요 없이 기사 논조를 직접 판단하면 되므로, LLM 단일 노드로 JSON만 출력하는 구조가 더 효율적이다.

### 설정값

| 항목 | 값 |
| --- | --- |
| 모델 | gemini-2.5-pro |
| temperature | 0.1 |

**temperature를 0.1로 낮춘 이유:**
감성 분류와 홍보성 판단은 일관성이 핵심이다. 같은 기사를 두 번 넣었을 때 결과가 달라지면 서비스 신뢰도가 떨어진다. 창의성이 전혀 필요 없는 분류 작업이므로 최저값에 가깝게 설정했다.

### 입력변수

| 변수명 | 연결 |
| --- | --- |
| articles_validated | 코드1 노드 → articles_validated |

### 시스템 프롬프트

```
You are a Korean news article analyzer.
Analyze each article and return results in JSON format.
The response must be a valid JSON object containing the key "results".
You must respond ONLY with a valid JSON object.
Do not include any explanation, markdown, or text outside the JSON.
The word "json" is included in this prompt for format compliance.
```

### 유저 프롬프트

```
아래 기사 목록을 분석해서 각 기사의 감성과 홍보성 여부를 판단해줘.

기사 목록: {{#코드1노드ID.articles_validated#}}

각 기사에 대해 다음 기준으로 분석해:

[감성 분석 기준 - sentiment]
- "긍정": 기사 전체 논조가 긍정적 (성장, 성과, 호재 등)
- "부정": 기사 전체 논조가 부정적 (하락, 사고, 악재 등)
- "중립": 사실 전달 위주이거나 논조가 혼재

[홍보성 분류 기준 - is_promotion]
- true: 특정 제품/서비스/기업 홍보가 주목적인 보도자료성 기사
- false: 독립적 취재 또는 사실 보도 기사

반드시 아래 JSON 형식으로만 응답해:
{
  "results": [
    {
      "article_id": 101,
      "sentiment": "중립",
      "is_promotion": false
    },
    {
      "article_id": 102,
      "sentiment": "긍정",
      "is_promotion": true
    }
  ]
}
```

---

## 코드2 노드 (JSON 파싱)

### 역할

LLM 노드의 출력을 파싱해 구조화된 배열로 변환한다.

### 입력변수

| 변수명 | 연결 |
| --- | --- |
| arg1 | LLM 노드 → 출력 텍스트 |

### 출력변수

| 변수명 | 타입 | 설명 |
| --- | --- | --- |
| results | string | article_id, sentiment, is_promotion 배열 JSON 문자열 |

### 방어 처리 목록

| 처리 항목 | 내용 |
| --- | --- |
| 리터럴 \n 변환 | `\n` 문자열을 실제 줄바꿈으로 변환 후 파싱 |
| 마크다운 코드블록 제거 | ` ```json ` ` ``` ` 제거 후 파싱 |
| JSON 객체 추출 | 가장 바깥쪽 중괄호 기준으로 추출 |
| results 키 유연 탐색 | `results` → `결과` 순으로 시도 |
| article_id 누락 처리 | `article_id` → `기사id` → `기사_id` 순으로 is not None 비교 |
| sentiment 유효값 검증 | 긍정/부정/중립 외 값은 `분석실패`로 대체 |
| is_promotion 타입 보정 | None이면 `홍보성` 키로 재시도, 이후에도 None이면 null 유지 |

### 코드

```python
import json
import re

def main(arg1: str) -> dict:
    text = arg1.strip()

    # 리터럴 \n을 실제 줄바꿈으로 변환
    text = text.replace('\\n', '\n')

    # 마크다운 펜스 제거
    text = re.sub(r'```json|```', '', text).strip()

    # JSON 블록 추출
    match = re.search(r'\{.*\}', text, re.DOTALL)
    if not match:
        return {"results": "[]"}

    try:
        parsed = json.loads(match.group())
    except json.JSONDecodeError:
        return {"results": "[]"}

    results_raw = (
        parsed.get("results") or
        parsed.get("결과") or
        []
    )

    results = []
    for item in results_raw:
        article_id = (
            item.get("article_id")
            if item.get("article_id") is not None
            else item.get("기사id")
            if item.get("기사id") is not None
            else item.get("기사_id")
        )

        sentiment = (
            item.get("sentiment") or
            item.get("감성") or
            "분석실패"
        )
        if sentiment not in ["긍정", "부정", "중립"]:
            sentiment = "분석실패"

        is_promotion = item.get("is_promotion")
        if is_promotion is None:
            is_promotion = item.get("홍보성")
        if is_promotion is not None:
            is_promotion = bool(is_promotion)

        if article_id is not None:
            results.append({
                "article_id": article_id,
                "sentiment": sentiment,
                "is_promotion": is_promotion
            })

    return {"results": json.dumps(results, ensure_ascii=False)}
```

---

## 출력 노드

코드2 노드의 `results`를 그대로 반환한다.

```json
{
  "results": [
    {"article_id": 101, "sentiment": "중립", "is_promotion": false},
    {"article_id": 102, "sentiment": "긍정", "is_promotion": true},
    {"article_id": 103, "sentiment": "부정", "is_promotion": false}
  ]
}
```

---

## 변수 흐름 요약

```
시작 노드
  └─ articles ──────────────→ 코드1 (입력검증)
                                     ↓ (articles_validated)
                              LLM (감성/분류 분석)
                                     ↓ (JSON 텍스트 출력)
                              코드2 (JSON 파싱)
                                     ↓ (results 배열)
                                   출력
```

---

## 변경 전/후 비교

| 항목 | 변경 전 | 변경 후 |
| --- | --- | --- |
| temperature | 0.7 | 0.1 |
| 시스템 프롬프트 | JSON 형식 안내만 | JSON 외 텍스트 일절 금지 명시 추가 |
| 입력검증 | 없음 | 코드1 노드에서 빈값·파싱오류·건수초과 방어 |
| article_id 방어 처리 | or 체인 (0 falsy 오류 가능) | is not None 명시적 비교로 수정 |
| is_promotion 폴백 | false 고정 | null 유지 (분석실패 구분 가능) |
| sentiment 폴백 | 중립 고정 | 분석실패 명시 (정상값과 구분 가능) |
