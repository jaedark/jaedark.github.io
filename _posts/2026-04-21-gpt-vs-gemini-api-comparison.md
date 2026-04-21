---
title: "GPT API vs Gemini API — 구조 비교와 선택 기준"
date: 2026-04-21 22:00:00 +0900
categories: [AI 개발 실습]
tags: [gemini, openai, gpt, api, llm, python]
---

## 개요

OpenAI GPT API와 Google Gemini API는 현재 가장 널리 사용되는 LLM API다.  
두 API는 동일한 목적(텍스트 생성)을 수행하지만 인터페이스 설계, 요금 정책, 생태계 측면에서 차이가 있다.  
이 글에서는 코드 구조 비교, 요금 모델, 호환성 전략을 중심으로 다룬다.

---

## 1. 요금 정책 비교

| 항목 | OpenAI GPT API | Google Gemini API |
|------|---------------|-------------------|
| 무료 티어 | 신규 가입 크레딧 (소진 후 유료) | 무료 티어 상시 제공 |
| 카드 등록 | 필요 | 불필요 (무료 구간) |
| 종량제 | 입력/출력 토큰 기준 | 입력/출력 토큰 기준 |
| 무료 한도 (Flash) | — | 분당 15req, 일 1,500req |

프로토타이핑 및 학습 단계에서는 Gemini API의 무료 티어가 실질적인 진입 장벽을 낮춰준다.

---

## 2. API 인터페이스 구조 비교

### 2-1. OpenAI GPT API

```python
from openai import OpenAI

client = OpenAI(api_key=os.getenv("OPENAI_API_KEY"))

response = client.chat.completions.create(
    model="gpt-4o-mini",
    messages=[
        {"role": "system", "content": "당신은 반도체 공정 전문가입니다."},
        {"role": "user", "content": "CVD 공정에서 온도 편차가 수율에 미치는 영향은?"}
    ],
    temperature=0.7,
    max_tokens=512
)

print(response.choices[0].message.content)
print(f"토큰 사용량: {response.usage}")
```

핵심 구조: `messages` 리스트에 `role` + `content` 딕셔너리를 누적하는 방식이다.  
시스템 프롬프트는 `role: "system"` 메시지로 삽입한다.

### 2-2. Google Gemini API

```python
import google.generativeai as genai

genai.configure(api_key=os.getenv("GEMINI_API_KEY"))

model = genai.GenerativeModel(
    model_name="gemini-1.5-flash",
    system_instruction="당신은 반도체 공정 전문가입니다.",
    generation_config=genai.GenerationConfig(
        temperature=0.7,
        max_output_tokens=512
    )
)

response = model.generate_content(
    "CVD 공정에서 온도 편차가 수율에 미치는 영향은?"
)

print(response.text)
print(f"토큰 사용량: {response.usage_metadata}")
```

시스템 프롬프트는 모델 초기화 시 `system_instruction`으로 분리되어 있고,  
생성 파라미터는 `GenerationConfig` 객체로 묶어 전달한다.

---

## 3. 멀티턴 대화 구조 비교

### OpenAI — messages 누적 방식

```python
messages = [{"role": "system", "content": "반도체 공정 전문가"}]

def chat(user_input):
    messages.append({"role": "user", "content": user_input})
    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=messages
    )
    reply = response.choices[0].message.content
    messages.append({"role": "assistant", "content": reply})
    return reply
```

### Gemini — ChatSession 방식

```python
chat = model.start_chat(history=[])

def chat_gemini(user_input):
    response = chat.send_message(user_input)
    return response.text

# 히스토리 접근
for turn in chat.history:
    print(f"[{turn.role}] {turn.parts[0].text[:60]}")
```

Gemini의 `ChatSession`은 내부적으로 히스토리를 관리하므로 별도의 누적 로직이 필요 없다.  
단, 세션 객체가 소멸되면 히스토리도 사라지므로 영속성이 필요한 경우 직렬화가 필요하다.

---

## 4. OpenAI 호환 인터페이스 (Gemini)

Gemini API는 OpenAI SDK와 호환되는 엔드포인트를 제공한다.  
기존 GPT 기반 코드를 최소한의 수정으로 Gemini로 전환할 수 있다.

```python
from openai import OpenAI

# base_url과 모델명만 변경
client = OpenAI(
    api_key=os.getenv("GEMINI_API_KEY"),
    base_url="https://generativelanguage.googleapis.com/v1beta/openai/"
)

response = client.chat.completions.create(
    model="gemini-1.5-flash",   # 모델명만 변경
    messages=[
        {"role": "user", "content": "FastAPI와 Flask의 차이점은?"}
    ]
)

print(response.choices[0].message.content)
```

이 방식은 멀티 프로바이더 지원이 필요한 경우 유용하다.  
LangChain의 `ChatOpenAI`도 동일한 방식으로 Gemini에 연결 가능하다.

---

## 5. 모델 라인업 비교

### OpenAI

| 모델 | 특징 |
|------|------|
| gpt-4o | 최고 성능, 멀티모달 |
| gpt-4o-mini | 비용 효율, 일반 태스크 |
| o1, o3 | 추론 특화 (느림, 고비용) |

### Google Gemini

| 모델 | 특징 |
|------|------|
| gemini-1.5-flash | 빠름, 저비용, 무료 티어 |
| gemini-1.5-pro | 고성능, 긴 컨텍스트 (1M 토큰) |
| gemini-2.0-flash | 최신 경량, 실시간 처리 |

Gemini 1.5 Pro의 최대 컨텍스트 길이(1M 토큰)는 긴 문서 처리에서 GPT-4o(128K) 대비 강점이다.

---

## 6. LangChain 연동 비교

LangChain을 사용할 경우 프로바이더별 클래스가 분리되어 있다.

```python
# OpenAI
from langchain_openai import ChatOpenAI
llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

# Gemini
from langchain_google_genai import ChatGoogleGenerativeAI
llm = ChatGoogleGenerativeAI(model="gemini-1.5-flash", temperature=0)

# 이후 체인 구성은 동일
from langchain_core.prompts import ChatPromptTemplate

prompt = ChatPromptTemplate.from_messages([
    ("system", "반도체 공정 전문가로 답변하세요."),
    ("human", "{question}")
])

chain = prompt | llm
response = chain.invoke({"question": "APC란 무엇인가요?"})
print(response.content)
```

LangChain의 추상화 덕분에 `llm` 객체만 교체하면 나머지 코드는 동일하게 동작한다.

---

## 7. 선택 기준 정리

```
학습/프로토타이핑     → Gemini API (무료, 빠른 시작)
레퍼런스 중심 학습    → GPT API (커뮤니티, 튜토리얼 풍부)
긴 문서 처리         → Gemini 1.5 Pro (컨텍스트 우위)
추론 집약적 태스크    → OpenAI o1/o3
멀티 프로바이더 설계  → LangChain 추상화 레이어 활용
```

---

## 참고

- [OpenAI API 공식 문서](https://platform.openai.com/docs)
- [Gemini API 공식 문서](https://ai.google.dev/gemini-api/docs)
- [Gemini OpenAI 호환 가이드](https://ai.google.dev/gemini-api/docs/openai)
- [LangChain Google Generative AI](https://python.langchain.com/docs/integrations/chat/google_generative_ai/)