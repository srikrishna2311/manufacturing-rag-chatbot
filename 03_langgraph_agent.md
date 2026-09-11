# Phase 3 — LangGraph Agentic RAG

## Goal
Build a LangGraph agent that decides whether to retrieve from the manual index or invoke a fallback — making the chatbot "agentic" rather than a simple linear chain.

> **Prerequisite:** Complete [Phase 2](02_azure_ai_search.md) — Azure AI Search index populated.

---

## What is the difference between LangChain and LangGraph?

**LangChain (linear chain):**
```
query → retrieve → LLM → answer
```
Every query follows the same fixed path. No decisions.

**LangGraph (agent graph):**
```
query → LLM decides which tool to call
         ├── retrieve → LLM uses context → answer
         └── fallback → LLM says "not in manuals" → answer
```
The LLM chooses what to do at each step. This is the "agentic" pattern.

For interviews: "LangGraph lets me define the flow as a graph of nodes. The LLM router node decides which tool to invoke, then control returns to the router to decide if another tool call is needed, or if it's ready to respond."

---

## How LangGraph works (5-minute mental model)

1. You define **nodes** — each node is a function (agent, tool, etc.)
2. You define **edges** — which node to go to next (conditional or fixed)
3. You define **state** — a dictionary that flows through the graph and accumulates results
4. The graph runs until it hits the `END` node

```
START → agent_node ──(tool_calls?)──→ tool_node → agent_node → ... → END
                   └──(no tools)────────────────────────────────────→ END
```

---

## Step 1 — Install LangGraph

```bash
pip install langgraph==0.1.19 langchain-core==0.2.10
```

---

## Step 2 — Create the Agent (`src/agent.py`)

```python
# src/agent.py
import os
from dotenv import load_dotenv
from typing import TypedDict, Annotated
import operator

from langchain_core.messages import HumanMessage, SystemMessage
from langchain_core.tools import tool
from langchain_openai import AzureChatOpenAI
from langchain_community.vectorstores import AzureSearch
from langchain_openai import AzureOpenAIEmbeddings
from langgraph.graph import StateGraph, END
from langgraph.prebuilt import ToolNode

load_dotenv()

# --- State: the shared dictionary flowing through the graph ---
class AgentState(TypedDict):
    messages: Annotated[list, operator.add]   # list grows as nodes add messages


# --- Tools the LLM can call ---

@tool
def retrieve_from_manual(query: str) -> str:
    """
    Search the maintenance manual knowledge base for relevant information.
    Use this for any question about equipment, maintenance, specifications, or procedures.
    """
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
    docs = vectorstore.hybrid_search(query, k=4)
    if not docs:
        return "No relevant information found in the maintenance manual."

    context = "\n\n---\n\n".join([doc.page_content for doc in docs])
    return context


@tool
def topic_not_in_manual(query: str) -> str:
    """
    Use this when the query is clearly unrelated to equipment maintenance,
    industrial operations, or the content of maintenance manuals.
    """
    return f"The maintenance manual does not cover: '{query}'. Please ask about equipment, maintenance procedures, or specifications."


tools = [retrieve_from_manual, topic_not_in_manual]


# --- LLM with tools bound ---
llm = AzureChatOpenAI(
    azure_deployment="gpt-4o",
    azure_endpoint=os.getenv("AZURE_OPENAI_ENDPOINT"),
    api_key=os.getenv("AZURE_OPENAI_KEY"),
    temperature=0,    # deterministic — important for factual Q&A
)
llm_with_tools = llm.bind_tools(tools)


# --- System prompt ---
SYSTEM_PROMPT = """You are a maintenance engineer assistant. You help technicians and engineers 
by answering questions about equipment maintenance, installation, and troubleshooting.

Always retrieve relevant information from the maintenance manual before answering.
Base your answers strictly on the retrieved context. If the context doesn't contain 
the answer, say so clearly — do not guess or hallucinate information.

Be concise and technical. Include relevant specifications, steps, or warnings from the manual."""


# --- Graph nodes ---

def agent_node(state: AgentState) -> dict:
    """The LLM decides whether to call a tool or respond directly."""
    messages = [SystemMessage(content=SYSTEM_PROMPT)] + state["messages"]
    response = llm_with_tools.invoke(messages)
    return {"messages": [response]}


def should_continue(state: AgentState) -> str:
    """Router: check if the last message has tool calls."""
    last_message = state["messages"][-1]
    if last_message.tool_calls:
        return "tools"    # go to tool node
    return END            # done — return answer


# --- Build the graph ---

graph_builder = StateGraph(AgentState)

graph_builder.add_node("agent", agent_node)
graph_builder.add_node("tools", ToolNode(tools))

graph_builder.set_entry_point("agent")
graph_builder.add_conditional_edges("agent", should_continue)
graph_builder.add_edge("tools", "agent")   # after tools, go back to agent

rag_agent = graph_builder.compile()
```

---

## Step 3 — Test the Agent

Create a quick test script:

```python
# test_agent.py (temporary — not committed)
from src.agent import rag_agent
from langchain_core.messages import HumanMessage

def ask(question: str) -> str:
    result = rag_agent.invoke({
        "messages": [HumanMessage(content=question)]
    })
    return result["messages"][-1].content


# Test 1: should retrieve from manual
print("Q: What is the recommended oil change interval?")
print("A:", ask("What is the recommended oil change interval?"))
print()

# Test 2: should use fallback tool
print("Q: What is the weather today?")
print("A:", ask("What is the weather today?"))
```

Run:
```bash
python test_agent.py
```

---

## Checkpoint ✅

By the end of Phase 3 you should have:
- [ ] `src/agent.py` created with two tools and a compiled LangGraph
- [ ] Agent correctly retrieving from the manual for maintenance questions
- [ ] Agent correctly invoking the fallback for off-topic questions
- [ ] Responses grounded in retrieved context

**Next:** [Phase 4 — FastAPI Serving](04_fastapi_serving.md)
