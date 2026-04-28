---
title: "FastAPI + Gemini API로 멀티턴 챗봇 백엔드 구현"
date: 2026-04-27 22:00:00 +0900
categories: [AI 개발 실습]
tags: [fastapi, chatbot, backend, python, session, pydantic, uvicorn]
---

## 개요

이 글에서는 FastAPI와 Gemini API를 연동해 멀티턴 대화가 가능한 챗봇 백엔드를 구현한다.  
단순 단발성 응답에서 시작해 세션 기반 대화 맥락 유지, CORS 설정, 에러 핸들링까지 단계적으로 다룬다.

**최종 구현 스펙:**
- `POST /chat` — 메시지 전송 및 AI 응답
- `GET /sessions/{session_id}/history` — 대화 히스토리 조회
- `DELETE /sessions/{session_id}` — 세션 초기화
- 세션 기반 멀티턴 대화
- CORS 설정 (프론트엔드 연동 대비)

---

## 1. 프로젝트 구성

```
chatbot/
├── main.py
├── routers/
│   └── chat.py
├── models/
│   └── schemas.py
├── services/
│   └── ai_service.py
├── core/
│   └── config.py
├── .env
└── requirements.txt
```

**requirements.txt:**
```
fastapi==0.111.0
uvicorn[standard]==0.29.0
google-generativeai==0.7.2
python-dotenv==1.0.1
pydantic-settings==2.2.1
```

---

## 2. 환경 변수 관리

**core/config.py:**
```python
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    gemini_api_key: str
    gemini_model: str = "gemini-1.5-flash"
    system_instruction: str = "당신은 친절한 AI 어시스턴트입니다. 한국어로 답변하세요."
    session_ttl_minutes: int = 60

    class Config:
        env_file = ".env"

settings = Settings()
```

**.env:**
```
GEMINI_API_KEY=your_api_key_here
GEMINI_MODEL=gemini-1.5-flash
```

---

## 3. Pydantic 스키마

**models/schemas.py:**
```python
from pydantic import BaseModel, Field
from typing import Optional, List
from datetime import datetime

class ChatRequest(BaseModel):
    message: str = Field(..., min_length=1, max_length=2000)
    session_id: Optional[str] = Field(default=None, description="없으면 새 세션 생성")

class MessageHistory(BaseModel):
    role: str
    content: str

class ChatResponse(BaseModel):
    reply: str
    session_id: str
    tokens_used: int

class HistoryResponse(BaseModel):
    session_id: str
    messages: List[MessageHistory]
    total_turns: int
```

---

## 4. AI 서비스 레이어

**services/ai_service.py:**
```python
import google.generativeai as genai
from core.config import settings
from datetime import datetime, timedelta
from typing import Dict
import uuid

genai.configure(api_key=settings.gemini_api_key)

class SessionData:
    def __init__(self):
        self.chat = genai.GenerativeModel(
            model_name=settings.gemini_model,
            system_instruction=settings.system_instruction
        ).start_chat(history=[])
        self.created_at = datetime.now()
        self.last_accessed = datetime.now()

    def is_expired(self) -> bool:
        ttl = timedelta(minutes=settings.session_ttl_minutes)
        return datetime.now() - self.last_accessed > ttl

class AIService:
    def __init__(self):
        self._sessions: Dict[str, SessionData] = {}

    def _get_or_create_session(self, session_id: Optional[str] = None) -> tuple[str, SessionData]:
        # 만료 세션 정리
        self._cleanup_expired_sessions()

        if session_id and session_id in self._sessions:
            session = self._sessions[session_id]
            session.last_accessed = datetime.now()
            return session_id, session

        new_id = session_id or str(uuid.uuid4())
        self._sessions[new_id] = SessionData()
        return new_id, self._sessions[new_id]

    def _cleanup_expired_sessions(self):
        expired = [sid for sid, s in self._sessions.items() if s.is_expired()]
        for sid in expired:
            del self._sessions[sid]

    def chat(self, message: str, session_id: str = None) -> dict:
        sid, session = self._get_or_create_session(session_id)

        try:
            response = session.chat.send_message(message)
            return {
                "reply": response.text,
                "session_id": sid,
                "tokens_used": response.usage_metadata.total_token_count
            }
        except Exception as e:
            raise RuntimeError(f"AI 응답 생성 실패: {str(e)}")

    def get_history(self, session_id: str) -> list:
        if session_id not in self._sessions:
            return []
        history = self._sessions[session_id].chat.history
        return [
            {"role": turn.role, "content": turn.parts[0].text}
            for turn in history
        ]

    def delete_session(self, session_id: str) -> bool:
        if session_id in self._sessions:
            del self._sessions[session_id]
            return True
        return False

ai_service = AIService()
```

---

## 5. 라우터

**routers/chat.py:**
```python
from fastapi import APIRouter, HTTPException
from models.schemas import ChatRequest, ChatResponse, HistoryResponse
from services.ai_service import ai_service

router = APIRouter(prefix="/chat", tags=["chat"])

@router.post("", response_model=ChatResponse)
async def chat(request: ChatRequest):
    try:
        result = ai_service.chat(
            message=request.message,
            session_id=request.session_id
        )
        return ChatResponse(**result)
    except RuntimeError as e:
        raise HTTPException(status_code=503, detail=str(e))

@router.get("/sessions/{session_id}/history", response_model=HistoryResponse)
async def get_history(session_id: str):
    history = ai_service.get_history(session_id)
    return HistoryResponse(
        session_id=session_id,
        messages=history,
        total_turns=len(history) // 2
    )

@router.delete("/sessions/{session_id}")
async def delete_session(session_id: str):
    deleted = ai_service.delete_session(session_id)
    if not deleted:
        raise HTTPException(status_code=404, detail="세션을 찾을 수 없습니다")
    return {"message": "세션이 삭제되었습니다", "session_id": session_id}
```

---

## 6. 앱 진입점 (CORS 포함)

**main.py:**
```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from routers.chat import router as chat_router

app = FastAPI(
    title="AI Chatbot API",
    description="Gemini 기반 멀티턴 챗봇 백엔드",
    version="1.0.0"
)

# CORS 설정 (프론트엔드 연동 대비)
app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:3000", "http://127.0.0.1:5500"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

app.include_router(chat_router)

@app.get("/health")
async def health():
    return {"status": "ok"}
```

---

## 7. API 동작 흐름

```
# 1. 첫 메시지 (session_id 없음 → 자동 생성)
POST /chat
{ "message": "안녕하세요" }
→ { "reply": "...", "session_id": "abc-123", "tokens_used": 45 }

# 2. 이어지는 메시지 (session_id 포함)
POST /chat
{ "message": "제 이름은 잠탱이예요", "session_id": "abc-123" }
→ { "reply": "반갑습니다, 잠탱이님!", "session_id": "abc-123", "tokens_used": 62 }

# 3. 맥락 확인
POST /chat
{ "message": "제 이름이 뭐라고요?", "session_id": "abc-123" }
→ { "reply": "잠탱이라고 하셨습니다.", "session_id": "abc-123", "tokens_used": 58 }

# 4. 히스토리 조회
GET /chat/sessions/abc-123/history
→ { "session_id": "abc-123", "messages": [...], "total_turns": 3 }

# 5. 세션 삭제
DELETE /chat/sessions/abc-123
→ { "message": "세션이 삭제되었습니다", "session_id": "abc-123" }
```

---

## 8. 실행 및 테스트

```bash
uvicorn main:app --reload
```

Swagger UI: `http://127.0.0.1:8000/docs`

curl 테스트:
```bash
curl -X POST "http://127.0.0.1:8000/chat" \
  -H "Content-Type: application/json" \
  -d '{"message": "FastAPI가 뭔가요?"}'
```

---

## 9. 한계 및 개선 방향

| 현재 구현 | 개선 방향 |
|----------|----------|
| 메모리 세션 저장 | Redis / DB 영속화 |
| 단일 프로세스 | uvicorn workers + 공유 세션 저장소 |
| 인증 없음 | API Key / JWT 인증 추가 |
| 동기 AI 호출 | `generate_content_async()` 비동기 전환 |
| 세션 만료 정책 단순 | LRU 캐시 또는 TTL 기반 세션 관리 |

---

## 참고

- [FastAPI 공식 문서](https://fastapi.tiangolo.com)
- [Gemini API Python SDK](https://ai.google.dev/gemini-api/docs/quickstart?lang=python)
- [pydantic-settings](https://docs.pydantic.dev/latest/concepts/pydantic_settings/)