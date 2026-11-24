# Google AI Overview Scraper Service

A high-performance, commercial-grade Google AI Overview scraper service designed to compete with Apify's \$3/1k pricing model (targeting ~\$0.85/1k + pass-through user proxy costs).

Built in Python using **direct HTTP/2 BoringSSL TLS/JA3/JA4 emulation (`curl_cffi`)**, multi-tier resilient DOM/JSON parsing, thread-safe proxy pooling, token-bucket rate limiting, and a FastAPI REST service.

---

## Architecture Overview

```
google-ai/
├── app/
│   ├── api/                     # FastAPI REST API & Routing
│   │   ├── __init__.py
│   │   └── routes.py            # /health, /v1/scrape, /v1/scrape/batch, /v1/jobs/{id}
│   ├── config.py                # Environment & Settings (Pydantic V2)
│   ├── main.py                  # Service entrypoint & lifespan manager
│   ├── models/                  # Pydantic Schemas & Types
│   │   ├── common.py            # Enums: BlockType, JobStatus, DeviceType
│   │   ├── request.py           # ScrapeRequest, BatchScrapeRequest
│   │   └── response.py          # AIOverviewData, SourceCitation, ScrapeResponse
│   ├── scraper/                 # BoringSSL Direct HTTP/2 Engine
│   │   ├── client.py            # Dual-stage extractor with /async/folsrch
│   │   ├── cookies.py           # Protobuf Google SOCS consent cookie generator
│   │   ├── detector.py          # Bot detection & CAPTCHA interception
│   │   └── headers.py           # Chrome 131 Client Hints & Fetch Metadata
│   ├── parser/                  # 5-Tier Resilient AI Overview Parser
│   │   ├── aio_parser.py        # Pipeline orchestrator
│   │   ├── base.py              # Strategy interfaces
│   │   ├── normalizer.py        # Text & Google /url?q= unquoting
│   │   ├── selectors.py         # 5-tier selector definitions
│   │   └── strategies.py        # Semantic, Anchor, SERP, Class & Script strategies
│   ├── proxy/                   # Proxy Pool & Health Management
│   │   ├── manager.py           # Round-robin, least-used, circuit-breaker
│   │   └── node.py              # Thread-safe proxy nodes (HTTP/SOCKS5)
│   └── concurrency/             # Concurrency & Batch Engine
│       ├── job_store.py         # Async in-memory batch job store
│       ├── rate_limiter.py      # Async Token Bucket rate limiter
│       └── worker_pool.py       # Semaphore-bounded worker pool
├── tests/                       # 293 Automated Tests
│   ├── e2e/                     # 4-tier E2E test harness (87 tests)
│   │   ├── test_tier1_features.py
│   │   ├── test_tier2_boundaries.py
│   │   ├── test_tier3_combinations.py
│   │   └── test_tier4_scenarios.py
│   ├── conftest.py              # Mock Google SERP fixtures & test app
│   ├── test_client.py           # Scraper engine tests (35 tests)
│   ├── test_models.py           # Pydantic schema tests (116 tests)
│   ├── test_parser.py           # Parser engine tests (44 tests)
│   └── test_proxy.py            # Proxy pool tests (11 tests)
├── requirements.txt
├── pyproject.toml
└── .env.example
```

---

## Key Features

1. **Direct HTTP/2 BoringSSL TLS Fingerprinting (`curl_cffi`)**:
   * Emulates authentic Google Chrome 131 JA3/JA4 TLS signatures and HTTP/2 settings frames without the massive compute/memory footprint of headless browsers.
2. **Dual-Stage Extraction**:
   * Captures initial search response and automatically handles Google's dynamic `/async/folsrch` endpoint for deferred AI Overview payloads.
3. **Protobuf SOCS Consent Cookie Engine**:
   * Uses binary protobuf varint serialization to generate valid Google `SOCS` consent cookies, preventing European and worldwide consent interstitals.
4. **5-Tier Resilient Extraction Pipeline**:
   * Falls back gracefully across Semantic attributes (`data-attrid`), Text Anchors, Structural SERP containers, Obfuscated classes, and Embedded script tags.
5. **Pass-Through & Managed Proxy Pool**:
   * Supports user-supplied HTTP/HTTPS/SOCKS5 proxies with automated health checks, sticky sessions, latency tracking, and cooldown on 429/banned nodes.
6. **Commercial Economics**:
   * Lightweight footprint allows executing thousands of queries per minute for ~\$0.85/1k platform cost plus pass-through proxy bandwidth.

---

## Quickstart

### 1. Install Dependencies
```bash
pip install -r requirements.txt
```

### 2. Run Tests
```bash
python -m pytest tests/
```
All **293 / 293 tests** pass in ~7 seconds.

### 3. Start the REST API Service
```bash
uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload
```

---

## API Endpoints

### `GET /health`
Returns service status, version, and proxy pool health.
```json
{
  "status": "healthy",
  "version": "0.1.0",
  "proxy_count": 0,
  "proxies_available": 0
}
```

### `POST /v1/scrape`
Synchronous single-query scrape.
```json
// Request
{
  "query": "what are the benefits of intermittent fasting",
  "language": "en",
  "country": "us",
  "device": "desktop",
  "proxy_url": "http://user:pass@proxy.example.com:8000" // optional
}

// Response
{
  "job_id": "00000000-0000-0000-0000-000000000000",
  "query": "what are the benefits of intermittent fasting",
  "ai_overview": {
    "is_present": true,
    "text": "Intermittent fasting has several health benefits...",
    "bullets": [
      "Weight loss and visceral fat reduction",
      "Improved insulin sensitivity",
      "Cellular repair processes (autophagy)"
    ],
    "markdown": "Intermittent fasting has several health benefits...",
    "sources": [
      {
        "index": 1,
        "title": "Healthline: 10 Evidence-Based Health Benefits of Intermittent Fasting",
        "url": "https://www.healthline.com/nutrition/10-health-benefits-of-intermittent-fasting",
        "domain": "healthline.com"
      }
    ]
  },
  "search_metadata": {
    "status_code": 200,
    "search_url": "https://www.google.com/search?q=what+are+the+benefits+of+intermittent+fasting&hl=en&gl=us",
    "timing_ms": 342
  }
}
```

### `POST /v1/scrape/batch`
Asynchronous batch processing for high-volume jobs (up to 1,000 queries per batch).
```json
// Request
{
  "queries": [
    "benefits of green tea",
    "how to train for a half marathon",
    "what is quantum computing"
  ]
}

// Response (HTTP 202)
{
  "job_id": "9b1deb4d-3b7d-4bad-9bdd-2b0d7b3dcb6d",
  "status": "pending",
  "total": 3,
  "message": "Batch scrape job queued successfully"
}
```

### `GET /v1/jobs/{job_id}`
Poll results of a batch scrape job.
