# Corpus

Latest annual reports (Form 10-K) from SEC EDGAR. Each main document was saved as a PDF from the plain HTML view (not the XBRL viewer), all pages. Exhibits are separate files on EDGAR and were not included.

| Ticker | Fiscal year | FY end | Filed | Pages | Source |
|---|---|---|---|---|---|
| NVDA | FY2026 | Jan 25, 2026 | Feb 25, 2026 | 113 | https://www.sec.gov/Archives/edgar/data/1045810/000104581026000021/nvda-20260125.htm |
| AMD | FY2025 | Dec 27, 2025 | | 149 | https://www.sec.gov/Archives/edgar/data/2488/000000248826000018/amd-20251227.htm |
| MU | FY2026 | Sep 3, 2026 | Oct 9, 2026 | 163 | https://www.sec.gov/Archives/edgar/data/723125/000072312526000023/mu-20260903.htm |
| INTC | FY2025 | Dec 27, 2025 | | 195 | https://www.sec.gov/Archives/edgar/data/50863/000005086326000011/intc-20251227.htm |

Total: 620 pages.

## Chunk counts (Pinecone index `semis-10k`)
| Ticker | `fixed` (Token 512/50) | `recursive` (2000 chars/200) |
|---|---|---|
| NVDA | 221 | 252 |
| AMD | 263 | 310 |
| INTC + MU | 638 | 725 |
| **Total** | **1,122** | **1,287** |

Fiscal periods differ by company, so every evaluation question names the fiscal year.
