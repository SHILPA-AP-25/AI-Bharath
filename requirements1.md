# Requirements Document: Credible - AI-Powered Financial News Verification System

**Version:** 1.0  
**Date:** February 15, 2026  
**Project Type:** AI-Powered Financial Intelligence Platform

---

## 1. Executive Summary

Credible is an AI-powered financial news verification and analysis platform that combats misinformation by providing real-time, fact-checked insights grounded in verified data sources. The system leverages Retrieval-Augmented Generation (RAG) architecture, Google's Gemini LLM, and multiple financial data APIs to deliver trustworthy investment intelligence.

---

## 2. Business Requirements

### 2.1 Problem Statement
- Rapid spread of financial misinformation impacts investment decisions
- Users lack tools to quickly verify claims against credible sources
- Need for real-time, AI-powered financial analysis with source attribution
- Difficulty in aggregating data from multiple financial providers

### 2.2 Business Objectives
- Provide instant verification of financial news and claims
- Aggregate data from 10+ trusted financial APIs
- Deliver AI-generated insights with full source transparency
- Support both US and Indian stock markets
- Enable conversational follow-up questions through chat interface
- Maintain user history for personalized experience

### 2.3 Success Criteria
- Response time < 5 seconds for verification queries
- Source attribution for 100% of claims
- Support for 1000+ stock symbols (US + Indian markets)
- User authentication with session persistence
- Real-time market data updates

---

## 3. Functional Requirements

### 3.1 Core Features

#### FR-1: News Verification Engine
- **Priority:** Critical
- **Description:** Analyze user claims against verified news database
- **Acceptance Criteria:**
  - Accept natural language queries about stocks, markets, or financial events
  - Extract entities (stock symbols, companies) from queries
  - Return structured verdict with reasoning and sources
  - Support both specific stock queries and general market questions
  - Filter irrelevant non-financial queries

#### FR-2: RAG Pipeline
- **Priority:** Critical
- **Description:** Implement Retrieval-Augmented Generation for fact-checking
- **Components:**
  1. Query Analysis & Entity Extraction
  2. Real-time Data Fetching (10+ APIs)
  3. Vector Database Ingestion
  4. Hybrid Search (Semantic + Keyword)
  5. LLM Answer Generation
  6. Fact-Checking Verification Layer
- **Acceptance Criteria:**
  - Ingest news from multiple sources in real-time
  - Perform hybrid search with top-k=10 results
  - Generate responses in structured markdown format
  - Include sentiment analysis (0-100 scale)
  - Provide investment verdict (Buy/Hold/Sell)

#### FR-3: Interactive Chat Interface
- **Priority:** High
- **Description:** Conversational AI for follow-up questions
- **Acceptance Criteria:**
  - Create and manage chat sessions per user
  - Maintain conversation context across messages
  - Auto-generate session titles from first message
  - Support session deletion
  - Store chat history persistently

#### FR-4: Real-time Market Data
- **Priority:** High
- **Description:** Provide live market intelligence
- **Endpoints:**
  - Market summary (US/India indices)
  - Crypto leaderboard
  - Sector performance (11 US sectors)
  - Market movers (gainers/losers/active)
  - Fixed income ETFs
  - Global ticker tape
  - Watchlist tracking (1D, 5D, 1M, 6M changes)
- **Acceptance Criteria:**
  - Support region-specific data (US/IN)
  - Include sparkline data for visualizations
  - Calculate percentage changes accurately
  - Handle batch requests efficiently

#### FR-5: User Authentication & Personalization
- **Priority:** High
- **Description:** Secure user identity and preferences
- **Acceptance Criteria:**
  - Firebase Google Sign-In integration
  - Session persistence across page reloads
  - User-specific chat history storage
  - Protected routes (chat requires authentication)
  - User ID header propagation (X-User-ID)

#### FR-6: Multi-Source Data Aggregation
- **Priority:** Critical
- **Description:** Fetch and normalize data from multiple providers
- **Data Sources:**
  - **Stock Data:** Alpha Vantage, yFinance, Indian API
  - **News:** Finnhub, NewsAPI, GNews, FMP, MarketAux, Alpha Vantage News
  - **Web Scraping:** Firecrawl (parallel scraping), DuckDuckGo Search
  - **Market Data:** yFinance (real-time quotes, history)
- **Acceptance Criteria:**
  - Normalize data formats across sources
  - Handle API failures gracefully
  - Implement intelligent source selection based on query context
  - Support Indian market (.NS/.BO symbols)

#### FR-7: Web Content Scraping
- **Priority:** Medium
- **Description:** Deep scraping for comprehensive analysis
- **Features:**
  - Query-relevant domain selection (crypto/Indian/global)
  - Targeted search on priority domains
  - Parallel URL scraping (up to 5 URLs)
  - Full content extraction (15,000 char limit)
- **Acceptance Criteria:**
  - Detect query context (crypto/Indian/global)
  - Prioritize domain lists dynamically
  - Scrape content in parallel for speed
  - Append full content to summaries

### 3.2 API Endpoints

#### Verification API
```
POST /verify
Request: { "query": "string" }
Response: {
  "answer": "markdown string",
  "sources": [{ "title": "string", "meta": { "url": "string", "title": "string" } }],
  "meta": {
    "query": "string",
    "symbol": "string | null",
    "sentiment_score": 0-100,
    "sentiment_label": "Bearish|Neutral|Bullish",
    "is_accurate": boolean
  }
}
```

#### Chat API
```
POST /chat/sessions - Create new session
GET /chat/sessions - List all sessions
GET /chat/sessions/{id} - Get session details
POST /chat/sessions/{id}/message - Send message
DELETE /chat/sessions/{id} - Delete session
```

#### Market API
```
GET /market/summary?region=US|IN - Market indices
GET /market/crypto - Crypto leaderboard
GET /market/sectors - Sector performance
GET /market/movers?type=gainers|losers|active
GET /market/fixed-income - Fixed income ETFs
GET /market/ticker - Global ticker tape
POST /market/watchlist - Watchlist details
```

---

## 4. Non-Functional Requirements

### 4.1 Performance
- **NFR-1:** API response time < 5 seconds for 95% of requests
- **NFR-2:** Support 100 concurrent users
- **NFR-3:** Vector search latency < 500ms
- **NFR-4:** Parallel web scraping (5 URLs in < 10 seconds)

### 4.2 Scalability
- **NFR-5:** ChromaDB should handle 100,000+ document embeddings
- **NFR-6:** Support horizontal scaling of FastAPI backend
- **NFR-7:** Session storage should support 10,000+ users

### 4.3 Security
- **NFR-8:** API keys stored in environment variables only
- **NFR-9:** Firebase authentication for user identity
- **NFR-10:** CORS configured for frontend origin
- **NFR-11:** Sanitize user IDs for file storage
- **NFR-12:** No PII in logs or error messages

### 4.4 Reliability
- **NFR-13:** Graceful degradation when APIs fail
- **NFR-14:** Fallback mechanisms for symbol search (yFinance → AlphaVantage)
- **NFR-15:** Error logging for debugging
- **NFR-16:** Fact-checking layer to prevent hallucinations

### 4.5 Usability
- **NFR-17:** Responsive UI (mobile + desktop)
- **NFR-18:** Markdown rendering for formatted responses
- **NFR-19:** Loading indicators during API calls
- **NFR-20:** Clear error messages for users

### 4.6 Maintainability
- **NFR-21:** Modular architecture (routes, RAG, utils)
- **NFR-22:** Configuration via .env files
- **NFR-23:** Comprehensive logging
- **NFR-24:** Code documentation for key functions

---

## 5. Data Requirements

### 5.1 Vector Database (ChromaDB)
- **Storage:** Local file system (`chroma_db/`)
- **Schema:**
  ```json
  {
    "id": "md5_hash",
    "text": "headline + summary",
    "metadata": {
      "url": "string",
      "source": "string",
      "date": "string",
      "type": "news"
    }
  }
  ```
- **Embedding Model:** text-embedding-004 (Google)

### 5.2 Session Storage
- **Format:** JSON files per user
- **Location:** `backend/data/sessions_{user_id}.json`
- **Schema:**
  ```json
  {
    "session_id": {
      "id": "uuid",
      "title": "string",
      "created_at": timestamp,
      "updated_at": timestamp,
      "messages": [
        {
          "role": "user|assistant",
          "content": "string",
          "timestamp": float,
          "sources": [],
          "meta": {}
        }
      ]
    }
  }
  ```

### 5.3 API Keys Required
- GEMINI_API_KEY (Google AI)
- ALPHAVANTAGE_API_KEY
- FINNHUB_API_KEY
- NEWS_API_KEY
- FMP_API_KEY
- TWELVE_DATA_API_KEY
- GNEWS_API_KEY
- MARKET_AUX_API_KEY
- INDIAN_API_KEY
- FIRECRAWL_API_KEY

---

## 6. Integration Requirements

### 6.1 External APIs
- **Google Gemini:** LLM inference + embeddings
- **Alpha Vantage:** Symbol search, stock quotes, news sentiment
- **Finnhub:** Company news, market news
- **yFinance:** Real-time quotes, historical data, company profiles
- **NewsAPI:** Global headlines, Indian business news
- **GNews:** Global news aggregation
- **FMP:** Financial ratios, stock news
- **MarketAux:** Sentiment-tagged news
- **Indian API:** Direct BSE/NSE data
- **Firecrawl:** Web scraping
- **DuckDuckGo:** Web search

### 6.2 Frontend Integration
- **Firebase SDK:** Client-side authentication
- **REST API:** All backend communication via JSON
- **CORS:** Allow all origins in development

---

## 7. User Stories

### US-1: Verify Stock News
**As a** retail investor  
**I want to** verify if news about a stock is accurate  
**So that** I can make informed investment decisions

**Acceptance Criteria:**
- Enter stock symbol or company name
- Receive verdict with sources within 5 seconds
- See sentiment score and investment recommendation

### US-2: Ask Follow-up Questions
**As a** user  
**I want to** ask follow-up questions in a chat  
**So that** I can explore topics in depth

**Acceptance Criteria:**
- Create new chat session
- Send multiple messages with context retention
- View chat history

### US-3: Monitor Market Movers
**As a** day trader  
**I want to** see top gainers and losers  
**So that** I can identify trading opportunities

**Acceptance Criteria:**
- View top 5 gainers/losers
- See real-time price changes
- Filter by market (US/India)

### US-4: Track Watchlist
**As a** portfolio manager  
**I want to** track multiple stocks' performance  
**So that** I can monitor my investments

**Acceptance Criteria:**
- Submit list of symbols
- View 1D, 5D, 1M, 6M changes
- See current prices

### US-5: Verify Indian Stocks
**As an** Indian investor  
**I want to** verify news about NSE/BSE stocks  
**So that** I can trust my local market data

**Acceptance Criteria:**
- Support .NS and .BO symbols
- Fetch data from Indian API
- Include Indian finance news sources

---

## 8. Constraints & Assumptions

### 8.1 Constraints
- API rate limits (varies by provider)
- Free tier limitations on some APIs
- ChromaDB runs locally (not distributed)
- Session storage is file-based (not scalable to millions)

### 8.2 Assumptions
- Users have stable internet connection
- API keys are valid and active
- Python 3.8+ and Node.js are available
- Users understand basic financial terminology

---

## 9. Future Enhancements (Out of Scope for v1.0)

- Real-time WebSocket updates for market data
- Advanced charting and technical analysis
- Portfolio tracking and performance analytics
- Email/SMS alerts for price movements
- Mobile native apps (iOS/Android)
- Multi-language support
- Premium subscription tier
- Social features (share insights, follow analysts)
- Backtesting and paper trading
- Integration with brokerage APIs for live trading

---

## 10. Acceptance Criteria Summary

The system is considered complete when:
1. All FR-1 through FR-7 are implemented and tested
2. All API endpoints return correct responses
3. User authentication works end-to-end
4. RAG pipeline generates accurate, sourced responses
5. Market data endpoints provide real-time information
6. Chat sessions persist across page reloads
7. Indian market support is fully functional
8. Web scraping enhances response quality
9. Fact-checking layer prevents hallucinations
10. Documentation is complete (README, setup guide)

---

**Document Status:** Final  
**Approved By:** [Stakeholder Name]  
**Date:** February 15, 2026
