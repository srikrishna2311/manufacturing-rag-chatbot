# Interview Talking Points — Project 2

---

## The Project Story (30-second version)

> "I built an agentic RAG chatbot that answers questions from industrial maintenance manuals. The ingestion pipeline uses LangChain to load PDFs, chunk them, embed them with Azure OpenAI, and index them in Azure AI Search with hybrid BM25 plus vector retrieval. A LangGraph agent decides whether to retrieve from the manual or invoke a fallback for off-topic queries. I evaluated it with Ragas — faithfulness, answer relevancy, and context recall — and the whole thing is served via FastAPI, containerised with Docker, and running on AKS."

---

## On RAG vs Fine-tuning

**Q: Why RAG instead of fine-tuning for this use case?**
> "Fine-tuning bakes knowledge into model weights — it's expensive, requires a large labelled dataset, and the knowledge becomes stale as manuals update. RAG keeps the knowledge in an index that you can update independently of the model. For industrial manuals that get revised periodically, RAG is the right choice — you reindex the updated PDF and the system immediately reflects the change."

---

## On Hybrid Search

**Q: What is hybrid search and why did you use it?**
> "Azure AI Search runs BM25 keyword search and vector similarity search simultaneously, then fuses the results using Reciprocal Rank Fusion. BM25 is precise for specific terms — a part number, a torque spec — but fails when the query is worded differently from the document. Vector search handles semantic similarity but can surface plausible-sounding but wrong chunks. Together they consistently outperform either approach alone. This is exactly the pattern I implemented manually in my production RAG POC — Azure AI Search just handles the infrastructure."

---

## On LangGraph vs a simple chain

**Q: Why LangGraph instead of a simple LangChain RAG chain?**
> "A simple chain always retrieves and always responds — it can't decide to skip retrieval for a clearly off-topic query, or to call a second tool if the first returns empty results. LangGraph lets the LLM act as a router: it sees the query, decides which tool to call, gets the result back, and decides whether to call another tool or respond. For a real deployment where users might ask anything, that decision-making layer is important."

---

## On Ragas Metrics

**Q: How did you evaluate the RAG pipeline quality?**
> "I used Ragas with three metrics. Faithfulness measures whether the answer is grounded in the retrieved context — low faithfulness means hallucination. Answer relevancy measures whether the response addresses the actual question. Context recall measures whether retrieval surfaced the right chunks in the first place. The three metrics together tell you exactly where a failure is coming from: retrieval problem versus generation problem. I targeted above 0.80 faithfulness, 0.75 answer relevancy, and 0.70 context recall."

---

## On the agentic pattern

**Q: What do you mean by 'agentic'?**
> "In a standard RAG chain, every query follows the same fixed steps: retrieve, then respond. An agentic pattern means the LLM itself decides what to do next. In my LangGraph setup, the agent node sees the query and chooses to call the retriever tool, a fallback tool, or respond directly — and it can call multiple tools in sequence if needed. The key is the conditional edge in the graph: after every LLM response, I check whether it produced tool calls. If yes, execute the tools and return to the agent. If no, we're done. This loop is what makes it agentic."

---

## Connecting to your production experience

**Q: How does this connect to your real work?**
> "The RAG chatbot extends my existing hybrid search POC from my Gen AI work at Wipro. In that POC I implemented BM25 and vector retrieval manually for an annual report chatbot. This project puts a proper engineering shell around the same concept — LangChain for the pipeline, LangGraph for the agentic loop, Azure AI Search for the managed vector store, Ragas for quantitative evaluation, and a full CI/CD deployment on AKS. It demonstrates that I can take a proof-of-concept and operationalise it as a production-grade service."
