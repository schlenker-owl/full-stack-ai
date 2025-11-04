# AI Service — FastAPI Microservice (Sanitized)

> **Scope:** Internal AI microservice that exposes a small, versioned REST surface.  
> **Design:** `Routers` (HTTP) → `Services` (business logic) → `Provider Adapters` (LLM/Embedding/Moderation APIs).  
> **Audience:** Backend/infra engineers, code agents.  
> **Security:** **Server-to-server only** via API key; no end-user tokens are accepted here.

---

## 1) What this service does (at a glance)

- Provides **text** and **media** endpoints (e.g., generate text, analyze/describe media, embed, moderate).  
- Normalizes request/response shapes so callers don’t care which upstream AI provider is used.  
- Adds pre/post-processing (validation, chunking, formatting, JSON contracts).  
- Tracks latency, token/cost (if available), and request IDs for observability.  
- Runs in a container on AWS (e.g., App Runner/ECS)—but is infra-agnostic by design.

---

## 2) Architecture

```mermaid
flowchart LR
  Caller["Backend API (trusted)"]
  G[Gate: API Key & Idempotency]
  R[Routers (/text, /media, /embed, /moderate)]
  S[Service Layer (validation, business rules, JSON contracts)]
  A[Provider Adapter (LLM/Embed/Moderate)]
  P[(External AI Provider)]

  Caller -->|HTTPS| G --> R --> S --> A --> P
  S -->|metrics/logs/traces| Obs[Observability]
````

**Key boundaries**

* **Routers:** HTTP input/output only. No provider calls here.
* **Services:** Orchestrate steps; construct prompts/payloads; enforce response contracts.
* **Adapters:** Small, swappable shims per AI provider; the *only* code allowed to touch SDKs or REST APIs.

---

## 3) HTTP surface (sanitized)

All endpoints are versioned under `/v1` and return JSON.

### 3.1 Text

* `POST /v1/text/generate` → `{ text, meta }`
* `POST /v1/text/chat` → `{ reply, meta }`
* `POST /v1/text/moderate` → `{ result, categories, meta }`

### 3.2 Media

* `POST /v1/media/analyze` → `{ width_px, height_px, duration_s?, meta }`
* `POST /v1/media/describe` → `{ summary, labels[], safety, meta }`

### 3.3 Vectors

* `POST /v1/embed/text` → `{ vector[], model, meta }`

> **Headers:**
> `X-API-Key: <server key>` (required) · `Idempotency-Key: <uuid>` (optional) · `traceparent: …` (optional)

---

## 4) Request/Response conventions

* **Requests** use typed JSON bodies validated by Pydantic models.
* **Responses** include `meta`: `{ request_id, latency_ms, provider, model, usage? }`.
* **Errors** are normalized:

  ```json
  { "error": { "type": "bad_request|unauthorized|upstream_error|timeout",
               "message": "human-readable", "retryable": false } }
  ```

---

## 5) Directory layout (pattern)

```
ai-service/
  app/
    main.py                # FastAPI app, CORS, health, exception handlers
    core/
      config.py            # env-driven settings (pydantic-settings)
      security.py          # API key verification, idempotency
      observability.py     # OpenTelemetry setup, logging
      errors.py            # ServiceError → HTTP mapping
    routers/
      text_router.py       # /v1/text/*
      media_router.py      # /v1/media/*
      embed_router.py      # /v1/embed/*
      moderate_router.py   # /v1/text/moderate (or /v1/moderate)
    services/
      text_service.py      # generate/chat pipelines
      media_service.py     # analyze/describe pipelines
      embed_service.py     # text embeddings
      moderate_service.py  # moderation pipeline
    providers/
      base.py              # abstract Provider interface(s)
      openai.py            # example concrete adapter (pattern)
      ...
    models/
      text.py              # Pydantic requests/responses
      media.py
      embed.py
      common.py            # Meta, Safety, Error schemas
    tests/
      …
```

---

## 6) Dependencies & configuration (sanitized)

Environment variables (examples—extend as needed):

```
PORT=8082
API_BASE_PATH=/v1
SERVER_API_KEY=********

# Provider selection & defaults
AI_PROVIDER=openai                   # adapter key (see Section 8)
AI_MODEL=general-1                   # default model id for text/chat
EMBED_MODEL=embed-1                  # default embedding model
AI_TIMEOUT_SECONDS=30
AI_MAX_TOKENS=512

# Observability
OTEL_EXPORTER_OTLP_ENDPOINT=https://otel-collector:4317
LOG_LEVEL=info

# Optional media processing (if used)
FFMPEG_BIN=ffmpeg
FFPROBE_BIN=ffprobe
```

**Notes**

* Secrets come from your secret manager in cloud; never commit real keys.
* Keep provider-specific env names next to each adapter (see Section 8).

---

## 7) Security model

* **Server-to-server only:** reject end-user tokens. Only your backend API calls this service.
* **API Key:** `X-API-Key` checked on every request (constant-time comparison).
* **Network:** run privately (VPC/VPC Connector / SG allowlist).
* **Idempotency:** honor `Idempotency-Key` for safe retries.
* **CORS:** disabled or restricted to backend origin(s) only.
* **PII:** do not log payloads; redact prompts unless debug is explicitly enabled in non-prod.

---

## 8) Provider adapters (major options)

> This service is adapter-based. Choose an adapter via `AI_PROVIDER=<key>`.
> Below is a **representative list of major providers & gateways** you can wire in.
> (Not exhaustive; you can add more by implementing the adapter interface.)

### 8.1 General LLM providers (chat/generate; many also support JSON-mode & tools)

* `openai` — OpenAI (Chat/JSON/Images/Embeddings/Moderation)
* `anthropic` — Anthropic (Claude; chat/generate, tool use)
* `google_vertex` — Google (Gemini via Vertex AI; chat/vision/embedding)
* `azure_openai` — Microsoft Azure OpenAI (OpenAI models via Azure endpoint)
* `meta_llama` — Meta Llama API (chat/generate; open-weights ecosystem)
* `xai` — xAI (Grok; chat/generate)
* `cohere` — Cohere (command/text/embeddings/rerank)
* `mistral` — Mistral (chat/generate/embeddings)
* `ibm_watsonx` — IBM watsonx.ai (LLMs & guardrails)
* `aws_bedrock` — Amazon Bedrock (multiple model families behind a unified API)

### 8.2 Hosted inference platforms & aggregators (route to many models; often lower latency or cost)

* `groq` — Groq Cloud (LPU-accelerated inference for select models)
* `together` — Together AI (hosted OSS & commercial models)
* `fireworks` — Fireworks.ai (hosted OSS/commercial models, fine-tuning)
* `openrouter` — OpenRouter (multi-provider gateway under one key)
* `replicate` — Replicate (hosted community models; async jobs common)
* `huggingface` — Hugging Face Inference Endpoints / TGI
* `octoai` — OctoAI (hosted OSS & vision models)
* `perplexity` — Perplexity API (RAG-style answering; chat)
* `databricks` — Databricks Model Serving / Mosaic AI
* `snowflake_cortex` — Snowflake Cortex LLMs (SQL/Python functions)
* `nvidia_nim` — NVIDIA NIM microservices (self/managed LLM/ASR/TTS endpoints)

### 8.3 Vision / image / video generation (pick as needed)

* `stability` — Stability AI (image generation/edits)
* `adobe_firefly` — Adobe Firefly API (image/text effects; license-aware)
* `google_vision_gen` — Google (Imagen family via Vertex)
* `luma` — Luma (video generation)
* `runway` — Runway (video gen; availability may vary)
* `pika` — Pika (video gen; availability may vary)
* `openai_images` — OpenAI Images (image generation/edits/variations)

### 8.4 Speech (ASR/TTS) & audio

* `openai_audio` — OpenAI (Whisper ASR, TTS)
* `deepgram` — Deepgram (ASR/TTS)
* `assemblyai` — AssemblyAI (ASR, audio intelligence)
* `google_speech` — Google Cloud Speech & TTS
* `azure_speech` — Azure Cognitive Services Speech (ASR/TTS)
* `aws_speech` — Amazon Transcribe (ASR) & Polly (TTS)
* `elevenlabs` — ElevenLabs (TTS)
* `playht` — PlayHT (TTS)
* `cartesia` — Cartesia (TTS/voice)

### 8.5 Embedding-first APIs & rerankers (useful when you only need vectors/rerank)

* `voyage` — Voyage AI (embeddings/rerank)
* `jina` — Jina AI (embeddings)
* `nomic` — Nomic (embeddings)
* (also supported via general providers: OpenAI, Cohere, Mistral, Google, Bedrock)

> **Tip:** Some adapters require extra settings (project/region/endpoint). Keep those adapter-local in `core/config.py` and document them inline with the adapter.

---

## 9) Provider env patterns (sanitized examples)

> Use the adapter key to choose which block applies. **Never** commit real keys.

```
# OpenAI
AI_PROVIDER=openai
OPENAI_API_KEY=***

# Anthropic
AI_PROVIDER=anthropic
ANTHROPIC_API_KEY=***

# Google (Vertex AI)
AI_PROVIDER=google_vertex
VERTEX_PROJECT_ID=your-project
VERTEX_LOCATION=us-central1
GOOGLE_APPLICATION_CREDENTIALS=/secrets/vertex-sa.json

# Azure OpenAI
AI_PROVIDER=azure_openai
AZURE_OPENAI_ENDPOINT=https://<name>.openai.azure.com/
AZURE_OPENAI_API_KEY=***

# AWS Bedrock
AI_PROVIDER=aws_bedrock
AWS_REGION=us-east-1
BEDROCK_MODEL_ID=anthropic.claude-3-...
# (use task role / instance profile for creds)

# Meta Llama API
AI_PROVIDER=meta_llama
LLAMA_API_KEY=***

# xAI
AI_PROVIDER=xai
XAI_API_KEY=***

# Cohere / Mistral
AI_PROVIDER=cohere
COHERE_API_KEY=***
# or
AI_PROVIDER=mistral
MISTRAL_API_KEY=***

# Groq / Together / Fireworks / OpenRouter
AI_PROVIDER=groq
GROQ_API_KEY=***
# (swap GROQ_… with TOGETHER_… / FIREWORKS_… / OPENROUTER_… respectively)

# Replicate / Hugging Face / OctoAI
AI_PROVIDER=replicate
REPLICATE_API_TOKEN=***
# HF/OctoAI use HUGGINGFACE_* / OCTOAI_* keys

# Speech (pick one)
AI_PROVIDER=deepgram
DEEPGRAM_API_KEY=***
# etc.
```

---

## 10) Router → Service → Adapter pattern (examples, sanitized)

### 10.1 Provider interface

```py
# providers/base.py
from typing import Protocol, Sequence, Mapping, Any

class TextProvider(Protocol):
    async def generate(self, prompt: str, *, max_tokens: int | None = None, json_mode: bool = False) -> dict: ...
    async def chat(self, messages: Sequence[Mapping[str, Any]], *, max_tokens: int | None = None, stream: bool = False) -> dict: ...

class EmbedProvider(Protocol):
    async def embed_text(self, text: str) -> list[float]: ...

class ModerateProvider(Protocol):
    async def moderate_text(self, text: str) -> dict: ...
```

### 10.2 Service uses provider

```py
# services/text_service.py (sanitized)
from models.text import GenerateRequest, GenerateResponse, Meta
from core.observability import with_metrics
from core.errors import ServiceError

class TextService:
    def __init__(self, provider):
        self.p = provider

    @with_metrics("text_generate")
    async def generate(self, req: GenerateRequest) -> GenerateResponse:
        try:
            out = await self.p.generate(req.prompt, max_tokens=req.max_tokens, json_mode=req.json_mode)
            return GenerateResponse(text=out["text"], meta=Meta.from_provider(out.get("meta", {})))
        except TimeoutError as e:
            raise ServiceError.timeout("upstream timeout") from e
        except Exception as e:
            raise ServiceError.upstream("provider error") from e
```

### 10.3 Router stays thin

```py
# routers/text_router.py (sanitized)
from fastapi import APIRouter, Depends
from core.security import verify_api_key
from services.text_service import TextService
from models.text import GenerateRequest, GenerateResponse
from di import get_text_service  # returns TextService with selected provider

router = APIRouter(prefix="/v1/text", tags=["text"])

@router.post("/generate", response_model=GenerateResponse, dependencies=[Depends(verify_api_key)])
async def generate_text(req: GenerateRequest, svc: TextService = Depends(get_text_service)):
    return await svc.generate(req)
```

---

## 11) JSON schemas (sanitized)

```py
# models/text.py
from pydantic import BaseModel

class GenerateRequest(BaseModel):
    prompt: str
    max_tokens: int | None = 512
    json_mode: bool = True

class Meta(BaseModel):
    request_id: str | None = None
    latency_ms: int | None = None
    provider: str | None = None
    model: str | None = None
    usage: dict | None = None  # tokens/cost if available

class GenerateResponse(BaseModel):
    text: str
    meta: Meta
```

```py
# models/media.py
class AnalyzeRequest(BaseModel):
    url: str | None = None
    s3_bucket: str | None = None
    s3_key: str | None = None

class AnalyzeResponse(BaseModel):
    width_px: int
    height_px: int
    duration_s: float | None = None
    meta: Meta
```

---

## 12) Observability

* **OpenTelemetry** tracing on inbound requests and all provider calls.
* **Metrics** (RED/USE): request rate, error counts, latency; per-endpoint histograms.
* **Logs**: structured JSON, correlation with `request_id`; prompt logging is redacted by default.

**Health**

* `GET /healthz` (liveness)
* `GET /readyz` (readiness; optionally checks provider availability with a short timeout)

---

## 13) Performance & reliability guidelines

* **Timeouts:** per-call upstream timeout (e.g., `AI_TIMEOUT_SECONDS`), plus overall request deadline.
* **Circuit breaker:** trip on consecutive upstream failures; serve fast failures until half-open.
* **Streaming:** if provider supports it, stream tokens to caller; otherwise return whole response.
* **Batching:** favor batch embedding where available to reduce overhead/cost.
* **Media:** if analyzing media, perform lightweight probes; treat thumbnails/poster frames as best-effort.

---

## 14) Local development

```bash
# 1) Configure env (safe defaults)
cp .env.example .env
# 2) Run
uvicorn app.main:app --host 0.0.0.0 --port 8082 --reload
# 3) Call it
curl -H "X-API-Key: dev" localhost:8082/v1/text/generate -d '{"prompt":"hello"}' -H "content-type: application/json"
```

---

## 15) Deployment notes (sanitized)

* Container exposes **PORT** (default 8082).
* Configure env/secret injection and private networking.
* Add health paths (`/healthz`, `/readyz`) to your orchestrator’s checks.
* Rotate API keys and provider credentials regularly; store secrets outside the image.

---

## 16) Acceptance checklist

* [ ] `GET /healthz` and `GET /readyz` return 200 without provider creds.
* [ ] `POST /v1/text/generate` returns `{text, meta}` with a valid provider key.
* [ ] `POST /v1/embed/text` returns a vector of expected length.
* [ ] All responses include `meta` with `latency_ms` and `provider`.
* [ ] Requests without `X-API-Key` get 401; with wrong key get 401/403.
* [ ] Traces show Router → Service → Adapter spans; error tags set on failures.

---

## 17) Security checklist (prod)

* [ ] Private networking (VPC/VPC Connector); CORS restricted; API key required.
* [ ] No PII in logs; prompts redacted unless explicit debug in non-prod.
* [ ] Per-endpoint timeouts + circuit breaker; exponential backoff on retries.
* [ ] Secrets in a manager (not env files checked into VCS); regular rotation.
* [ ] Least-priv IAM for any storage access (e.g., read-only for probes).

