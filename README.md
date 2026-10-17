# Semiconductor 10-K RAG (n8n, no-code)

A retrieval-augmented question-answering system over the latest 10-K filings of **NVIDIA, AMD, Micron and Intel**, built entirely in n8n. It answers investor questions with page-level citations and refuses when the filings don't contain the answer.

Built for The Gen Academy, *Mastering Agentic AI*, Week 2 (Track 1: no-code, Project 2: Financial Document Intelligence).

> My RAG app helps investors researching semiconductors answer business, risk, and financial questions from the latest 10-K filings of NVIDIA, AMD, Micron, and Intel, with cited answers, correct refusals on out-of-scope questions, and answers in seconds.

---

## Results (15-question evaluation set)

| Config | Hit@5 (of 12 answerable) | False refusals (of 12) | Refusal accuracy (of 3) | Faithfulness | Avg time |
|---|---|---|---|---|---|
| Fixed chunks, no rerank | 7 (58%) | 4 (33%) | 3/3 | 1.00 | 2.8 s |
| Recursive chunks, no rerank | 7 (58%) | 5 (42%) | 3/3 | 1.00 | 4.1 s |
| **Fixed chunks + Cohere rerank** | **9 (75%)** | **3 (25%)** | **3/3** | **1.00** | ~4.3 s* |

\*11.3 s measured, including a 7 s throttle added on the evaluation path to stay under the Cohere trial key's 10 calls/minute limit.

**Findings**
- **No hallucinations in any run.** Every failure was a refusal, not a wrong answer. Faithfulness of 1.00 is inflated, since refusals score as faithful by design.
- **Reranking helped:** Hit@5 rose from 58% to 75%, and two multi-company questions were fixed (one full answer instead of a partial one, one answer instead of a refusal). The cost was about 1.5 s of added latency and 16% more prompt tokens. This is directional only at n = 12.
- **Recursive (structure-aware) chunking gave no measurable benefit** over fixed 512-token chunks, despite producing 15% more chunks. (The two strategies also differ in overlap: 0 tokens for fixed vs 200 characters for recursive.)
- **Number questions did well:** page-split PDF parsing kept the financial tables readable.

Full per-question analysis and the iterations log: [`eval/results.md`](eval/results.md)

---

## Architecture

```
INGESTION (workflow: 10-k Ingestion)
Form upload (PDF + ticker + fiscal year)
  ├─ Default Data Loader (PDF, split pages, metadata) ─ Token Splitter 512, no overlap ─┐
  └─ Default Data Loader (PDF, split pages, metadata) ─ Recursive Character 2000/200 ─┤
                                                Embeddings: Qwen3-Embedding-8B (Nebius)
                                                Pinecone index semis-10k (4096 dims, cosine)
                                                namespaces: fixed (1,122) | recursive (1,287)

QUESTION ANSWERING (workflow: 10-K Q&A)
Chat trigger / Evaluation trigger (+7 s Wait)
  → Normalize question
  → Retrieve chunks (Pinecone, top 20) → Cohere rerank-v3.5 (top 5)
  → Build context (numbered excerpts with ticker + page)
  → Enough evidence? (score gate)
      ├─ yes → Answer (Gemma-3-27b-it, temp 0, cite-or-refuse prompt)
      └─ no  → Refuse
  → Final answer (strips empty "Sources:")
  → [evaluation only] Faithfulness judge (Qwen3-235B) → Set Outputs → Set Metrics
```

**Why a fixed chain and not an AI Agent:** an agent decides for itself whether to search, so it can skip retrieval and answer from memory. A fixed chain always retrieves and always exposes its chunks and scores, which makes faithfulness measurable.

---

## Tools used and why

| Tool | Role | Why |
|---|---|---|
| **n8n Cloud** | Orchestration | It's the no-code track's platform. Its built-in Evaluations feature ran all 15 test questions and recorded the scores without any code. |
| **SEC EDGAR** | Data source | The official, free source for 10-K filings, so every answer traces back to the filing itself. |
| **Nebius Token Factory** | AI model hosting | Hosts open models through an OpenAI-compatible API. The course credits made the project nearly free (about $0.0007 per question, including judging). |
| `Qwen/Qwen3-Embedding-8B` | Embeddings | Turns text into vectors for search (4,096 dimensions). Its long context easily fits 500-token chunks. |
| `google/gemma-3-27b-it` | Answer model | Cheap, and follows strict "cite or refuse" rules well at temperature 0. Answers in about 3 to 4 seconds. |
| `Qwen/Qwen3-235B-A22B-Instruct-2507` | Judge model | A larger model from a different family grades faithfulness, so the answer model isn't graded by a model with its own style. |
| **Pinecone** (serverless, cosine) | Vector database | The free tier hosts both chunking strategies in one index as separate namespaces, so the comparison is a single setting change. |
| **Cohere** `rerank-v3.5` | Reranker | Reorders the top 20 search results by relevance, which raised retrieval hits from 58% to 75%. It plugs into n8n's Pinecone node directly. |
| **GitHub** | Hosting | Holds the workflow exports, the evaluation set and the results, so anyone can import and rerun the system. |
| **Claude** | AI assistant | Helped design the architecture, write the prompts and n8n expressions, debug errors from screenshots, and draft the documentation. |

---

## Corpus
Four 10-K filings from SEC EDGAR, 620 pages in total. See [`docs/corpus.md`](docs/corpus.md) for the source URLs. The PDFs are not stored in this repo.

## Repository layout
```
workflows/   n8n workflow exports (10-K_Ingestion.json, 10-K_QA.json)
eval/        eval_questions.csv (15 questions with reference answers and evidence), results.md
docs/        corpus.md (EDGAR links, fiscal years, page counts)
```

## How to run it yourself
1. In n8n, go to **Workflows**, then **Import from File**, and import both JSON files from `workflows/`.
2. Create these credentials (no secrets are included in the exports):
   - **OpenAI-type credential named `Nebius`:** API key from Nebius Token Factory, Base URL `https://api.tokenfactory.nebius.com/v1`
   - **Pinecone:** API key
   - **Cohere:** API key
3. In Pinecone, create the index `semis-10k` with 4,096 dimensions and the cosine metric.
4. On every OpenAI Chat Model node, turn **Use Responses API** OFF. Nebius serves Chat Completions only.
5. Run the ingestion form once per filing.
6. Import `eval/eval_questions.csv` as an n8n data table and run the evaluation from the **Evaluations** tab.

## Limitations
- The fixed splitter ran with 0 token overlap (the n8n default; the planned 50 was never set). A fact that straddles a chunk boundary can be split across two chunks.
- Dense-only retrieval, with no hybrid keyword search. One question (NVIDIA's foundries) fails because the question's wording ("produce") doesn't match the filing's ("utilize foundries").
- n8n has no embedding-based semantic chunker. Recursive Character is the closest no-code equivalent.
- Browser-printed PDFs add print headers and "Table of Contents" text to every chunk.
- Micron chunks carry `fiscal_year: FY2025` because of a typo in the upload form (it should be FY2026). It isn't used in retrieval.
- 15 questions is a small sample: differences of one or two questions are noise.
- The Cohere trial key is rate-limited (10 calls/minute). Production use would need a paid key or batching.

## Next steps
- Hybrid search (dense plus keyword) to catch exact terms and figures
- Metadata filtering by ticker to fix multi-company crowding
- Stripping print headers before chunking
- Adding 10-Q filings for quarterly freshness
- A chatbot UI tab in the Week 1 Streamlit analyzer (bonus)

## How AI tools were used
Claude helped design the architecture, write the prompts and n8n expressions, debug errors from screenshots, and draft the reference answers from the filing text with page citations. The three highest-risk answers were cross-checked against the source pages. n8n's own AI features were not used to build the workflows.
