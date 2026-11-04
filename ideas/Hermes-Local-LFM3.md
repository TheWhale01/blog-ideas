---
title: Hermes on local LFM3-1.7B
created: 2026-10-26
updated: 2026-10-26
type: idea
tags: [erebos, IT, AI, homelab]
sources: [https://huggingface.co/LiquidAI/LFM3-1.7B]
---

# Hermes on local LFM3-1.7B

**Status**: unimplemented

## Goal

Move Odin (the Hermes Agent service on Erebos) from the current **OpenRouter cloud model** (`sys/services/hermes.nix` → `model.provider = "openrouter"`, `default = "qwen/qwen3.8-flash"`) to a **local custom provider** running **[LFM3-1.7B](https://huggingface.co/LiquidAI/LFM3-1.7B)** — full privacy, zero token cost, no external egress for everything the agent reads and writes (this vault, secrets paths, service configs…).

## Why LFM3-1.7B specifically

Released **2026-10-27** by Liquid AI (announced alongside this audit's own timeline — fresh model, day-0 support). Key fits:

- **Post-trained for agentic harnesses explicitly including Hermes Agent** (per the [model card](https://huggingface.co/LiquidAI/LFM3-1.7B) and [Liquid's docs](https://docs.liquid.ai/lfm/models/lfm3-1.7b)) — native tool calling + structured outputs.
- 1.73B params, tri-hybrid conv/attention architecture → **fits ~1 GB at 4-bit**, 256K context, "fastest model on device" benchmarks even on **CPU-only servers** (Xeon) and cheap KV cache: 128K context ≈ 1.6 GB vs 4.3 GB for a comparable Qwen3.5-4B.
- Erebos feasibility is already proven: the box runs **Ollama with `qwen3.5:9b` and `translategemma:4b` today**, has **12 cores / 31 GiB (21 Gi available)** — a 1.7B model is far lighter than what's already running.

## Serving options

1. **Ollama** (already deployed, loopback :11434): [`lfm3`](https://ollama.com/library/lfm3) is in the library with tags `lfm3:1.7b` (default quant), `-q4_`, `-q8_0`, `-mxfp8`, `-nvfp4`, `-bf16`, `-mlx-bf16`, `-qad-q4_0`. On CPU: `ollama pull lfm3:1.7b`.
2. **llama.cpp** (`llama-server --port 8081` exposing OpenAI-compatible `/v1/chat/completions`) — most control over quant/context; day-0 supported upstream.
3. vLLM/SGLang — GPU-oriented; irrelevant until Erebos gets that RX 7600XT ([[ideas/Remote-Gaming-Server]] hardware overlap!).

## Pointing Hermes at it

Hermes' [providers docs](https://hermes-agent.nousresearch.com/docs/integrations/providers) treat local/custom OpenAI-compatible endpoints as first-class (`provider: custom` + `base_url`). Interactive: `hermes model` → *Custom endpoint*, URL `http://127.0.0.1:11434/v1`, model `lfm3:1.7b`. On Erebos it's declarative in `sys/services/hermes.nix`:

```nix
settings = {
  model = {
    provider = "custom";
    base_url = "http://127.0.0.1:11434/v1";
    default = "lfm3:1.7b";
    context_length = 64000;   # must match the EFFECTIVE server-side context
  };
  matrix = { … };  # unchanged
};
```

**The classic local-model trap** (called out in the Hermes docs): context length is set **server-side**, not via the API — Ollama defaults to a tiny `num_ctx`; either set `OLLAMA_CONTEXT_LENGTH=64000` in the Ollama systemd env, or bake a Modelfile (`PARAMETER num_ctx 64000`), then set the same number in Hermes' `context_length`. Mismatch = truncated tool-call history and confusing behavior. Also expect slower turns than OpenRouter: raise timeout settings rather than letting long generations abort.

## Honest trade-offs

- The model card itself warns: **not recommended for agentic coding or complex multi-step reasoning**. Wiki ingestion, summarization, page maintenance, cron triage = its sweet spot; a big debug or multi-file edit will be *worse than qwen3.8-flash*.
- Suggested landing zone: **hybrid** — keep a cloud provider for the interactive agent (me, doing schema-sensitive wiki surgery), and run LFM3 for scheduled/background jobs (e.g. the future [[todo/scheduled-security-audit]] first-pass triage, morning briefings, Translaitor-style batch work) where free tokens × dozens of agents beat raw capability. Liquid explicitly pitches this ("tokens are free"). Alternatively full switch + keep OpenRouter as documented fallback config.
- License: **LFM Open License v1.0** — open-weights with commercial use reserved above $10M revenue; a personal server is clearly fine, worth knowing anyway.
- Benchmark first: `ollama run lfm3:1.7b` tok/s on 12 vCPU; then a real wiki-maintenance eval (one ingest end-to-end) before deciding.

## Tasks

- [ ] `ollama pull lfm3:1.7b`; measure tok/s and set effective context (env or Modelfile)
- [ ] Smoke-test tool calling via the OpenAI endpoint (`/v1/chat/completions` with a forced function call)
- [ ] Eval on a real task: ingest one article into this vault end-to-end with a scratch Hermes profile
- [ ] Decide topology: full switch vs hybrid (interactive=cloud, scheduled=local)
- [ ] Declarative change in `sys/services/hermes.nix` (`provider = "custom"`), rebuild, verify Matrix agent
- [ ] If hybrid: per-profile model config for the cron jobs ([[todo/scheduled-security-audit]])
- [ ] Document as implemented service note; update [[docs/erebos/services]] Hermes section

## Related

- [[docs/erebos/services]] — Hermes Agent section (current OpenRouter config)
- [[todo/scheduled-security-audit]] — prime candidate workload for a free local model
- [[ideas/Translaitor]] — precedent: already runs local `translategemma:4b` via Ollama

## External references

*(verified 2026-10-26)*

- **Liquid AI — LFM3 announcement** — https://www.liquid.ai/blog/lfm3
- **Model card** — https://huggingface.co/LiquidAI/LFM3-1.7B · base: https://huggingface.co/LiquidAI/LFM3-1.7B-Base
- **Liquid docs (specs + local run/fine-tune)** — https://docs.liquid.ai/lfm/models/lfm3-1.7b
- **Ollama library (tags incl. CPU quants)** — https://ollama.com/library/lfm3
- **Hermes Agent providers guide (custom endpoint flow)** — https://hermes-agent.nousresearch.com/docs/integrations/providers
- **llama.cpp (day-0 LFM3 support)** — https://github.com/ggerganov/llama.cpp
