---
title: "FastAPI + uvicorn 트러블슈팅 가이드"
date: 2026-04-28 22:00:00 +0900
categories: [AI 개발 실습]
tags: [fastapi, uvicorn, troubleshooting, python, cors, pydantic, debug]
---

## 개요

FastAPI + uvicorn 기반 AI 백엔드 개발 중 자주 마주치는 에러와 해결 방법을 정리한다.  
환경 설정 오류부터 CORS, 비동기 처리, Gemini API 한도 초과까지 실제 발생 빈도가 높은 케이스를 다룬다.

---

## 1. 환경 설정 오류

### 1-1. `ModuleNotFoundError: No module named 'fastapi'`

**원인:** 가상환경 미활성화 상태에서 실행

```bash
# 가상환경 활성화
source venv/bin/activate        # Mac/Linux
venv\Scripts\activate           # Windows

# 활성화 확인 (프롬프트에 (venv) 표시)
which python                    # Mac/Linux
where python                    # Windows
```

**예방:** `requirements.txt` + 가상환경을 프로젝트 단위로 관리

```bash
python -m venv venv
source venv/bin/activate
pip install -r requirements.txt
```

### 1-2. `uvicorn: command not found`

```bash
# 방법 1: python -m 으로 실행
python -m uvicorn main:app --reload

# 방법 2: 재설치
pip install "uvicorn[standard]"
```

`uvicorn[standard]` 는 `websockets`, `httptools` 등 성능 최적화 패키지를 포함한다.

---

## 2. 서버 실행 오류

### 2-1. `OSError: [Errno 48] Address already in use`

```bash
# 사용 중인 PID 확인
lsof -i :8000          # Mac/Linux
netstat -ano | findstr :8000   # Windows

# 프로세스 종료
kill -9 [PID]          # Mac/Linux
taskkill /PID [PID] /F # Windows

# 또는 포트 변경
uvicorn main:app --reload --port 8001
```

### 2-2. 코드 수정 후 미반영

`--reload` 옵션 누락이 원인인 경우가 대부분이다.

```bash
uvicorn main:app --reload
```

`--reload` 는 `watchfiles` 라이브러리를 통해 파일 변경을 감지한다.  
터미널에 `Reloading...` 메시지가 출력되면 정상 반영된 것이다.

---

## 3. 요청/응답 오류

### 3-1. `422 Unprocessable Entity`

Pydantic 유효성 검사 실패 시 발생한다. 응답 바디에 상세 원인이 포함된다.

```json
{
  "detail": [
    {
      "loc": ["body", "message"],
      "msg": "field required",
      "type": "value_error.missing"
    }
  ]
}
```

**주요 원인:**

| 원인 | 해결 |
|------|------|
| `Content-Type: application/json` 헤더 누락 | 헤더 추가 |
| 필수 필드 누락 | 요청 바디 확인 |
| 타입 불일치 (str vs int) | 필드 타입 확인 |
| 중첩 모델 형식 오류 | 전체 바디 구조 확인 |

**디버깅용 전역 핸들러 추가:**

```python
from fastapi.exceptions import RequestValidationError
from fastapi.responses import JSONResponse

@app.exception_handler(RequestValidationError)
async def validation_exception_handler(request, exc):
    return JSONResponse(
        status_code=422,
        content={
            "error": "요청 형식 오류",
            "detail": exc.errors(),
            "body": exc.body
        }
    )
```

### 3-2. `405 Method Not Allowed`

```python
# GET으로 정의했는데 POST로 호출한 경우
@app.get("/chat")   # GET
def chat(): ...

# POST /chat → 405 에러
```

`/docs` 에서 엔드포인트 메서드(GET/POST)를 먼저 확인한다.

---

## 4. CORS 오류

### 4-1. `Access-Control-Allow-Origin` 오류

브라우저에서 직접 API를 호출할 때 발생한다. ASGI 미들웨어로 해결한다.

```python
from fastapi.middleware.cors import CORSMiddleware

# 개발 환경
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# 프로덕션 환경 (출처 명시)
app.add_middleware(
    CORSMiddleware,
    allow_origins=[
        "https://yourdomain.com",
        "http://localhost:3000",
    ],
    allow_credentials=True,
    allow_methods=["GET", "POST", "DELETE"],
    allow_headers=["Content-Type", "Authorization"],
)
```

**주의:** `allow_credentials=True` 설정 시 `allow_origins=["*"]` 는 동작하지 않는다.  
크레덴셜을 허용할 때는 출처를 명시해야 한다.

---

## 5. 비동기 처리 오류

### 5-1. `RuntimeError: no running event loop`

```python
# 잘못된 패턴 — 동기 함수에서 await 사용
@app.post("/chat")
def chat(request: ChatRequest):
    result = await async_function()  # SyntaxError 또는 RuntimeError
    return result

# 올바른 패턴
@app.post("/chat")
async def chat(request: ChatRequest):
    result = await async_function()
    return result
```

### 5-2. 동기 블로킹 함수를 비동기 핸들러에서 호출

CPU 집약적이거나 블로킹 I/O 작업은 `run_in_executor`로 별도 스레드에서 실행한다.

```python
import asyncio
from fastapi import FastAPI

app = FastAPI()

def blocking_operation():
    # 시간이 오래 걸리는 동기 작업
    import time
    time.sleep(3)
    return "done"

@app.get("/slow")
async def slow_endpoint():
    loop = asyncio.get_event_loop()
    result = await loop.run_in_executor(None, blocking_operation)
    return {"result": result}
```

Gemini SDK의 동기 `generate_content()`를 비동기 핸들러에서 사용할 때도 동일하게 적용한다.  
또는 SDK의 비동기 버전인 `generate_content_async()`를 사용한다.

---

## 6. Gemini API 오류

### 6-1. `ResourceExhausted (429)` — 요청 한도 초과

```python
import google.api_core.exceptions as google_exceptions
import time

def safe_generate(model, prompt: str, max_retries: int = 3):
    for attempt in range(max_retries):
        try:
            return model.generate_content(prompt)
        except google_exceptions.ResourceExhausted:
            if attempt < max_retries - 1:
                wait_time = 2 ** attempt * 5  # 지수 백오프: 5, 10, 20초
                print(f"한도 초과. {wait_time}초 후 재시도...")
                time.sleep(wait_time)
            else:
                raise
```

### 6-2. `InvalidArgument` — 잘못된 요청

```python
except google_exceptions.InvalidArgument as e:
    # 주요 원인: 빈 문자열 프롬프트, 지원하지 않는 모델명
    raise HTTPException(status_code=400, detail=f"잘못된 요청: {e}")
```

### 6-3. `.env` API 키가 `None` 반환

```python
from dotenv import load_dotenv
import os

load_dotenv()  # genai.configure() 이전에 반드시 호출

api_key = os.getenv("GEMINI_API_KEY")
if not api_key:
    raise RuntimeError("GEMINI_API_KEY가 설정되지 않았습니다")
```

**.env 파일 작성 규칙:**
```bash
# 올바른 형식
GEMINI_API_KEY=AIzaSy...

# 잘못된 형식 (공백, 따옴표 사용 금지)
GEMINI_API_KEY = "AIzaSy..."
```

---

## 7. 디버깅 전략

### 로깅 설정

```python
import logging

logging.basicConfig(
    level=logging.DEBUG,
    format="%(asctime)s [%(levelname)s] %(name)s: %(message)s"
)
logger = logging.getLogger(__name__)

@app.post("/chat")
async def chat(request: ChatRequest):
    logger.debug(f"요청 수신: {request.message[:50]}")
    try:
        response = model.generate_content(request.message)
        logger.info(f"응답 생성 완료: {response.usage_metadata.total_token_count} tokens")
        return {"reply": response.text}
    except Exception as e:
        logger.error(f"오류 발생: {e}", exc_info=True)
        raise HTTPException(status_code=500, detail="서버 오류")
```

### 요청/응답 미들웨어

```python
import time
from fastapi import Request

@app.middleware("http")
async def log_requests(request: Request, call_next):
    start = time.time()
    response = await call_next(request)
    duration = time.time() - start
    logger.info(f"{request.method} {request.url.path} → {response.status_code} ({duration:.3f}s)")
    return response
```

---

## 에러 코드 빠른 참조표

| 상태 코드 | 원인 | 해결 |
|----------|------|------|
| 404 | 잘못된 URL 경로 | `/docs`에서 엔드포인트 확인 |
| 405 | 잘못된 HTTP 메서드 | GET/POST 메서드 확인 |
| 422 | 요청 바디 형식 오류 | Pydantic 모델과 요청 바디 비교 |
| 429 | Gemini API 한도 초과 | time.sleep() 또는 지수 백오프 |
| 500 | 서버 내부 오류 | 터미널 스택 트레이스 확인 |
| 503 | 외부 서비스 연결 실패 | Gemini API 상태 및 키 확인 |

---

## 참고

- [FastAPI 예외 처리 공식 문서](https://fastapi.tiangolo.com/tutorial/handling-errors/)
- [Pydantic 유효성 검사](https://docs.pydantic.dev/latest/concepts/validators/)
- [Google API Core 예외 목록](https://googleapis.dev/python/google-api-core/latest/exceptions.html)