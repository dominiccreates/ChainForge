# ChainForge AI Service

AI-powered document verification, proof-of-life analysis, humanitarian claim verification, and PII anonymization for the ChainForge humanitarian aid platform.

## Architecture

The AI Service sits between the ChainForge backend and external LLM/ML providers, providing a unified API for:

- **OCR Processing** — Identity document text extraction using Tesseract
- **Proof-of-Life Verification** — Face detection and liveness analysis via OpenCV
- **Humanitarian Claim Verification** — LLM-driven verification against Sphere Handbook criteria
- **PII Anonymization** — Privacy-preserving text sanitization before external processing
- **Fraud Detection** — Unsupervised anomaly detection on claim metadata

## Quick Start

```bash
pip install -r requirements.txt
python main.py
```

The service starts at `http://localhost:8000`. Interactive API docs are available at `/docs`.

## Environment Configuration

Copy `.env.example` to `.env` and configure at least one AI provider:

| Variable | Default | Description |
|---|---|---|
| `OPENAI_API_KEY` | — | OpenAI API key |
| `GROQ_API_KEY` | — | Groq API key |
| `OPENAI_MODEL` | `gpt-4o-mini` | Default OpenAI model |
| `GROQ_MODEL` | `llama-3.3-70b-versatile` | Default Groq model |
| `AI_DETERMINISTIC_MODE` | `false` | Stable responses for CI/testing |
| `TEST_PROVIDER_MODE` | `false` | Fixture-driven responses (no API keys needed) |
| `LLM_TIMEOUT_SECONDS` | `30` | Timeout for LLM API requests |
| `APP_ENV` | `development` | `development`, `staging`, `production`, or `test` |
| `LOG_LEVEL` | `INFO` | Logging verbosity |
| `REDIS_URL` | `redis://localhost:6379/0` | Redis connection for task queue |
| `BACKEND_WEBHOOK_URL` | `http://localhost:3001/ai/webhook` | Backend notification endpoint |

## Core Services

### Health & Discovery

| Endpoint | Description |
|---|---|
| `GET /health` | Service health status |
| `GET /health/dependencies` | Redis, provider, and filesystem probe |
| `GET /` | Service root with API links |

### OCR Processing

```
POST /ai/ocr
```

Upload an identity document image (JPEG, PNG, BMP, TIFF, WebP) and receive extracted fields with confidence scores.

```bash
curl -X POST http://localhost:8000/ai/ocr \
  -F "image=@document.jpg"
```

### Proof-of-Life Verification

```
POST /ai/proof-of-life
```

Analyze a selfie and optional burst frames for face detection and liveness signals (blink detection, head movement).

```json
{
  "selfie_image_base64": "<base64-image>",
  "burst_images_base64": ["<base64-image>"],
  "confidence_threshold": 0.65
}
```

### Humanitarian Claim Verification

```
POST /ai/humanitarian/verify
```

Evaluate aid claims against Sphere Handbook criteria using LLM providers with automatic fallback and circuit breaker protection.

```json
{
  "aid_claim": "Relief teams delivered hygiene kits to all registered households in Sector B.",
  "supporting_evidence": ["Distribution list #B-17"],
  "context_factors": {
    "security_status": "stable",
    "weather": "heavy_rain"
  },
  "provider_preference": "auto"
}
```

### PII Anonymization

```
POST /ai/anonymize
```

Detect and mask personal identifiers (names, locations, dates, emails, phone numbers, IDs) before forwarding text to external LLM services.

```json
{
  "text": "On 15 Jan 2025, Mary Johnson received aid in Maiduguri Camp."
}
```

### Fraud Detection

```
POST /v1/ai/fraud/detect
```

Analyze claim metadata batches using Local Outlier Factor and flag anomalous patterns.

## Versioned API

All routes are available under versioned and legacy paths during the transition period.

| Prefix | Status |
|---|---|
| `/v1/ai/...` | ✅ Canonical — all new development |
| `/ai/...` | ⏳ Legacy — 308 redirects to `/v1` |

## Deployment

### Docker (CPU)

```bash
docker compose up ai-service
```

### Docker (GPU)

```bash
docker compose --profile gpu up ai-service-gpu
```

### Dockerfile Targets

| Target | Base | Use Case |
|---|---|---|
| `development` | CUDA 12.1 | Dev with hot-reload |
| `production` | Python 3.10-slim | Production CPU |
| `production-gpu` | CUDA 12.1 | Production GPU |

### Kubernetes / Cloud

Set `APP_ENV=production` and configure `OPENAI_API_KEY` or `GROQ_API_KEY`. The service scales horizontally behind a load balancer; each instance manages its own circuit breaker state and Redis-backed task queue.

## Testing

```bash
# Run all tests
pytest -v

# Run with coverage
pytest --cov=. -v

# Run specific test suite
pytest tests/test_routes.py -v
```

Use `AI_DETERMINISTIC_MODE=true` for stable verification outputs in CI. Use `TEST_PROVIDER_MODE=true` when no API keys are available — responses are served from fixture files under `fixtures/`.

## Project Structure

```
app/ai-service/
├── main.py                   # FastAPI application entry point
├── config.py                 # Environment configuration
├── tasks.py                  # Celery background task processing
├── metrics.py                # Prometheus metrics collection
├── exceptions.py             # Shared error types
├── proof_of_life.py          # OpenCV face/liveness analysis
├── conftest.py               # Pytest fixtures and stubs
├── api/
│   ├── routes.py             # Legacy OCR route
│   └── v1/
│       ├── router.py         # Versioned API aggregator
│       ├── ocr.py
│       ├── inference.py
│       ├── proof_of_life.py
│       ├── anonymize.py
│       ├── humanitarian.py
│       ├── fraud.py
│       └── artifacts.py
├── schemas/
│   ├── ocr.py
│   ├── anonymization.py
│   ├── humanitarian.py
│   ├── fraud.py
│   └── errors.py
├── services/
│   ├── ocr.py                # Tesseract OCR pipeline
│   ├── preprocessing.py      # Image preprocessing (threshold, denoise)
│   ├── pii_scrubber.py       # PII detection and masking
│   ├── humanitarian_verification.py  # LLM verification with fallbacks
│   ├── humanitarian_prompt.py        # Sphere Handbook prompt templates
│   ├── fraud_detection.py            # LOF-based anomaly detection
│   ├── artifact_access.py           # Signed URL artifact serving
│   ├── circuit_breaker.py           # Provider circuit breaker pattern
│   └── test_provider.py             # Fixture-driven deterministic provider
├── fixtures/                 # Test fixture response files
├── tests/                    # Unit and integration tests
├── Dockerfile                # Multi-stage Docker build
├── docker-compose.yml        # Service orchestration
└── requirements.txt          # Python dependencies
```

## Contributing

See [CONTRIBUTING.md](../CONTRIBUTING.md) for development guidelines.
