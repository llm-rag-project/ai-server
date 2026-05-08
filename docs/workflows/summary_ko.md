# summary_ko 워크플로우 설계

> 뉴스 기사 1건을 입력받아 한국어 요약문을 생성하는 워크플로우
> 

---

## 개요

| 항목 | 내용 |
| --- | --- |
| 워크플로우 이름 | summary_ko |
| 역할 | 뉴스 기사 1건 → 한국어 요약문 생성 |
| 트리거 방식 | 사용자가 버튼을 눌렀을 때 호출 |
| 모델 | gemini-2.5-pro |
| temperature | 0.3 |

---

## 구조

```
시작 → 코드(전처리) → LLM → 출력
```

---

## 시작 노드 입력변수

| 변수명 | 타입 | 필수 여부 | 설명 |
| --- | --- | --- | --- |
| user_id | string | 필수 | 사용자 식별자 |
| article_id | string | 필수 | 기사 식별자 |
| title | string | 필수 | 기사 제목 |
| content | string | 필수 | 기사 본문 |
| message | string | 선택 | 현재 미사용. 향후 기능 확장을 위해 필드만 유지 |

> **message 필드 설계 의도**
현재 이 워크플로우는 버튼 트리거 방식이라 사용자가 요약 방식을 직접 지정하는 케이스가 없다. 하지만 나중에 "3줄로 요약해줘" 같은 옵션이 추가될 수 있어 필드는 시작 노드에만 존재하고 어디에도 연결하지 않는다. 기능 확장 시 코드 노드 → 프롬프트 순서로 붙이면 된다.
> 

---

## 전처리 코드 노드

### 역할

LLM에게 언어 판별과 길이 판단을 맡기는 대신, 코드에서 미리 계산해서 힌트 값으로 주입한다.

**이렇게 하는 이유:**

- LLM이 매번 언어를 추론하는 비용을 줄인다
- 언어 판별과 길이 판단이 코드 레벨에서 일관되게 처리된다
- LLM의 판단 결과가 매번 달라지는 불안정성을 제거한다

### 입력변수

| 변수명 | 연결 |
| --- | --- |
| content | 시작 노드 → content |

> message는 코드 노드에 연결하지 않는다.
> 

### 코드

```python
def main(content: str) -> dict:
    content = content.strip() if content else ""
    char_count = len(content)
    is_short = char_count < 200
    korean_chars = sum(1 for c in content if '\uAC00' <= c <= '\uD7A3')
    ratio = korean_chars / max(char_count, 1)
    lang_hint = "korean" if ratio > 0.3 else "non-korean"
    return {
        "char_count": char_count,
        "is_short": "true" if is_short else "false",
        "lang_hint": lang_hint
    }
```

### 출력변수

| 변수명 | 타입 | 설명 |
| --- | --- | --- |
| char_count | int | 본문 글자 수 (내부 계산용, LLM에 전달하지 않음) |
| is_short | string | "true" 또는 "false" (Boolean 대신 String 사용 — Dify가 Boolean 타입을 유저 프롬프트 변수 목록에 표시하지 않기 때문) |
| lang_hint | string | "korean" 또는 "non-korean" |

### 판별 기준

- **lang_hint**: 전체 글자 중 한국어 문자(가~힣) 비율이 30% 초과면 "korean", 이하면 "non-korean"
- **is_short**: 본문 글자 수 200자 미만이면 "true", 이상이면 "false"

---

## LLM 노드

### 설정값

| 항목 | 값 |
| --- | --- |
| 모델 | gemini-2.5-pro |
| temperature | 0.3 |

**temperature를 0.3으로 낮춘 이유:**
요약은 창의적 표현이 아니라 사실의 정확한 전달이 목적이다. temperature가 높으면 같은 기사를 요약해도 매번 다른 표현, 다른 강조점이 나온다. 0.3은 자연스러운 한국어 문장을 유지하면서도 일관된 출력을 보장하는 적정값이다.

---

### 시스템 프롬프트

```
당신은 뉴스 기사 요약 전문가입니다. 입력된 기사 1건을 한국어로 정확하고 간결하게 요약합니다.

---

[언어 처리]
- lang_hint가 "korean"이면 바로 요약하세요.
- lang_hint가 "non-korean"이면 내용을 완전히 이해한 뒤 한국어로 번역하여 요약하세요.
- 출력은 반드시 한국어로 작성하세요.

---

[요약 구조]
아래 순서로 문장을 구성하세요.
1. [핵심 사실 문장]: 누가/무엇을/어떻게 했는지 — 가장 중요한 사실 1문장
2. [배경·맥락 문장]: 왜 이 일이 일어났는지, 어떤 상황에서인지 — 1문장
3. [결과·전망 문장]: 어떤 결과가 나왔거나 예상되는지 — 필요할 때만 1문장

출력은 2~3문장으로 작성하세요.

---

[짧은 본문 처리]
- is_short가 "true" (200자 미만)이면:
  - 요약 구조 대신 핵심 내용을 1문장으로 압축하세요.
  - 요약문 끝에 "(본문 내용이 짧아 요약이 제한될 수 있습니다.)"를 부기하세요.

---

[금지 사항]
- 추측, 과장, 원문에 없는 정보 추가 금지
- 광고성 문구, 불필요한 수식어, 인사말, 머리말 금지
- "기사 id:", "요약본:", JSON, 코드블록, 설명 문장 출력 금지
- 오직 최종 요약문만 반환하세요.

---

[Few-shot 예시]

예시 1 — 영어 기사, 일반 케이스
lang_hint: non-korean
is_short: false
title: "Fed Raises Interest Rates by 25 Basis Points Amid Inflation Concerns"
content: "The Federal Reserve raised its benchmark interest rate by 25 basis points on Wednesday, bringing it to a 22-year high..."

출력:
미국 연방준비제도(Fed)가 인플레이션 억제를 위해 기준금리를 0.25%포인트 인상해 22년 만의 최고 수준에 도달했다. 물가 상승세가 지속되는 가운데 추가 긴축 가능성을 차단하지 않겠다는 신호로 해석된다.

예시 2 — 영어 기사, 짧은 본문
lang_hint: non-korean
is_short: true
title: "Apple Delays New Product Launch"
content: "Apple has postponed its scheduled product event due to supply chain issues."

출력:
애플이 공급망 문제로 신제품 발표 행사를 연기했다. (본문 내용이 짧아 요약이 제한될 수 있습니다.)

예시 3 — 한국어 기사, 일반 케이스
lang_hint: korean
is_short: false
title: "서울시, 대중교통 심야 운행 확대 추진"
content: "서울시가 오는 하반기부터 지하철과 버스의 심야 운행 시간을 현행보다 1시간 연장하는 방안을 검토 중이다..."

출력:
서울시가 하반기부터 지하철·버스 심야 운행을 1시간 연장하는 방안을 검토 중이다. 늦은 귀가 시민의 편의를 높이기 위해 관계 기관과 협의를 진행하고 있다.
```

**Few-shot을 추가한 이유:**
Few-shot은 LLM에게 "이런 형태로 출력하면 된다"는 구체적 패턴을 보여주는 가장 효과적인 방법이다. 특히 영어 기사 번역 요약처럼 다단계 처리가 필요한 케이스는 예시 없이는 출력 품질이 불안정하다.

추가한 예시:

- 영어 기사 → 한국어 번역 요약 (일반 케이스)
- 영어 기사 + 본문 짧음 → 1문장 + 부기 (짧은 본문 케이스)
- 한국어 기사 → 2문장 요약 (일반 케이스)

---

### 유저 프롬프트

```
article_id: {{article_id}}
title: {{title}}
content: {{content}}
lang_hint: {{lang_hint}}
is_short: {{is_short}}
```

> char_count는 내부 계산용이므로 LLM에 전달하지 않는다.
> 

---

## 출력

| 변수명 | 타입 | 설명 |
| --- | --- | --- |
| summary_text | string | 한국어 요약문 |

---

## 변수 흐름 요약

```
시작 노드
  ├─ content ──────────────→ 코드 노드 (lang_hint, is_short 계산)
  │                                  ↓
  ├─ article_id ───────────→ LLM 유저 프롬프트
  ├─ title ────────────────→ LLM 유저 프롬프트
  ├─ content ──────────────→ LLM 유저 프롬프트
  │          코드 노드 출력 ─→ LLM 유저 프롬프트 (lang_hint, is_short)
  │
  └─ message ──────────────→ (연결 없음, 향후 확장용)
```

---

## 변경 전/후 비교

| 항목 | 변경 전 | 변경 후 |
| --- | --- | --- |
| temperature | 0.6 | 0.3 |
| 구조 | 시작 → LLM → 출력 | 시작 → 코드(전처리) → LLM → 출력 |
| 언어 판별 | LLM이 직접 판단 | 코드 노드에서 계산 후 주입 |
| 짧은 본문 처리 | 미정의 | 200자 미만 시 1문장 + 부기 |
| 문장 구조 | 미정의 (매번 다름) | 2문장 고정 |
| Few-shot | 없음 | 3개 추가 |

---

## 향후 message 기능 확장 시 수정 순서

1. 코드 노드 입력변수에 `message` 추가
2. 코드 노드에서 허용 키워드 필터링 후 `message_safe` 출력
3. 시스템 프롬프트에 분량 기준 추가
4. 유저 프롬프트에 `message_safe: {{message_safe}}` 추가
