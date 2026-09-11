# Manufacturing Q&A RAG Chatbot — GenAI / LLMOps

An agentic RAG (Retrieval-Augmented Generation) chatbot that answers questions from industrial maintenance manuals. Built on Azure OpenAI, LangChain, LangGraph, and Azure AI Search, served via FastAPI, containerised with Docker, and deployed to AKS through a GitHub Actions CI/CD pipeline. Evaluated with Ragas.

## Architecture

```
PDF Manuals (source documents)
     ↓
LangChain document loaders + text splitter (512-token chunks)
     ↓
Azure OpenAI Embeddings (text-embedding-3-small)
     ↓
Azure AI Search — hybrid index (BM25 keyword + vector similarity, fused via RRF)
     ↓
LangGraph Agent
  ├── Tool 1: Retriever (queries Azure AI Search)
  └── Tool 2: Fallback (topic not in manuals)
     ↓
GPT-4o — grounded response generation
     ↓
FastAPI endpoint: POST /chat → answer + sources
     ↓
Ragas evaluation (faithfulness · answer relevancy · context recall)
     ↓
Docker → Azure Container Registry → AKS
```

## Tech Stack

| Layer | Tool |
|---|---|
| LLM | Azure OpenAI GPT-4o |
| Embeddings | Azure OpenAI text-embedding-3-small |
| Orchestration | LangChain + LangGraph |
| Vector/hybrid search | Azure AI Search |
| Serving | FastAPI + Uvicorn |
| Containerisation | Docker |
| Container registry | Azure Container Registry (ACR) |
| Orchestration | Azure Kubernetes Service (AKS) |
| CI/CD | GitHub Actions |
| RAG evaluation | Ragas |

## Dataset

Publicly available industrial equipment manuals in PDF format (pump, valve, compressor manuals).
Alternative: NASA Prognostics and Health Management dataset.

## Project Phases

| Phase | Doc | Status |
|---|---|---|
| 1 — Document ingestion | [docs/01_ingestion_pipeline.md](docs/01_ingestion_pipeline.md) | 🔲 |
| 2 — Azure AI Search index | [docs/02_azure_ai_search.md](docs/02_azure_ai_search.md) | 🔲 |
| 3 — LangGraph agent | [docs/03_langgraph_agent.md](docs/03_langgraph_agent.md) | 🔲 |
| 4 — FastAPI serving | [docs/04_fastapi_serving.md](docs/04_fastapi_serving.md) | 🔲 |
| 5 — Ragas evaluation | [docs/05_ragas_evaluation.md](docs/05_ragas_evaluation.md) | 🔲 |
| 6 — Docker + CI/CD + AKS | [docs/06_docker_cicd_aks.md](docs/06_docker_cicd_aks.md) | 🔲 |
| 7 — Interview notes | [docs/07_interview_talking_points.md](docs/07_interview_talking_points.md) | 🔲 |

## Repo Structure

```
manufacturing-rag-chatbot/
├── README.md
├── requirements.txt
├── .env.example                     ← environment variable template
├── docs/
│   ├── 01_ingestion_pipeline.md
│   ├── 02_azure_ai_search.md
│   ├── 03_langgraph_agent.md
│   ├── 04_fastapi_serving.md
│   ├── 05_ragas_evaluation.md
│   ├── 06_docker_cicd_aks.md
│   └── 07_interview_talking_points.md
├── data/
│   └── manuals/                     ← place your PDF manuals here
├── src/
│   ├── ingest.py                    ← ingestion + indexing pipeline
│   ├── agent.py                     ← LangGraph agent
│   ├── app.py                       ← FastAPI application
│   └── schemas.py                   ← Pydantic input/output models
├── evaluation/
│   └── ragas_eval.py                ← evaluation script
└── deployment/
    ├── Dockerfile
    ├── k8s-deployment.yaml
    └── k8s-service.yaml
```
