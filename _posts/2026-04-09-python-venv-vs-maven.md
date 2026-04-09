---
title: "Python 가상환경 venv — Java Maven 쓰던 개발자가 이해한 방식"
date: 2026-04-09 22:00:00 +0900
categories: [Python, 환경설정]
tags: [Python, venv, 가상환경, pip, Maven, Java, 환경설정]
description: Java Maven 개발자 관점에서 Python 가상환경 venv를 이해하고 설정하는 방법을 정리합니다.
comments: true
---

Python 입문 시 가장 먼저 마주치는 개념인 가상환경을 Java Maven 관점에서 정리합니다.

---

## 왜 가상환경이 필요한가

Python은 기본적으로 패키지를 전역(global)으로 설치합니다. 프로젝트가 하나라면 문제없지만, 여러 프로젝트를 동시에 관리할 때 의존성 충돌이 발생합니다.

```
프로젝트 A → requests==2.28.0 필요
프로젝트 B → requests==2.20.0 필요 (하위 호환성 문제)
```

가상환경은 프로젝트별로 독립된 패키지 공간을 격리합니다.

---

## Java Maven과의 비교

| 개념 | Java (Maven) | Python (venv) |
|---|---|---|
| 의존성 파일 | `pom.xml` | `requirements.txt` |
| 패키지 설치 | `mvn install` | `pip install` |
| 로컬 저장소 | `.m2` | 가상환경 폴더 |
| 의존성 복원 | `mvn install` | `pip install -r requirements.txt` |
| 버전 잠금 | `pom.xml` 버전 명시 | `pip freeze` |

구조적으로 거의 동일한 개념입니다.

---

## 기본 사용법

```bash
# 1. 가상환경 생성
python -m venv venv

# 2. 활성화 (Windows)
venv\Scripts\activate

# 2. 활성화 (Mac/Linux)
source venv/bin/activate

# 3. 패키지 설치
pip install fastapi google-generativeai

# 4. 의존성 저장 (pom.xml 역할)
pip freeze > requirements.txt

# 5. 의존성 복원
pip install -r requirements.txt

# 6. 비활성화
deactivate
```

---

## VS Code 인터프리터 설정

```
Ctrl + Shift + P
→ Python: Select Interpreter
→ ./venv/Scripts/python.exe 선택
```

설정 후 터미널 오픈 시 자동으로 가상환경이 활성화됩니다.

---

## .gitignore 설정

```gitignore
# 가상환경 폴더
venv/

# 캐시
__pycache__/
*.pyc

# 환경변수 (API 키 등)
.env
```

`requirements.txt` 로 환경 재현이 가능하므로 `venv/` 폴더는 반드시 gitignore 처리합니다.

---

## AI 개발 프로젝트 기본 패키지

```bash
pip install google-generativeai    # Gemini API
pip install fastapi                # 백엔드 프레임워크
pip install uvicorn                # ASGI 서버
pip install python-dotenv          # .env 파일 로드
pip install langchain              # LLM 파이프라인
pip install chromadb               # 벡터 DB

pip freeze > requirements.txt
```

다음 글은 Gemini API 첫 연동입니다.