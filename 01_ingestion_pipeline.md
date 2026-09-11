# Phase 1 — Document Ingestion Pipeline

## Goal
Load PDF maintenance manuals, split them into chunks, and prepare them for indexing into Azure AI Search.

---

## Key Concepts

**Why chunk documents?**
LLMs have a context window limit — you can't pass a 200-page manual into GPT-4o. Chunking breaks it into small, searchable pieces. At query time, only the 3-5 most relevant chunks are retrieved and sent to the LLM.

**Why 512 tokens with 50-token overlap?**
- 512 tokens ≈ one to two paragraphs — enough context per chunk to be meaningful
- 50-token overlap means consecutive chunks share a boundary — prevents answers being split across chunk edges

---

## Step 1 — Set Up Azure OpenAI

1. Go to [portal.azure.com](https://portal.azure.com)
2. Search → **Azure OpenAI** → Create
3. Fill in:
   - Resource group: `rag-chatbot-rg` (create new)
   - Name: `rag-chatbot-openai`
   - Region: **East US** (has the most model availability)
   - Pricing tier: Standard S0
4. Once created, go to the resource → **Model deployments** → **Manage deployments**
5. Deploy two models:
   - `gpt-4o` — for response generation
   - `text-embedding-3-small` — for embeddings

Note down:
- **Endpoint:** looks like `https://rag-chatbot-openai.openai.azure.com/`
- **API Key:** under Keys and Endpoint

---

## Step 2 — Get PDF Manuals

Option A (recommended to start): Download a free industrial manual PDF.
- Grundfos pump manual: search "Grundfos CM pump installation manual PDF"
- Any machinery service manual in PDF works

Place it in `data/manuals/pump_manual.pdf`

---

## Step 3 — Install Dependencies

```bash
pip install langchain==0.2.6 \
            langchain-openai==0.1.14 \
            langchain-community==0.2.6 \
            azure-search-documents==11.4.0 \
            azure-identity==1.17.0 \
            pypdf==4.2.0 \
            tiktoken==0.7.0
```

---

## Step 4 — Environment Variables

Create a `.env` file (never commit this to GitHub):

```bash
# .env
AZURE_OPENAI_ENDPOINT=https://rag-chatbot-openai.openai.azure.com/
AZURE_OPENAI_KEY=your_key_here
AZURE_SEARCH_ENDPOINT=https://rag-chatbot-search.search.windows.net
AZURE_SEARCH_KEY=your_search_key_here
AZURE_SEARCH_INDEX=maintenance-manuals
```

Create `.env.example` (this one IS committed — shows what vars are needed without values):

```bash
AZURE_OPENAI_ENDPOINT=
AZURE_OPENAI_KEY=
AZURE_SEARCH_ENDPOINT=
AZURE_SEARCH_KEY=
AZURE_SEARCH_INDEX=maintenance-manuals
```

Add `.env` to `.gitignore`:
```bash
echo ".env" >> .gitignore
```

---

## Step 5 — Ingestion Script (`src/ingest.py`)

```python
# src/ingest.py
import os
from dotenv import load_dotenv
from langchain_community.document_loaders import PyPDFLoader
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain_openai import AzureOpenAIEmbeddings

load_dotenv()

def load_and_chunk(pdf_path: str) -> list:
    """Load a PDF and split it into chunks."""
    print(f"Loading: {pdf_path}")
    loader = PyPDFLoader(pdf_path)
    pages  = loader.load()
    print(f"  Loaded {len(pages)} pages")

    splitter = RecursiveCharacterTextSplitter(
        chunk_size=512,
        chunk_overlap=50,
        separators=["\n\n", "\n", " ", ""]   # tries to split on paragraphs first
    )
    chunks = splitter.split_documents(pages)
    print(f"  Split into {len(chunks)} chunks")
    return chunks


def get_embeddings():
    """Return the Azure OpenAI embeddings model."""
    return AzureOpenAIEmbeddings(
        azure_deployment="text-embedding-3-small",
        azure_endpoint=os.getenv("AZURE_OPENAI_ENDPOINT"),
        api_key=os.getenv("AZURE_OPENAI_KEY"),
    )


if __name__ == "__main__":
    chunks = load_and_chunk("data/manuals/pump_manual.pdf")

    # Preview first chunk
    print("\n--- First chunk preview ---")
    print(chunks[0].page_content[:300])
    print(f"\nMetadata: {chunks[0].metadata}")
    print(f"\nTotal chunks ready to index: {len(chunks)}")
```

Run it:
```bash
python src/ingest.py
```

Expected output:
```
Loading: data/manuals/pump_manual.pdf
  Loaded 42 pages
  Split into 187 chunks

--- First chunk preview ---
Chapter 1: Installation
...
```

---

## Checkpoint ✅

By the end of Phase 1 you should have:
- [ ] Azure OpenAI resource created with GPT-4o and text-embedding-3-small deployed
- [ ] At least one PDF manual in `data/manuals/`
- [ ] `.env` configured with OpenAI credentials
- [ ] `src/ingest.py` running without errors — chunks loading and splitting correctly

**Next:** [Phase 2 — Azure AI Search Index](02_azure_ai_search.md)
