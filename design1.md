# Design Document: Credible — AI-Powered Financial News Verification System

**Version:** 1.1  
**Date:** February 15, 2026  
**Architecture Type:** Client–Server + Retrieval-Augmented Generation (RAG)  
**Related Documents:** [IDEA.md](./IDEA.md), [SOLUTION_OVERVIEW.md](./SOLUTION_OVERVIEW.md), [HOW_IT_WORKS.md](./HOW_IT_WORKS.md)

---

## 1. Goals and Non-Goals

### 1.1 Goals
- Verify finance/market claims using grounded context from real-time data + retrieved news snippets.
- Provide source attribution links alongside responses.
- Support US + India market queries (including `.NS` / `.BO` symbols).
- Provide both a single-shot verification flow and a multi-turn chat flow with persisted sessions.

### 1.2 Non-Goals (v1)
- No server-side verification of Firebase auth tokens (client passes `X-User-ID`; backend stores sessions by that value).
- No distributed vector DB or multi-node scaling.
- No deterministic, reproducible output guarantees (LLM output can vary).

---

## 2. High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                           FRONTEND                                  │
│   Static HTML + JS modules + Tailwind CSS                            │
│   Pages: frontend/index.html, frontend/chat.html, frontend/docs       │
│   JS: frontend/js/{api,auth,app,market,ui,utils}.js                   │
└─────────────────────────────┬───────────────────────────────────────┘
                              │ HTTPS/REST (JSON)
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                           BACKEND                                   │
│   FastAPI app: backend/app.py                                        │
│   Routes: backend/routes/*                                           │
│   RAG: backend/rag/*                                                 │
│   Utilities: backend/utils/*                                         │
└─────────────────────────────┬───────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    EXTERNAL DATA + SERVICES                          │
│   Gemini (LLM + embeddings)                                          │
│   AlphaVantage, Finnhub, NewsAPI, FMP, GNews, MarketAux, IndianAPI    │
│   DuckDuckGo Search + Firecrawl (deep scrape)                         │
│   yfinance (quotes/history/profile)                                   │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 3. Backend Design

### 3.1 App Composition

The FastAPI application is assembled in [backend/app.py](backend/app.py). It enables permissive CORS for development and registers routers:
- Verification: [backend/routes/verify.py](backend/routes/verify.py)
- Chat sessions: [backend/routes/chat.py](backend/routes/chat.py)
- Market data: [backend/routes/market.py](backend/routes/market.py)
- Price history: [backend/routes/history.py](backend/routes/history.py)
- Predictions/news via web search: [backend/routes/predictions.py](backend/routes/predictions.py)

### 3.2 Configuration

Configuration is loaded from `backend/.env` by [backend/config.py](backend/config.py).

Key settings:
- `GEMINI_API_KEY` for Gemini client
- `LLM_MODEL` (defaults to `gemini-3-flash-preview`)
- `EMBEDDING_MODEL` (defaults to `gemini-embedding-001`)
- Web scraping toggles and time budgets (`WEB_SCRAPE_*`, `DDG_TIMEOUT_SECONDS`)

On Windows/corporate TLS environments, [backend/config.py](backend/config.py) can inject OS trust store certificates via `truststore` when `TRUSTSTORE_ENABLED=true`.

### 3.3 Persistence

**Vector store (ChromaDB):**
- Implemented in [backend/rag/vector_store.py](backend/rag/vector_store.py)
- Persistent path: `./chroma_db` (relative to the backend process working directory)
- Collection: `news_chunks`
- Embeddings: Gemini embeddings via `client.models.embed_content(...)` with fallback model names

**Chat sessions (file-based JSON):**
- Implemented in [backend/routes/chat.py](backend/routes/chat.py)
- Stored under `data/` (relative to working directory)
- Per-user files: `data/sessions_{safe_uid}.json` where `safe_uid` is derived from `X-User-ID`

---

## 4. RAG Pipeline Design

The verification and chat answers share the same pipeline function: `run_rag_pipeline(query)` in [backend/rag/rag_pipeline.py](backend/rag/rag_pipeline.py).

### 4.1 Pipeline Stages (Current Implementation)

1) **Symbol/entity resolution**
- Attempts `search_symbol(query)` first.
- If unresolved, performs a short-timeout Gemini extraction to pull a candidate company/ticker string and retries symbol search.

2) **Structured data fetching**
If a symbol is present:
- Quote via AlphaVantage (`GLOBAL_QUOTE`)
- Company profile via yfinance
- Financial ratios via FMP
- Company news via Finnhub, plus additional feeds (FMP, AlphaVantage News, MarketAux)
- For `.NS` / `.BO`, also fetches direct IndianAPI stock details

If no symbol is present:
- Requires the query to be “financially relevant” using a keyword list; otherwise returns an “irrelevant to stock market” response.
- Fetches Finnhub general market news and query-based NewsAPI results.

3) **Ingestion into ChromaDB**
- Normalizes news items into text + metadata and stores them in `news_chunks`.
- IDs are md5 hashes of URL + text to reduce duplicates.

4) **Web augmentation (DuckDuckGo + Firecrawl)**
- Uses `search_web_content(query)` in [backend/utils/finance_api.py](backend/utils/finance_api.py).
- Domain targeting adapts to query context (crypto / India / global).
- Deep-scrapes the top `WEB_SCRAPE_MAX_URLS` results (default 3) in parallel via Firecrawl.

5) **Retrieval (hybrid search)**
- Uses `hybrid_search(query, top_k=10)`.
- Combines vector similarity with a lightweight keyword scoring pass.

6) **Generation (Gemini)**
- Prompts Gemini to return a strict JSON structure containing:
  - `answer` (Markdown)
  - `sentiment_score` (0–100)
  - `sentiment_label`

7) **Verifier pass**
- Calls `verify_response(...)` in [backend/rag/verifier.py](backend/rag/verifier.py).
- If inaccurate, returns a corrected answer and marks the output.

### 4.2 Concurrency Model

FastAPI endpoints use `run_in_threadpool(...)` to execute the synchronous RAG pipeline without blocking the event loop:
- [backend/routes/verify.py](backend/routes/verify.py)
- [backend/routes/chat.py](backend/routes/chat.py)

---

## 5. API Design (Current Surface)

### 5.1 Verification
`POST /verify`
- Request: `{ "query": "..." }`
- Response: `{ "answer": "<markdown>", "sources": [...], "meta": {...} }`

`meta` commonly includes:
- `query`, `symbol`, `sentiment_score`, `sentiment_label`, `is_accurate`
- `web`: `{ enabled, always_on, ran, web_results, deep_scraped, ... }`

### 5.2 Chat
All endpoints under `/chat`:
- `POST /chat/sessions`
- `GET /chat/sessions`
- `GET /chat/sessions/{session_id}`
- `POST /chat/sessions/{session_id}/message`
- `DELETE /chat/sessions/{session_id}`

The optional `X-User-ID` header scopes session storage to a per-user JSON file.

### 5.3 Market Data (yfinance-backed)
All endpoints under `/market`:
- `GET /market/summary?region=IN|US`
- `GET /market/crypto`
- `POST /market/watchlist` (JSON array of symbols)
- `GET /market/sectors`
- `GET /market/movers?type=gainers|losers|active`
- `GET /market/fixed-income`
- `GET /market/ticker`

### 5.4 History
`GET /history/{symbol}` returns ~1 month of `{date, close}`.

### 5.5 Predictions
Under `/predictions`:
- `GET /predictions/news?region=global|IN|US`
- `GET /predictions/markets`

---

## 6. Frontend Design

### 6.1 Pages
- `frontend/index.html`: verification UI (single query → response)
- `frontend/chat.html`: chat UI (multi-turn sessions)
- `frontend/documentation.html`: user-facing docs

### 6.2 JS Modules
- `frontend/js/api.js`: backend REST client
- `frontend/js/auth.js`: Firebase client auth
- `frontend/js/market.js`: market sidebar/widgets
- `frontend/js/ui.js`: rendering helpers

---

## 7. Operational Notes and Risks

### 7.1 Constraints
- Third-party API rate limits and quota exhaustion can reduce coverage.
- Firecrawl may return “Payment Required” (402); the client backs off and falls back to basic HTTP fetch.
- Local file/vector storage is suitable for demos and small deployments.

### 7.2 Production Hardening (Future)
- Validate identity server-side (Firebase Admin / JWT verification) instead of trusting `X-User-ID`.
- Restrict CORS origins and headers.
- Add structured logging + tracing.

---

**Document Status:** Updated to match repository implementation
