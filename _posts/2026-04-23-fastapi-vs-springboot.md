---
title: "FastAPI 입문 — Spring Boot 개발자를 위한 구조 비교"
date: 2026-04-23 22:00:00 +0900
categories: [AI 개발 실습]
tags: [fastapi, springboot, python, uvicorn, backend, rest-api, pydantic]
---

## 개요

FastAPI는 Python 기반의 비동기 웹 프레임워크로, AI 모델 서빙과 챗봇 백엔드 구축에 널리 사용된다.  
Spring Boot 경험이 있는 개발자라면 개념적 대응 관계를 파악하면 학습 곡선을 크게 낮출 수 있다.  
이 글에서는 두 프레임워크의 구조적 차이, 코드 패턴, 프로젝트 구성 방식을 비교한다.

---

## 1. 기술 스택 비교

| 항목 | Spring Boot | FastAPI |
|------|-------------|---------|
| 언어 | Java / Kotlin | Python |
| 서버 | 내장 Tomcat (동기) | uvicorn / ASGI (비동기) |
| 의존성 관리 | Maven / Gradle | pip / requirements.txt |
| 유효성 검사 | Bean Validation | Pydantic |
| API 문서 | springdoc-openapi (별도) | 자동 생성 (/docs, /redoc) |
| DI 컨테이너 | Spring IoC | 없음 (Depends 활용) |

---

## 2. 설치 및 실행

```bash
pip install fastapi uvicorn python-dotenv
```

**서버 실행:**
```bash
uvicorn main:app --reload --host 0.0.0.0 --port 8000
```

| 옵션 | 설명 |
|------|------|
| `main:app` | `main.py` 내 `app` 인스턴스 |
| `--reload` | 코드 변경 시 자동 재시작 (개발용) |
| `--host 0.0.0.0` | 외부 접근 허용 |
| `--workers 4` | 프로덕션 멀티프로세스 |

---

## 3. 라우터 및 핸들러

### Spring Boot
```java
@RestController
@RequestMapping("/api/chat")
public class ChatController {

    @PostMapping
    public ResponseEntity<ChatResponse> chat(@RequestBody ChatRequest request) {
        // 처리 로직
        return ResponseEntity.ok(new ChatResponse(reply));
    }
}
```

### FastAPI
```python
from fastapi import FastAPI, APIRouter
from pydantic import BaseModel

router = APIRouter(prefix="/api/chat", tags=["chat"])

class ChatRequest(BaseModel):
    message: str
    user_id: str = "anonymous"

class ChatResponse(BaseModel):
    reply: str
    tokens_used: int

@router.post("", response_model=ChatResponse)
async def chat(request: ChatRequest):
    # 처리 로직
    return ChatResponse(reply=reply, tokens_used=token_count)

app = FastAPI()
app.include_router(router)
```

`response_model` 지정 시 자동으로 응답 스키마가 `/docs`에 반영된다.

---

## 4. 요청 파라미터 처리

```python
from fastapi import FastAPI, Query, Path
from typing import Optional

app = FastAPI()

# 경로 파라미터 (@PathVariable)
@app.get("/sessions/{session_id}")
async def get_session(session_id: str = Path(..., description="세션 ID")):
    return {"session_id": session_id}

# 쿼리 파라미터 (@RequestParam)
@app.get("/messages")
async def get_messages(
    session_id: str,
    limit: int = Query(default=20, ge=1, le=100),
    offset: int = Query(default=0, ge=0)
):
    return {"session_id": session_id, "limit": limit, "offset": offset}
```

`Query(ge=1, le=100)` 같은 제약 조건이 자동으로 `/docs` 스키마에 반영된다.

---

## 5. Pydantic 모델 (DTO 대응)

Spring Boot의 DTO + Bean Validation에 대응하는 Pydantic 사용법:

```python
from pydantic import BaseModel, Field, validator
from typing import Optional
from datetime import datetime

class ChatRequest(BaseModel):
    message: str = Field(..., min_length=1, max_length=2000, description="사용자 메시지")
    session_id: Optional[str] = Field(default=None, description="세션 ID (없으면 신규 생성)")
    temperature: float = Field(default=0.7, ge=0.0, le=1.0)

    @validator("message")
    def message_must_not_be_empty(cls, v):
        if not v.strip():
            raise ValueError("메시지는 공백일 수 없습니다")
        return v.strip()

class ChatResponse(BaseModel):
    reply: str
    session_id: str
    created_at: datetime
    tokens_used: int
```

---

## 6. 의존성 주입 (Depends)

Spring의 `@Autowired` 대신 FastAPI는 `Depends`를 사용한다.

```python
from fastapi import Depends, HTTPException, Header
import google.generativeai as genai
import os

# 서비스 레이어 (싱글턴처럼 동작)
def get_gemini_model():
    genai.configure(api_key=os.getenv("GEMINI_API_KEY"))
    return genai.GenerativeModel("gemini-1.5-flash")

# 인증 미들웨어
def verify_api_key(x_api_key: str = Header(...)):
    if x_api_key != os.getenv("INTERNAL_API_KEY"):
        raise HTTPException(status_code=401, detail="Invalid API Key")
    return x_api_key

@app.post("/chat")
async def chat(
    request: ChatRequest,
    model = Depends(get_gemini_model),
    _ = Depends(verify_api_key)
):
    response = model.generate_content(request.message)
    return {"reply": response.text}
```

---

## 7. 예외 처리

```python
from fastapi import HTTPException
from fastapi.responses import JSONResponse
from fastapi.exceptions import RequestValidationError

# 커스텀 예외
class AIServiceException(Exception):
    def __init__(self, message: str, status_code: int = 500):
        self.message = message
        self.status_code = status_code

# 전역 예외 핸들러
@app.exception_handler(AIServiceException)
async def ai_service_exception_handler(request, exc: AIServiceException):
    return JSONResponse(
        status_code=exc.status_code,
        content={"error": exc.message}
    )

@app.exception_handler(RequestValidationError)
async def validation_exception_handler(request, exc):
    return JSONResponse(
        status_code=422,
        content={"error": "요청 형식 오류", "detail": exc.errors()}
    )

# 라우터에서 사용
@app.post("/chat")
async def chat(request: ChatRequest):
    try:
        response = model.generate_content(request.message)
        return {"reply": response.text}
    except Exception as e:
        raise AIServiceException(message=f"AI 서비스 오류: {str(e)}", status_code=503)
```

---

## 8. 프로젝트 구조

```
project/
├── main.py                  # 앱 진입점, 라우터 등록
├── routers/
│   ├── __init__.py
│   └── chat.py              # 채팅 관련 엔드포인트
├── models/
│   ├── __init__.py
│   └── schemas.py           # Pydantic 모델 (DTO)
├── services/
│   ├── __init__.py
│   └── ai_service.py        # AI 호출 비즈니스 로직
├── core/
│   ├── config.py            # 환경 변수 관리
│   └── dependencies.py      # 공통 Depends
├── .env
└── requirements.txt
```

---

## 9. 비동기 처리

FastAPI는 `async/await`를 네이티브로 지원한다.  
I/O 바운드 작업(DB 조회, 외부 API 호출)은 비동기로 처리하는 것이 권장된다.

```python
import asyncio
import httpx

@app.post("/chat")
async def chat(request: ChatRequest):
    async with httpx.AsyncClient() as client:
        # 비동기 외부 API 호출
        response = await client.post(
            "https://external-service.com/api",
            json={"message": request.message}
        )
    return response.json()
```

Gemini SDK의 경우 비동기 버전도 제공한다:

```python
model = genai.GenerativeModel("gemini-1.5-flash")
response = await model.generate_content_async(request.message)
```

---

## 10. 개념 대응 정리

| Spring Boot | FastAPI |
|-------------|---------|
| `@RestController` | `APIRouter` |
| `@GetMapping` / `@PostMapping` | `@router.get` / `@router.post` |
| `@PathVariable` | `Path(...)` |
| `@RequestParam` | `Query(...)` |
| `@RequestBody` + DTO | `BaseModel` (Pydantic) |
| `@Valid` / Bean Validation | `Field(...)` + `@validator` |
| `@Autowired` | `Depends(...)` |
| `@ExceptionHandler` | `@app.exception_handler` |
| `application.yml` | `.env` + `pydantic-settings` |
| Swagger (springdoc) | `/docs` (자동) |

---

## 참고

- [FastAPI 공식 문서](https://fastapi.tiangolo.com)
- [Pydantic v2 문서](https://docs.pydantic.dev)
- [uvicorn 공식 문서](https://www.uvicorn.org)
- [Starlette (FastAPI 기반 ASGI)](https://www.starlette.io)