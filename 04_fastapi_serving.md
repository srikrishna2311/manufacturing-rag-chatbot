# Phase 4 — FastAPI Serving Layer

## Goal
Wrap the LangGraph agent in a FastAPI application with `/health` and `/chat` endpoints.

> **Prerequisite:** Complete [Phase 3](03_langgraph_agent.md) — agent responding correctly.

---

## Step 1 — Schemas (`src/schemas.py`)

```python
# src/schemas.py
from pydantic import BaseModel

class ChatInput(BaseModel):
    question: str
    session_id: str = "default"   # for future: session memory per user

class ChatOutput(BaseModel):
    answer: str
    session_id: str
```

---

## Step 2 — FastAPI App (`src/app.py`)

```python
# src/app.py
from fastapi import FastAPI, HTTPException
from src.schemas import ChatInput, ChatOutput
from src.agent import rag_agent
from langchain_core.messages import HumanMessage
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

app = FastAPI(
    title="Manufacturing Maintenance Chatbot API",
    description="Answers questions from industrial maintenance manuals using RAG.",
    version="1.0.0"
)


@app.get("/health")
def health():
    """Liveness check for Kubernetes readinessProbe."""
    return {"status": "ok"}


@app.post("/chat", response_model=ChatOutput)
def chat(data: ChatInput):
    """
    Accepts a natural language question, returns a grounded answer from the manual.

    Example request:
    {
        "question": "What is the oil change interval for the centrifugal pump?",
        "session_id": "user-123"
    }
    """
    try:
        logger.info(f"Question received: {data.question}")

        result = rag_agent.invoke({
            "messages": [HumanMessage(content=data.question)]
        })

        answer = result["messages"][-1].content
        logger.info(f"Answer generated ({len(answer)} chars)")

        return ChatOutput(answer=answer, session_id=data.session_id)

    except Exception as e:
        logger.error(f"Error: {e}")
        raise HTTPException(status_code=500, detail=str(e))
```

---

## Step 3 — Requirements File

```
# requirements.txt
fastapi==0.111.0
uvicorn==0.29.0
langchain==0.2.6
langchain-openai==0.1.14
langchain-community==0.2.6
langgraph==0.1.19
langchain-core==0.2.10
azure-search-documents==11.4.0
azure-identity==1.17.0
pypdf==4.2.0
tiktoken==0.7.0
python-dotenv==1.0.1
pydantic==2.7.1
```

---

## Step 4 — Run and Test Locally

```bash
uvicorn src.app:app --reload --port 8000
```

Test with curl:
```bash
curl -X POST http://localhost:8000/chat \
  -H "Content-Type: application/json" \
  -d '{
    "question": "What are the maintenance intervals for the pump bearings?",
    "session_id": "test-session"
  }'
```

Open Swagger UI: http://localhost:8000/docs

---

## Checkpoint ✅

- [ ] `src/schemas.py` and `src/app.py` created
- [ ] Server running on port 8000
- [ ] `/chat` returning grounded answers
- [ ] Error handling in place (500 response on failures)

**Next:** [Phase 5 — Ragas Evaluation](05_ragas_evaluation.md)
