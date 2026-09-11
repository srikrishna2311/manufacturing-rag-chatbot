# Phase 2 — Azure AI Search (Hybrid Index)

## Goal
Create an Azure AI Search index and push all document chunks into it with embeddings, enabling hybrid search (BM25 keyword + vector similarity).

> **Prerequisite:** Complete [Phase 1](01_ingestion_pipeline.md) — chunks loading correctly, Azure OpenAI deployed.

---

## What is Hybrid Search? (Interview-ready explanation)

Azure AI Search runs **two searches simultaneously:**

1. **BM25 (keyword):** Finds documents containing the exact words in the query. Good for specific terms like "model XY-2000" or "valve seat replacement."
2. **Vector (semantic):** Converts the query to an embedding vector and finds documents with similar meaning, even if worded differently. Good for "how do I fix the pump" → finds "troubleshooting centrifugal pumps."

The results are merged using **Reciprocal Rank Fusion (RRF)** — each document gets a combined score based on its rank in both result lists. This reliably outperforms either approach alone.

> This is the same pattern you implemented manually in your Gen AI POC (BM25 + dense vector similarity). Here, Azure manages the infrastructure.

---

## Step 1 — Create Azure AI Search Resource

```bash
az search service create \
  --name rag-chatbot-search \
  --resource-group rag-chatbot-rg \
  --sku Basic \
  --location eastus

# Get the admin key
az search admin-key show \
  --service-name rag-chatbot-search \
  --resource-group rag-chatbot-rg
```

Copy the `primaryKey` value into your `.env` as `AZURE_SEARCH_KEY`.
Your `AZURE_SEARCH_ENDPOINT` = `https://rag-chatbot-search.search.windows.net`

---

## Step 2 — Update `src/ingest.py` to Push to Azure AI Search

Add this to your existing `ingest.py`:

```python
# Add to src/ingest.py
from langchain_community.vectorstores import AzureSearch

def create_vectorstore(chunks: list, embeddings) -> AzureSearch:
    """Create Azure AI Search index and push chunks with embeddings."""
    vectorstore = AzureSearch(
        azure_search_endpoint=os.getenv("AZURE_SEARCH_ENDPOINT"),
        azure_search_key=os.getenv("AZURE_SEARCH_KEY"),
        index_name=os.getenv("AZURE_SEARCH_INDEX", "maintenance-manuals"),
        embedding_function=embeddings.embed_query,
    )

    print(f"Indexing {len(chunks)} chunks into Azure AI Search...")
    vectorstore.add_documents(chunks)
    print("Indexing complete.")
    return vectorstore


if __name__ == "__main__":
    chunks     = load_and_chunk("data/manuals/pump_manual.pdf")
    embeddings = get_embeddings()
    vectorstore = create_vectorstore(chunks, embeddings)

    # Quick test search
    print("\n--- Test search ---")
    results = vectorstore.similarity_search("oil change interval", k=3)
    for i, r in enumerate(results):
        print(f"\nResult {i+1}:")
        print(r.page_content[:200])
```

Run:
```bash
python src/ingest.py
```

---

## Step 3 — Verify in Azure Portal

1. Go to [portal.azure.com](https://portal.azure.com)
2. Find your search service → **Indexes**
3. You should see `maintenance-manuals` with a document count matching your chunk count
4. Click the index → **Search Explorer** → type a query → verify results return

---

## Step 4 — Test Hybrid Search

Add a test to `src/ingest.py` to verify hybrid retrieval:

```python
# Test hybrid search (keyword + vector)
from langchain_community.vectorstores import AzureSearch
from azure.search.documents.models import VectorizedQuery

# Hybrid search returns results combining both BM25 and vector scores
results = vectorstore.hybrid_search("pump maintenance schedule", k=4)

print("\n--- Hybrid search results ---")
for i, doc in enumerate(results):
    print(f"\n[{i+1}] Page {doc.metadata.get('page', '?')}")
    print(doc.page_content[:250])
```

---

## Checkpoint ✅

By the end of Phase 2 you should have:
- [ ] Azure AI Search resource created: `rag-chatbot-search`
- [ ] Index `maintenance-manuals` populated with document chunks + embeddings
- [ ] Test search returning relevant results
- [ ] Hybrid search working (BM25 + vector)

**Next:** [Phase 3 — LangGraph Agent](03_langgraph_agent.md)
