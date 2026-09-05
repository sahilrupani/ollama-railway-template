# Deploy and Host Ollama on Railway — Private, OpenAI-Compatible LLM API

Ollama is an open-source server for running large language models locally. It exposes a simple REST API for generation, chat and embeddings on port 11434, along with an OpenAI-compatible endpoint at `/v1` so existing OpenAI SDK code works by swapping the base URL. This template deploys the official `ollama/ollama` image as a self-hosted, no-GPU-required LLM and embeddings API on Railway.

[![Deploy on Railway](https://railway.com/button.svg)](https://railway.com/deploy/ollama-llm-api?referralCode=zxcgoT&utm_medium=integration&utm_source=template&utm_campaign=generic)

## 🚀 Quick Start Deployment Guide

### Step 1: Deploy on Railway
1. Click **Deploy on Railway** above
2. Wait for the service to finish building

### Step 2: Mount a volume
1. Add a Railway Volume mounted at `/root/.ollama`
2. Model weights are downloaded once and cached there — without it every redeploy re-downloads gigabytes
3. Size it for the models you plan to pull

### Step 3: Restrict access before exposing it
1. Ollama ships **no authentication of its own**
2. Decide now whether the service gets a public domain at all — a private Railway network endpoint reachable only by your other services is the safer default
3. If you do expose it publicly, put an authenticating proxy in front of it

### Step 4: Pull your first model
1. Call `POST /api/pull` with a small model such as `llama3.2:1b`
2. Railway runs on CPU, so start small — a 1B–8B quantised model is realistic; a 70B model will not run
3. Wait for the pull to finish; it is written to the volume

### Step 5: Send your first request
1. Call `POST /api/generate` with the model name and a prompt
2. For OpenAI SDK clients, point the base URL at `https://<your-domain>/v1` instead
3. Expect CPU-speed latency — seconds per response, not milliseconds

```bash
# Pull a model once (stored on the volume)
curl -X POST https://<your-domain>/api/pull \
  -d '{"model": "llama3.2:1b"}'

# Generate a completion
curl -X POST https://<your-domain>/api/generate \
  -d '{
    "model": "llama3.2:1b",
    "prompt": "Why is the sky blue?",
    "stream": false
  }'
```

## About Hosting Ollama

This template runs the official `ollama/ollama` image as a single service on Railway, giving you a private inference server without managing servers or GPU infrastructure yourself. A volume at `/root/.ollama` caches downloaded model weights so they survive redeploys; `OLLAMA_MODELS` points at `/root/.ollama/models` and `OLLAMA_HOST` is pre-set to `0.0.0.0:11434`.

Two constraints matter. Railway instances are CPU-only, so this is suited to small quantised models, embeddings and batch workloads rather than fast chat with large models — memory use tracks model size and latency is measured in seconds. And Ollama ships no authentication whatsoever: a public domain on this service is an open LLM endpoint, so keep it on the private network or front it with an authenticating proxy.

If you need frontier-class models, faster throughput, or GPU-backed serving, consider the OpenAI API, Anthropic API, or vendor-hosted open-model providers like Together or Groq — all per-token, all without infrastructure to run.

## Common Use Cases

- **Private RAG embeddings**: a private embeddings endpoint for RAG pipelines, where documents must never leave your infrastructure
- **OpenAI SDK drop-in**: an OpenAI-compatible drop-in for apps that already speak the OpenAI SDK — change the base URL, keep the code
- **Flat-cost NLP tasks**: running small quantised models for classification, extraction and summarisation at flat cost
- **Dev/staging inference**: development and staging inference without burning production API credits

## Dependencies for Ollama Hosting

### Deployment Dependencies
- [Ollama (upstream source)](https://github.com/ollama/ollama)
- [Ollama API reference](https://github.com/ollama/ollama/blob/main/docs/api.md)
- [Ollama model library](https://ollama.com/library)

## ⚙️ Configuration

| Variable | Required | Description |
|---|---|---|
| `OLLAMA_HOST` | Yes | Bind address. Pre-set to `0.0.0.0:11434` so Railway's proxy can reach the server |
| `OLLAMA_MODELS` | Yes | Where model weights are stored. Pre-set to `/root/.ollama/models` — must sit inside the mounted volume |
| `OLLAMA_KEEP_ALIVE` | No | How long a model stays resident in memory after a request, e.g. `5m` or `24h`. Longer keeps latency low, at the cost of held RAM |
| `OLLAMA_MAX_LOADED_MODELS` | No | How many models may be resident at once. Each loaded model holds its full weight in RAM |
| `OLLAMA_NUM_PARALLEL` | No | Parallel request slots per model |

## 🐳 Self-Host with Docker Compose

```bash
git clone https://github.com/sahilrupani/ollama-railway-template
cd ollama-railway-template
cp .env.example .env
docker compose up -d
```

Once running, open `http://localhost:11434` (or the port defined in your compose file) to reach the API. Remember to mount a volume at `/root/.ollama` — the compose file handles this, but confirm it before pulling large models so weights persist across restarts.

## ❓ Frequently Asked Questions (FAQ)

### Does this need a GPU?
No — and it does not get one. Railway instances are CPU-only, which is why this template suits small quantised models, embeddings and batch work rather than low-latency chat with large models.

### Is it really OpenAI-compatible?
Ollama exposes an OpenAI-compatible surface at `/v1` alongside its native `/api` routes, so most OpenAI SDK clients work by changing the base URL and passing any placeholder key.

### How much does it cost to run Ollama on Railway?
Railway compute only — there are no per-token fees, since inference runs in your own container. Cost is driven by the RAM and CPU your model needs and by how long the service stays up, not by request volume.

### Is my data private?
Yes. Prompts and documents are processed inside your own container and never reach a model vendor. That is the main reason to run this instead of a hosted API.

### Which models should I pick?
Start with a 1B–8B quantised model such as `llama3.2:1b`. Memory use tracks the model file size, and CPU inference gets slow quickly as models grow.

### Can I put a chat UI in front of it?
Yes — Open WebUI is the usual companion. Point its `OLLAMA_BASE_URL` at this service's URL.

### How do I secure the endpoint?
Ollama has no authentication. Either keep the service on Railway's private network so only your other services reach it, or place an authenticating reverse proxy in front of the public domain.

### Why does every redeploy re-download the model, and startup takes forever?
No volume mounted at `/root/.ollama`, or `OLLAMA_MODELS` points outside it. Weights must live on the volume for them to persist across deploys.

### Why does the container run out of memory or get killed while loading a model?
The model is too large for the instance. A model needs roughly its file size in RAM; pick a smaller or more heavily quantised variant, or raise the plan.

### Why do responses take many seconds?
Railway instances are CPU-only. This is expected for CPU inference — use a smaller model, or a GPU-backed provider for latency-sensitive work.

### Why can anyone on the internet call my API?
Ollama has no built-in authentication. A public Railway domain on port 11434 is an open LLM endpoint that anyone can use at your expense. Keep it on the private network or front it with an authenticating proxy.

### Why does a generate call say "model not found"?
The model was never pulled on this deployment. Call `POST /api/pull` first — model names are case- and tag-sensitive, e.g. `llama3.2:1b`.

## 🛠️ Support & Issues

For bugs or questions about this template, open an issue at [github.com/sahilrupani/ollama-railway-template/issues](https://github.com/sahilrupani/ollama-railway-template/issues). Please include a description of the problem, steps to reproduce it, and relevant logs.

---

*This is a community-maintained Railway template built around [Ollama](https://github.com/ollama/ollama). It is not affiliated with, endorsed by, or supported by the Ollama project or Railway.*