# LiteLLM OpenRouter Gateway for Cursor

<p align="center">

`cursor-openrouter-litellm-proxy` is a production-grade **OpenAI-compatible AI gateway for Cursor IDE** powered by **LiteLLM + OpenRouter**. This setup exposes a single AI gateway endpoint that automatically selects the best model, applies caching, and handles failover. The full smart-routing configuration is defined in `config.yaml`.

</p>

<p align="center">

![Docker](https://img.shields.io/badge/Docker-ready-blue)
![LiteLLM](https://img.shields.io/badge/LiteLLM-gateway-green)
![OpenRouter](https://img.shields.io/badge/OpenRouter-models-purple)
![Cursor](https://img.shields.io/badge/Cursor-compatible-orange)
![License](https://img.shields.io/badge/license-MIT-lightgrey)
![Maintained](https://img.shields.io/badge/maintained-yes-brightgreen)

</p>

---

## LiteLLM Smart AI Gateway for Cursor

**OpenAI-compatible gateway for Cursor**. smart routing over OpenRouter with caching, persistence, observability, and secure public ingress.

LiteLLM becomes a local **AI control plane** for Cursor:

- **One endpoint**: `/v1`
- **Three logical models**: `chat`, `coder`, `vision`
- **Backed by**: multiple upstream models, Redis cache, Postgres, Cloudflare Tunnel
- **Configurable via**: `config.yaml`

## Smart Routing Logic

The router uses cascading fallbacks defined in `config.yaml`:

### Chat

```shell
chat (glm-4.5-air)
  ↓
step_flash (step-3.5-flash)
  ↓
gemma27 (gemma-3-27b)
  ↓
llama70 (llama-3.3-70b)
  ↓
qwen_think (qwen3-235b-thinking)
```

### Coder

```shell
coder (qwen3-coder)
  ↓
gpt_oss (gpt-oss-20b)
  ↓
gemma27
  ↓
nemotron30 (nemotron-3-nano-30b)
```

### Vision

```shell
vision (nemotron-nano-12b-v2-vl)
  ↓
qwen_vl (qwen3-vl-30b)
```

---

## Quick Navigation

- [Cursor + OpenRouter Compatibility (Known Issues)](#cursor--openrouter-compatibility-known-issues)
- [Architecture](#architecture)
- [Features](#features)
- [Infrastructure Components](#infrastructure-components)
- [Project Structure](#project-structure)
- [Requirements](#requirements)
- [Environment Variables](#environment-variables)
- [Quick Start](#quick-start)
- [Stable Cloudflare Tunnel URL (Recommended)](#optional-recommended-stable-cloudflare-tunnel-url-permanent-domain)
- [Cursor Configuration](#cursor-configuration)
- [Example Request](#example-request)
- [Architecture Diagrams](#architecture-diagrams)
- [Performance Improvements](#performance-improvements)
- [Security](#security)
- [Scaling](#scaling)
- [Troubleshooting](#troubleshooting)
- [Future Improvements](#future-improvements)
- [Screenshots](#screenshots)
- [License](#license)
- [Contributing](#contributing)

---

## Cursor + OpenRouter Compatibility (Known Issues)

If you’re here because **OpenRouter models are flaky/unusable in Cursor**, you’re not alone. Depending on Cursor version and the specific model/provider behind OpenRouter, users have reported:

- **Request-shape errors** (e.g. missing `prompt` / `messages`)
- **Agent/tool-calling failures** with newer models due to evolving “reasoning” and function-calling request formats
- **“Override OpenAI Base URL” side-effects** where enabling a custom OpenAI-compatible base URL can cause **other models (including Cursor-provided models)** to fail until the override is disabled

Related threads and discussions:

- [Workaround to get OpenRouter models working in Cursor](https://forum.cursor.com/t/workaround-to-get-openrouter-models-working-in-cursor/123235)
- [Cursor is practically unusable with any new model through OpenRouter](https://forum.cursor.com/t/cursor-is-practically-unusable-with-any-new-model-through-openrouter/143979)
- [Cannot use any models if OpenAI base URL overridden](https://forum.cursor.com/t/cannot-use-any-models-if-openai-base-url-overridden/140266)
- [Use Cursor with Open-source LLMs](https://forum.cursor.com/t/use-cursor-with-open-source-llms/125686)
- [Include OpenRouter support within API Keys supported services](https://forum.cursor.com/t/include-openrouter-support-within-api-keys-supported-services/5347)
- Reddit: [Not able to use the custom OpenRouter model with Cursor](https://www.reddit.com/r/CursorAI/comments/1l0lnvs/not_able_to_use_the_custom_openrouter_model_with/)

### Recommended workaround (what this repo implements)

Use **LiteLLM as the OpenAI-compatible endpoint Cursor talks to**, and let LiteLLM talk to OpenRouter.

- **Cursor sees**: stable logical model names (`chat`, `coder`, `vision`)
- **LiteLLM handles**: routing, retries/cooldowns, provider abstraction, caching, and observability

**This approach tends to be more stable than pointing Cursor directly at OpenRouter, and it avoids many model-name and request-format edge cases discussed in the threads above.**

### Cursor settings tips

- **Prefer this gateway’s `/v1` endpoint** as Cursor’s OpenAI-compatible Base URL (your tunnel URL + `/v1`), not `https://openrouter.ai/api/v1`.
- **Disable models you aren’t using** in Cursor’s Models settings. Some Cursor builds behave better when only valid/active models remain enabled.
- If you must use OpenRouter directly in Cursor, some users report needing **non-default OpenRouter preset names** (i.e., not the vendor/model string) to avoid model-name conflicts in Cursor (see the workaround thread).

---

## Architecture

```mermaid
flowchart TB
    subgraph Client["Client Layer"]
        Cursor["Cursor IDE"]
    end

    subgraph Tunnel["Public Access"]
        CF["Cloudflare Tunnel"]
    end

    subgraph Gateway["LiteLLM Gateway"]
        L["LiteLLM Proxy"]
    end

    subgraph Cache["Cache Layer"]
        Redis["Redis Cache"]
    end

    subgraph Router["Smart Router"]
        SR["Latency-Based Router"]
    end

    subgraph Provider["Model Provider"]
        OR["OpenRouter"]
    end

    subgraph Models["Multiple AI Providers"]
        M1["GLM 4.5 Air"]
        M2["Qwen Coder"]
        M3["Nemotron Vision"]
        M4["Step Flash"]
        M5["Gemma 27B"]
        M6["Llama 70B"]
        M7["Qwen 235B"]
    end

    Cursor --> CF --> L --> Redis
    L --> SR --> OR
    OR --> M1 & M2 & M3 & M4 & M5 & M6 & M7
```

The gateway acts as an AI control plane between developer tools and multiple LLM providers.

## Features

### Smart Model Routing

Automatically routes prompts to the best model:

| Task Type  | Model    | Purpose                    |
| ---------- | -------- | -------------------------- |
| **Chat**   | `chat`   | Fast conversational models |
| **Coding** | `coder`  | Specialized coding models  |
| **Vision** | `vision` | Multimodal models          |

Cursor only sees stable logical models; the gateway can map them to **many upstream models** and change routing without breaking client config.

### Cascading Failover

If a provider fails or rate limits:

```shell
Primary Model → Fallback 1 → Fallback 2 → Fallback 3
```

This keeps requests flowing even under upstream outages, rate limits, or transient failures.

### Redis Prompt Cache

Reduces inference costs and improves latency:

- **60–80%** token reduction in practice
- Instant responses for repeated prompts
- Lower provider usage and bill

### Latency-Aware Routing

The router automatically prioritizes the fastest responding model (`routing_strategy: latency-based-routing`).

### Automatic Rate Protection

Protects against provider throttling with cooldown and retry logic.

### Logical Models (What Cursor Sees)

Cursor only sees three stable models:

| Model    | Use Case                  |
| -------- | ------------------------- |
| `chat`   | General conversation      |
| `coder`  | Code generation & editing |
| `vision` | Image understanding       |

Internally the gateway uses 15+ models with fallback chains.

## Infrastructure Components

| Service               | Purpose                     |
| --------------------- | --------------------------- |
| **LiteLLM**           | AI gateway and router       |
| **Postgres**          | Request tracking + metadata |
| **Redis**             | Prompt caching              |
| **Cloudflare Tunnel** | Public endpoint for Cursor  |
| **OpenRouter**        | Model provider              |

## Project Structure

```shell
project/
├── docker-compose.yml
├── config.yaml
├── .env
└── README.md
```

## Requirements

- Docker
- Docker Compose
- OpenRouter API key
- Cursor IDE

## Environment Variables

Create `.env`:

```env
LITELLM_PORT=4000
LITELLM_MASTER_KEY=sk-localmasterkey

OPENROUTER_API_KEY=your_openrouter_key

UI_USERNAME=admin
UI_PASSWORD=admin

POSTGRES_USER=litellm
POSTGRES_PASSWORD=litellm
POSTGRES_DB=litellm
DATABASE_URL=postgresql://litellm:litellm@postgres:5432/litellm?schema=public

REDIS_HOST=redis
REDIS_PORT=6379
REDIS_PASSWORD=
```

## Start the Stack

```bash
docker compose up -d
```

### Services started

- `litellm-proxy`
- `litellm-postgres`
- `litellm-redis`
- `litellm-cloudflare-tunnel`

## Get Public Endpoint

### Check the tunnel logs

```bash
docker logs litellm-cloudflare-tunnel
```

### You will see something like

```shell
https://example-name.trycloudflare.com
```

### Your LiteLLM API endpoint becomes

```shell
https://example-name.trycloudflare.com/v1
```

### Cursor configuration

- **Base URL**: `https://example-name.trycloudflare.com/v1`
- **API key**: `sk-localmasterkey`
- **Model**: `chat` / `coder` / `vision`

## Optional (Recommended): Stable Cloudflare Tunnel URL (Permanent Domain)

The default setup uses a random `trycloudflare.com` URL that can change.

For long-term use (and to avoid Cursor SSRF issues when pointing at non-public endpoints), create a **named Cloudflare Tunnel** + **DNS route** so Cursor always connects to the **same permanent domain**.

- **Goal**: `https://llm.yourdomain.com` → Cloudflare Tunnel → LiteLLM (Docker)

### 1) Login to Cloudflare (creates cert)

Run:

```powershell
& "C:\Program Files (x86)\cloudflared\cloudflared.exe" tunnel login
```

Complete the browser login and select your zone (e.g. `yourdomain.com`). This writes:

- `C:\Users\<user>\.cloudflared\cert.pem`

### 2) Create the tunnel

```powershell
& "C:\Program Files (x86)\cloudflared\cloudflared.exe" tunnel create litellm-tunnel
```

This writes tunnel credentials (for example):

- `C:\Users\<user>\.cloudflared\litellm-tunnel.json`

### 3) Create DNS route

```powershell
& "C:\Program Files (x86)\cloudflared\cloudflared.exe" tunnel route dns litellm-tunnel llm.yourdomain.com
```

This creates:

- `llm.yourdomain.com` → Cloudflare Tunnel

### 4) Create tunnel config

Create `config.yml` in `C:\Users\<user>\.cloudflared\config.yml` (or `~/.cloudflared/config.yml`):

```yaml
tunnel: litellm-tunnel
credentials-file: C:\Users\<user>\.cloudflared\litellm-tunnel.json

ingress:
  - hostname: llm.yourdomain.com
    service: http://localhost:4000
  - service: http_status:404
```

### 5) Test the tunnel (run on host)

```powershell
& "C:\Program Files (x86)\cloudflared\cloudflared.exe" tunnel run litellm-tunnel
```

Then test:

```bash
curl https://llm.yourdomain.com/v1/models -H "Authorization: Bearer sk-localmasterkey"
```

### 6) Auto-start the tunnel with Docker Compose

If you want the tunnel to start automatically with Docker, update your `cloudflared` service to use your named tunnel and mount the Cloudflare config.

1. Create a folder in this repo: `./.cloudflared/`
2. Copy these files into it:

- `cert.pem`
- `litellm-tunnel.json`
- `config.yml` (adjusted for Docker networking, see below)

1. Use this Docker-friendly `config.yml` (note the service points to `litellm:4000`):

```yaml
tunnel: litellm-tunnel
credentials-file: /etc/cloudflared/litellm-tunnel.json

ingress:
  - hostname: llm.yourdomain.com
    service: http://litellm:4000
  - service: http_status:404
```

1. Replace the `cloudflared` service in `docker-compose.yml` with:

```yaml
  cloudflared:
    image: cloudflare/cloudflared:latest
    container_name: litellm-cloudflare-tunnel
    command: tunnel --no-autoupdate run litellm-tunnel
    volumes:
      - ./.cloudflared:/etc/cloudflared
    depends_on:
      - litellm
    restart: unless-stopped
```

### 7) Cursor configuration (stable forever)

- **Base URL**: `https://llm.yourdomain.com/v1`
- **API key**: `sk-localmasterkey`
- **Model**: `chat` / `coder` / `vision`

## Test the Gateway

```bash
curl https://your-tunnel-url/v1/models \
  -H "Authorization: Bearer sk-localmasterkey"
```

### Expected response includes

- `chat`
- `coder`
- `vision`

## Cursor Configuration

1. Open **Cursor → Settings → Models**
2. Add a new OpenAI-compatible model provider
3. Configure:

- **Base URL:** `https://your-tunnel-url/v1`
- **API Key:** `sk-localmasterkey`
- **Model:** `chat` (or `coder` / `vision`)

## Example Request

```bash
curl https://your-tunnel-url/v1/chat/completions \
  -H "Authorization: Bearer sk-localmasterkey" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "chat",
    "messages": [{"role": "user", "content": "Explain transformers in AI"}]
  }'
```

## Architecture Diagrams

### High-Level Flow

```mermaid
flowchart LR
    Cursor["Cursor"] --> GW["Gateway"]
    GW --> Router["Router"]
    Router --> Best["Best Model"]
```

### Failover Architecture

```mermaid
flowchart TD
    Req["Request"] --> P1["Primary Model"]
    P1 -->|Success| Resp["Response"]
    P1 -->|Fail/Rate Limit| P2["Fallback 1"]
    P2 -->|Success| Resp
    P2 -->|Fail| P3["Fallback 2"]
    P3 -->|Success| Resp
    P3 -->|Fail| P4["Fallback 3"]
    P4 --> Resp
```

### Request Lifecycle

```mermaid
sequenceDiagram
    participant C as Cursor
    participant T as Cloudflare Tunnel
    participant L as LiteLLM
    participant R as Redis
    participant O as OpenRouter

    C->>T: Request
    T->>L: Forward
    L->>R: Check cache
    alt Cache hit
        R-->>L: Cached response
        L-->>C: Instant response
    else Cache miss
        L->>O: Forward to model
        O-->>L: Response
        L->>R: Store in cache
        L-->>C: Response
    end
```

### Container Infrastructure

```mermaid
flowchart TB
    subgraph Docker["Docker Compose Stack"]
        subgraph Network["litellm network"]
            L["litellm-proxy:4000"]
            P["litellm-postgres:5432"]
            R["litellm-redis:6379"]
            CF["litellm-cloudflare-tunnel"]
        end
    end

    L --> P
    L --> R
    CF --> L
```

## Performance Metrics

| Feature          | Impact                       |
| ---------------- | ---------------------------- |
| Redis cache      | 60–80% cost reduction        |
| Fallback routing | Near-zero provider failures  |
| Latency routing  | Faster responses             |
| Unified gateway  | Simplified developer tooling |

## Security

The gateway uses:

- API key authentication
- Redis cache isolation
- Postgres persistence
- Private Docker network
- Cloudflare encrypted tunnel

## Scaling

For production environments:

- Run LiteLLM behind Kubernetes
- Use managed Redis

## Troubleshooting

### 404 data-policy errors

Enable "Free model publication" at [https://openrouter.ai/settings/privacy](https://openrouter.ai/settings/privacy) if you get 404 data-policy errors.

### Models not visible

Check `/v1/models` with your API key.

### Cursor cannot connect

Verify tunnel endpoint:

```bash
docker logs litellm-cloudflare-tunnel
```

### Provider rate limits

Fallback routing will automatically switch models.

## Future Improvements

**Possible extensions:**

- Cost-aware routing
- Automatic prompt complexity routing (`smart_chat` model)
- Grafana AI observability dashboards
- Multi-provider failover
- Vector-database integration
- Observability (Prometheus metrics, Grafana dashboards, Latency and error-rate monitoring, Provider failure tracking and SLOs)
- Grafana dashboards

## License

MIT License

## Contributions

Contributions are welcome.

## Maintainer

- **Name**: Adnan Sattar  
- **Email**: `adnansattar09@gmail.com`  
- **GitHub**: [@AdnanSattar](https://github.com/AdnanSattar)  
- **LinkedIn**: [adnansattar09](https://linkedin.com/in/adnansattar09)
