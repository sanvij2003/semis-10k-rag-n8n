# Week 2 Evaluation Results (Oct 16, 2026)

## Runs
| Config | Hit@5 (/12 answerable) | False refusals (/12) | Refusal accuracy (/3) | Faithfulness | Avg exec time |
|---|---|---|---|---|---|
| Run 1: fixed, no rerank | 7 (58%) | 4 (33%) | 3/3 | 1.00 | 2.8 s |
| Run 2: recursive, no rerank | 7 (58%) | 5 (42%) | 3/3 (check table) | 1.00 | 4.1 s |
| Run #4: fixed + Cohere rerank (top 20 -> 5) | 9 (75%) | 3 (25%) | 3/3 (check table) | 1.00 | 11.3 s incl. 7 s throttle (~4.3 s real) |
| Run #3: fixed + rerank, no throttle | excluded: 5/15 tests failed on Cohere trial rate limit (10 calls/min) | | | | |
| Recursive + rerank | skipped (time; recursive showed no benefit in Run 2) | | | | |

How derived: n8n averages metrics over all 15 rows; unanswerable rows always score hit_at_5 = 0, so Hit@5 = (avg x 15) / 12. False refusals = (refused avg x 15) - 3.

## Run 1 per-question (fixed, no rerank)
| Q | Type | Result | Label |
|---|---|---|---|
| Q01 | fact | Refused | Retrieval miss (vocabulary: "manufactures/produce" vs "utilize foundries") |
| Q02 | fact | Answered | |
| Q03 | fact | Refused | Retrieval miss (Micron business-unit list, p143) |
| Q04-Q06 | fact/number | Answered | |
| Q07 | number | Answered "$8.1 billion" | Correct but rounded (MD&A text), likely Hit@5 miss: metric limitation |
| Q08-Q09 | number | Answered | |
| Q10 | multi | Answered (NVIDIA part only) | Partial answer |
| Q11 | multi | Refused | One-company crowding |
| Q12 | multi | Refused | One-company crowding |
| Q13-Q15 | unanswerable | Refused | Correct |

## Run #4 per-question (fixed + rerank) vs Run 1
| Q | Run 1 | Run #4 | Change |
|---|---|---|---|
| Q01 | Refused | Refused | Not fixed (vocabulary gap) |
| Q03 | Refused | Refused | Not fixed |
| Q07 | "$8.1 billion" (rounded) | "$8.091 billion" | More precise (table value) |
| Q10 | NVIDIA part only | Both NVIDIA and AMD | Partial -> full |
| Q11 | Refused | Partial: NVIDIA 42,000 given; says AMD's count isn't in the excerpts | Crowding partly fixed; AMD p19 chunk not retrieved |
| Q12 | Refused | Refused | Not fixed; lowest rerank score 0.570 |
| Q13-Q15 | Refused | Refused | Correct, 3/3 |
Cohere rerank scores: unanswerable 0.208 / 0.609 / 0.331; answerable 0.570-0.964. Q14 (NVIDIA stock price, 0.609) outscored answerable Q12 (0.570): no threshold separates them under reranking; the prompt refusal rule carried safety.

## Key findings
1. No hallucinations in any run. Every failure was a refusal, not a wrong answer. Faithfulness 1.00 is inflated: refusals score 1 by design (7 of 15 rows in Run 1).
2. Reranking improved Hit@5 from 58% to 75% (+17 points, +2 questions) and cut false refusals from 4 to 3, at ~+1.5 s latency and +16% prompt tokens. Directional only at n=12.
3. Recursive (structure-aware) chunking gave no measurable benefit vs fixed (Hit@5 equal, +1 false refusal), despite 14% more chunks.
4. Number questions did not struggle (hypothesis 3 rejected): page-split PDF parsing kept tables readable.
5. Score gate: in Run 1, unanswerable top_scores were 0.517-0.602 and all answerable were 0.660-0.775; a ~0.63 threshold would separate them on this set (0.058 gap, small sample). Gate left at 0.30 for all runs to keep runs comparable.

## Iterations log (chronological)
- Ingestion wrote to __default__ namespace twice (empty namespace field); fixed and re-ran.
- No page numbers until "Split Pages in PDF" was enabled (NVDA 155 -> 221 chunks).
- Pressing D deleted (not deactivated) a node; restored with Ctrl+Z.
- n8n Cloud workspace restarted (503) after the largest ingestion; Pinecone data unaffected.
- Micron FiscalYear typed as FY2025 instead of FY2026 (free-text field); left as known issue.
- Retrieval returned 0 items: namespace not set on the query node either.
- Page field is a flat key `loc.pageNumber`; first expression returned "page ?".
- If node type error (number vs string); fixed operator type.
- LLM 400 error: n8n "Use Responses API" default; Nebius serves Chat Completions only.
- Rules conflict: refusals still had an empty "Sources:"; fixed with regex in Final answer.
- Basic LLM Chain defaulted to "Connected Chat Trigger Node" (No prompt specified); set to Define below.
- Set Metrics: node name reference failed; used $json.text from input instead; empty metric row caused "Metric name missing".
- Cohere trial key 10 calls/min: Run 3 lost 5/15 tests; added 7 s Wait on the evaluation path only.

## Cleaning notes / limitations
- Every chunk carries "Table of Contents" and browser print headers (date, URL, page count).
- Dense-only retrieval; no hybrid keyword search.
- No true semantic chunker in n8n; recursive is the closest equivalent.
- 15 questions: differences of 1-2 questions are noise.
- Reference answers drafted by Claude from the filing text with page citations; Q06, Q10, Q12 cross-checked against source pages.
- Export check: Token Splitter has no chunkOverlap set, so the fixed strategy ran at 0 overlap (n8n default), not the planned 50. Docs corrected.
