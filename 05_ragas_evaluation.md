# Phase 5 — RAG Evaluation with Ragas

## Goal
Evaluate the RAG pipeline quality using Ragas — the standard framework for measuring faithfulness, answer relevancy, and context recall.

> **Prerequisite:** Complete [Phase 4](04_fastapi_serving.md) — chatbot serving questions.

---

## Why Evaluate? (Interview-ready explanation)

LLM outputs are non-deterministic and hard to eyeball at scale. Ragas gives you **quantitative metrics** so you can:
- Detect when retrieval quality degrades (e.g. after reindexing)
- Compare the effect of chunking strategies or embedding model changes
- Know whether failures are a retrieval problem or a generation problem

---

## The Three Metrics You Must Know

| Metric | What it measures | Failure mode it catches |
|---|---|---|
| **Faithfulness** | Is the answer grounded in the retrieved context? | Hallucination — LLM adds facts not in context |
| **Answer Relevancy** | Does the answer address the question asked? | Drift — LLM answers a different question |
| **Context Recall** | Did the retriever surface the right chunks? | Retrieval failure — correct answer not in retrieved context |

**Key insight for interviews:** If faithfulness is high but context recall is low, the LLM is being honest about bad retrieval. If context recall is high but faithfulness is low, retrieval is fine but the LLM is hallucinating. The metrics tell you *where* to fix the problem.

---

## Step 1 — Install Ragas

```bash
pip install ragas==0.1.9 datasets==2.19.1
```

---

## Step 2 — Build an Evaluation Dataset

You need questions + expected answers (ground truth) + the retrieved context your system returned.

Create `evaluation/ragas_eval.py`:

```python
# evaluation/ragas_eval.py
import os
from dotenv import load_dotenv
from ragas import evaluate
from ragas.metrics import faithfulness, answer_relevancy, context_recall
from datasets import Dataset
from langchain_core.messages import HumanMessage
from langchain_community.vectorstores import AzureSearch
from langchain_openai import AzureOpenAIEmbeddings
from src.agent import rag_agent

load_dotenv()


# --- Step 1: Define your evaluation questions and ground truth ---
# These are questions you'd answer from reading the manual yourself
eval_questions = [
    "What is the recommended bearing lubrication interval?",
    "What should be checked before starting the pump for the first time?",
    "What are the torque specifications for the coupling bolts?",
    "How do you bleed air from the pump system?",
]

ground_truths = [
    "Bearings should be lubricated every 2000 operating hours or annually.",  # from your manual
    "Check shaft alignment, coupling condition, and ensure the pump is primed before first start.",
    "Coupling bolts should be torqued to 45 Nm in a cross pattern.",
    "Open the bleed valve on top of the pump casing until a steady stream of liquid flows out.",
]


# --- Step 2: Run questions through your RAG system ---
def run_rag_pipeline(question: str) -> tuple[str, list[str]]:
    """Returns (answer, list_of_retrieved_contexts)."""
    # Get answer from agent
    result = rag_agent.invoke({"messages": [HumanMessage(content=question)]})
    answer = result["messages"][-1].content

    # Re-run retrieval to get the contexts (for evaluation)
    embeddings = AzureOpenAIEmbeddings(
        azure_deployment="text-embedding-3-small",
        azure_endpoint=os.getenv("AZURE_OPENAI_ENDPOINT"),
        api_key=os.getenv("AZURE_OPENAI_KEY"),
    )
    vectorstore = AzureSearch(
        azure_search_endpoint=os.getenv("AZURE_SEARCH_ENDPOINT"),
        azure_search_key=os.getenv("AZURE_SEARCH_KEY"),
        index_name=os.getenv("AZURE_SEARCH_INDEX", "maintenance-manuals"),
        embedding_function=embeddings.embed_query,
    )
    docs = vectorstore.hybrid_search(question, k=4)
    contexts = [doc.page_content for doc in docs]

    return answer, contexts


# --- Step 3: Build the evaluation dataset ---
answers  = []
contexts = []

print("Running evaluation questions through RAG pipeline...")
for i, question in enumerate(eval_questions):
    print(f"  [{i+1}/{len(eval_questions)}] {question}")
    answer, ctx = run_rag_pipeline(question)
    answers.append(answer)
    contexts.append(ctx)
    print(f"    Answer: {answer[:100]}...")

eval_dataset = Dataset.from_dict({
    "question":     eval_questions,
    "answer":       answers,
    "contexts":     contexts,
    "ground_truth": ground_truths,
})


# --- Step 4: Run Ragas evaluation ---
print("\nRunning Ragas evaluation...")
results = evaluate(
    eval_dataset,
    metrics=[faithfulness, answer_relevancy, context_recall],
)

print("\n=== Ragas Evaluation Results ===")
print(f"Faithfulness:     {results['faithfulness']:.3f}   (target: > 0.80)")
print(f"Answer Relevancy: {results['answer_relevancy']:.3f}   (target: > 0.75)")
print(f"Context Recall:   {results['context_recall']:.3f}   (target: > 0.70)")
print("\nFull results dataframe:")
print(results.to_pandas())
```

---

## Step 3 — Run Evaluation

```bash
python evaluation/ragas_eval.py
```

Expected output:
```
=== Ragas Evaluation Results ===
Faithfulness:     0.87   (target: > 0.80)
Answer Relevancy: 0.81   (target: > 0.75)
Context Recall:   0.74   (target: > 0.70)
```

---

## Step 4 — Interpreting Results

| Score | What to do |
|---|---|
| Faithfulness < 0.75 | LLM is hallucinating — tighten the system prompt, add "only use retrieved context" instructions |
| Answer Relevancy < 0.70 | Agent is not answering the question — review tool descriptions and system prompt |
| Context Recall < 0.65 | Retrieval is failing — try smaller chunks, more overlap, or different k value |

---

## Checkpoint ✅

- [ ] `evaluation/ragas_eval.py` created and running
- [ ] All 3 Ragas metrics computed and above target thresholds
- [ ] You can explain each metric and what low scores mean

**Next:** [Phase 6 — Docker + CI/CD + AKS](06_docker_cicd_aks.md)
