---
title: "Python 문법 핵심만 — C# 개발자 관점 정리"
date: 2026-04-20 22:00:00 +0900
categories: [Python, 입문]
tags: [Python, CSharp, Python문법, 타입힌트, lambda, enumerate, 개발자전환]
description: C# 개발자 관점에서 Python 핵심 문법을 비교 정리합니다. f-string, None, 딕셔너리, with문, *args/**kwargs, 람다, 타입 힌트.
comments: true
---

C# 개발자 관점에서 Python 핵심 문법을 비교 정리합니다.

---

## f-string (문자열 보간)

```csharp
// C#
Console.WriteLine($"이름: {name}, 나이: {age}");
```

```python
# Python
print(f"이름: {name}, 나이: {age}")
```

`$""` → `f""` 로 바뀐 것뿐, 구조는 동일합니다.

---

## None (null)

```python
result = None

# None 비교는 is 사용 권장
if result is None:
    print("결과 없음")
```

---

## 딕셔너리

```python
# 생성
data = {"name": "잠탱이", "age": 30}

# 접근
print(data["name"])

# 키 없을 때 기본값 (KeyNotFoundException 방지)
print(data.get("email", "없음"))  # "없음"
```

---

## 튜플 언패킹

```python
def get_token_info(text: str) -> tuple[int, int]:
    return len(text), len(text.encode('utf-8'))

chars, byte_size = get_token_info("안녕하세요")
```

함수에서 여러 값 반환 시 자주 사용합니다.

---

## enumerate

```python
items = ["사과", "바나나", "딸기"]
for i, item in enumerate(items):
    print(f"{i}: {item}")
```

C#의 `for (int i = 0; ...)` 패턴을 대체합니다.

---

## with 문 (using)

```csharp
// C#
using (var file = File.OpenRead("data.txt")) { ... }
```

```python
# Python
with open("data.txt", "r", encoding="utf-8") as f:
    content = f.read()
```

---

## *args, **kwargs

```python
def add(*args: int) -> int:
    return sum(args)

add(1, 2, 3, 4)  # 10

def create_model(**kwargs):
    print(kwargs)  # {'model': 'gemini-1.5-flash', 'temperature': 0.7}

create_model(model="gemini-1.5-flash", temperature=0.7)
```

LangChain, FastAPI 코드에서 자주 등장합니다.

---

## 람다

```python
items = ["banana", "apple", "cherry"]

# 길이 기준 정렬
sorted_items = sorted(items, key=lambda x: len(x))

# 조건 필터링
long_items = list(filter(lambda x: len(x) > 5, items))
```

---

## 타입 힌트

```python
from typing import Optional

def chat(
    message: str,
    history: list[dict] = [],
    temperature: float = 0.7,
    system_prompt: Optional[str] = None
) -> str:
    ...
```

FastAPI는 타입 힌트를 적극 활용합니다. C# 개발자라면 타입 힌트 작성을 습관화하는 것을 권장합니다.

---

## 비교 요약

| 개념 | C# | Python |
|---|---|---|
| 문자열 보간 | `$"값: {x}"` | `f"값: {x}"` |
| null | `null` | `None` |
| null 비교 | `== null` | `is None` |
| 파일 처리 | `using` | `with` |
| 가변 인수 | `params T[]` | `*args` |
| 키워드 가변 인수 | - | `**kwargs` |
| 람다 | `x => x.Length` | `lambda x: len(x)` |

다음 글은 Gemini API 첫 연동입니다.