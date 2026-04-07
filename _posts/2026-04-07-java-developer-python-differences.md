---
title: "Java 개발자가 Python 처음 배울 때 헷갈리는 것들"
date: 2026-04-07 22:00:00 +0900
categories: [Python, 입문]
tags: [Python, Java, CSharp, Python입문, 문법비교, 개발자전환]
description: Java/C# 개발자 관점에서 Python 입문할 때 헷갈리는 문법 차이점 7가지를 정리합니다.
comments: true
---

Java 개발자로 몇 년 일하다가 Python을 처음 접했을 때 당황했던 부분들을 정리했습니다.

문법 자체는 단순하지만, Java와 발상이 달라서 헷갈리는 포인트들이 있어요.

---

## 1. 들여쓰기가 문법이다

```java
// Java
if (score >= 90) {
    System.out.println("A학점");
}
```

```python
# Python
if score >= 90:
    print("A학점")
```

중괄호 `{}` 대신 들여쓰기로 블록을 구분합니다. 스페이스 4칸이 표준이며, 탭과 스페이스를 혼용하면 `IndentationError` 가 발생합니다.

---

## 2. 타입 선언이 없다

```java
// Java
String name = "잠탱이";
int age = 30;
```

```python
# Python
name = "잠탱이"
age = 30

# 타입 힌트 (선택사항)
name: str = "잠탱이"
age: int = 30
```

타입 힌트는 강제가 아니지만, AI 개발 코드에선 가독성을 위해 쓰는 걸 권장합니다.

---

## 3. `==` vs `is`

```python
a = [1, 2, 3]
b = [1, 2, 3]

a == b   # True  → 값 비교 (Java의 .equals())
a is b   # False → 동일 객체 비교 (Java의 ==)

# None 비교는 항상 is 사용
if result is None:
    print("결과 없음")
```

---

## 4. 세미콜론 없음

```python
name = "잠탱이"  # 세미콜론 불필요
age = 30
```

붙여도 에러는 아니지만 Python 컨벤션에 맞지 않습니다.

---

## 5. 리스트 컴프리헨션

```java
// Java
List<Integer> result = new ArrayList<>();
for (int i = 0; i < 10; i++) {
    if (i % 2 == 0) result.add(i * 2);
}
```

```python
# Python
result = [i * 2 for i in range(10) if i % 2 == 0]
```

데이터 가공 시 자주 사용합니다. AI 개발에서 특히 유용합니다.

---

## 6. `self` 는 항상 명시

```python
class ChatBot:
    def __init__(self, name: str):
        self.name = name      # Java의 this.name

    def greet(self) -> None:  # self 생략 불가
        print(f"안녕하세요, {self.name}입니다")
```

Java의 `this` 와 동일한 개념이지만, Python은 항상 명시해야 합니다.

---

## 7. `catch` → `except`

```java
// Java
try {
    // 코드
} catch (Exception e) {
    System.out.println(e.getMessage());
}
```

```python
# Python
try:
    # 코드
except Exception as e:
    print(str(e))
```

---

## 비교 요약

| Java/C# | Python | 비고 |
|---|---|---|
| `{}` 블록 | 들여쓰기 | 스페이스 4칸 |
| 타입 선언 필수 | 타입 추론 | 힌트는 선택 |
| `.equals()` | `==` | `None`은 `is` |
| `;` 필수 | `;` 없음 | - |
| `this` 생략 가능 | `self` 필수 | - |
| `catch` | `except` | - |

다음 글은 Python 가상환경 venv를 Maven과 비교해서 정리합니다.