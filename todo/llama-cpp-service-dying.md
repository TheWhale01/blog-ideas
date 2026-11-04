---
title: llama-cpp Service Dying at Startup
created: 2026-11-04
updated: 2026-11-04
type: todo
tags: [erebos, todo, fix, AI]
sources: ["erebos:6ef7e41"]
---

# llama-cpp Service Dying at Startup

- **Status**: Not started
- **Priority**: High (local model path is down; agents ride the OpenRouter fallback)
- **Parent**: [[todo/index]]

## Symptoms (measured 2026-11-04, from the hermes service user)

- `systemctl show llama-cpp`: `ActiveState=inactive`, `SubState=dead`; started 21:06:52, stopped
  21:07:00, **`code=exited status=0`** — clean exit ~8 s after start, no restart loop.
- Nothing listening on `:8080` (`/props` and `/v1/models` unreachable); `ollama` unit also inactive
  (module deleted in `6ef7e41`).
- Consequence live-observed: Hermes fell back to `fallback_providers` (OpenRouter qwen3.8-flash) —
  the fallback design worked, but the primary local path is not serving.

## Diagnosis steps

- [ ] `journalctl -u llama-cpp` as root/wheel (hermes user lacks `systemd-journal` group — by design)
- [ ] Suspects to check, in order:
  - **Model download**: `modelsPreset` HF fetch on first start — partial/cached GGUF or blocked
    network from the `llama-cpp` service user's sandbox (PrivateDevices note, proxy-less env).
  - **`-fa on` arg drift**: llama.cpp 0.8.0 `-fa` semantics — if it takes no value, `on` may be
    parsed oddly; test `llama-server -h`.
  - **exit 0 oddity**: clean exit suggests it finished loading then shut down (e.g. health/ready
    probe mode or a preset with no default model selected — inspect the generated
    `/nix/store/...-llama-models.ini`).
- [ ] After it stays up: **verify effective context** — `curl :8080/props | grep -i n_ctx`; Hermes
  declares 64000 by default while llama-server started with no `-c` → classic mismatch; if so, add
  `-c` to `extraFlags` and align `context_length`.
- [ ] Smoke: forced tool-call via `/v1/chat/completions` before pointing Hermes back at it.
- [ ] Upstream landmine: `modelsPreset` is **removed in current nixpkgs** — migrate
  `sys/services/llamacpp.nix` to the per-model `services.llama-cpp.<name>` form *before or with*
  this fix, or the next `nix flake update` will break the module evaluation.

## Related

- [[docs/erebos/services]] — llamacpp.nix section (design + flags)
- [[queries/security-audit]] — 0.0.0.0:8080 unauthenticated bind note
- [[todo/scheduled-security-audit]] — future detector for silent service death
