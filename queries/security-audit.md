---
title: Erebos Security Audit
created: 2026-10-26
updated: 2026-11-11
type: query
tags: [erebos, nixos, security, analysis]
sources: [live service inventory (2026-10-26), vendor advisories — see References]
confidence: high
---

# Erebos Security Audit

> Living document, refreshed by the scheduled audit job ([[todo/scheduled-security-audit]]).
> **Scope: open findings only** — remediated/retracted findings are removed from this page; their
> history lives in the Audit log at the bottom and in git.
> Last re-check: **2026-11-07** (contents API + live units + `ss -ltn`) — base inventory 2026-10-26.
> Config-state reconciled with pushed erebos changes (wiki sync, 2026-11-11, repo HEAD `98a56d4`):
> Alertmanager + Loki now loopback-bound (see P2).
> **Open: P1 Immich; P2 Jellyfin / KEV verify / binds (bridge :3004 + Jellyfin :8096); P3 llama.cpp drift; n8n owner MFA.**

Point-in-time security audit of Erebos: live version inventory (systemd units, Nix store paths, running service APIs) cross-checked against published advisories as of **2026-10-26**.

## Live inventory (verified on host)

| Component | Running version | Exposure |
|---|---|---|
| Traefik | 3.7.13 | public :80/:443 |
| Authentik | **2026.8.5** (container; upgraded `8d47326` 2026-11-06, September advisory batch closed — runbook [[docs/erebos/tutorials]] §7) | public (IdP for everything) |
| Immich | 2.7.5 (server + machine-learning units) | public via Traefik |
| n8n | 2.39.6 | LAN/VPN only (owner-confirmed 2026-10-26 — router does not forward it; **not** public); built-in owner auth by design — Authentik SSO is paid-only in n8n |
| Nextcloud | 35.0.2 + Collabora `collabora/code` container | public |
| Jellyfin | 12.2.0 | public (LDAP auth) |
| Matrix | Synapse 1.163.2 + MAS | public (federation) |
| Ollama → llama.cpp | **llama-cpp 0.8.0** (re-enabled 2026-11-04) | `127.0.0.1:8080` only ✓ (`ss` verified). But router serves **no loaded model**: Hermes' fallback id `LiquidAI/LFM3-1.7B-GGUF:Q6_K` matches a cache-derived entry, **not the preset alias `LFM3-1.7B`** — lazy HF fetch on demand, fragile offline |
| Vaultwarden | 1.37.4 | public (Authentik SSO) |
| Grafana | 13.0.11 | public (Authentik) |
| PostgreSQL | 16.15 | 0.0.0.0:5432, scram, pg_hba scoped to 127.0.0.1/10.88/::1 |
| Podman | 5.8.9 | — |
| kernel / nixpkgs | **7.3.3** / system `26.05.20261107.93fb760` | fresh; rebooted onto systemd stage-1 initrd 2026-11-07 ✓ |

⚠️ **Systemic caveat** *(updated 2026-11-04)*: commit `6ef7e41` **removed `system.autoUpgrade`** entirely — the old silent-rebuild-on-stale-pin trap is gone, replaced by a fully manual pipeline: `nix flake update` → `nix run .#apply`. Flakes were updated that day (nixpkgs-unstable + hermes-agent pins advanced). Patch cadence is now *entirely* a human habit — exactly what [[todo/scheduled-security-audit]] exists to remind about.

## Findings by risk

### P1 — Immich 2.7.5: three published CVEs, one unauthenticated *(sole open P1)*
Verified 2026-11-07: running unit still `immich-2.7.5`, `permittedInsecurePackages` still in `flake.nix`.
- **CVE-2026-119328** (6.5, fixed 3.3.0): **unauthenticated** `/predict` on the machine-learning service accepts an `options` object → `cache_dir` path injection → arbitrary directory write + `shutil.rmtree()` on chosen paths. Reachability: ML listens on loopback (`127.0.0.1:3003`), so exploitation needs either a path proxied through the immich server or a foothold on the host/podman net — but this is the same podman network as every container.
- **CVE-2026-119330** (fixed 3.2.0): cross-user shared links (authorized user can share another user's asset anonymously).
- **CVE-2026-119329** (4.3, fixed 3.0.0): view-only partner GPS leak of archived assets.
Note: nixpkgs itself flags `immich-2.7.5` via `permittedInsecurePackages` in `flake.nix` — the repo is *declaring* it knowingly vulnerable.
**Fix**: update to **≥ 3.3.0** (check current in nixpkgs 26.05 after `nix flake update`; if the NixOS module lags, move Immich to the official container like the rest of the fleet) and **remove the `permittedInsecurePackages` entry**.

### P2 — Jellyfin 12.2.0 inside "through 12.4" advisory set (2026-10-30)
- **CVE-2026-119549** (7.1 High): SSRF via gzip-SVG profile images (authenticated upload → server fetches internal URLs — on this host that means podman-net pivot to unauthenticated monitoring endpoints, see below).
- **CVE-2026-119551** (7.1): SuggestionsController IDOR (non-admin reads restricted items).
- **CVE-2026-119550** (3.5): playlist ID disclosure.
All require an authenticated (non-admin) account → single-user instance = low practical risk, but patched release (>12.4, GHSA-cf3c-8m59-2vhx) should land via `nix flake update` when available.

### P2 — Nextcloud / Collabora: actively-exploited adjacent CVE
- **CVE-2026-110447** (CVSS 9.2, **CISA KEV 2026-10-23, active exploitation since ~2026-09-10**): unauthenticated OS command injection in the Nextcloud **`richdocumentscode` app** (Built-in CODE Server), fixed in 26.4.303. You run Collabora as a **separate container** (`collabora/code`), which is a different distribution — but verify **the built-in CODE app is not installed/enabled in Nextcloud** (`occ app:list` → if `richdocumentscode` present, `occ app:remove`); its mere presence would put a live-exploited RCE behind only the Nextcloud login.
- Nextcloud 35.0.2: searched advisories affect 29–33.x — no known unpatched NC-35 advisory at audit time; keep the image fresh (it auto-updates daily via the update-containers service).

### P2 — Unauthenticated internal services still on all interfaces (lateral movement surface)
**2026-11-07 (post-`85c5ebe`):** Prometheus (`:9090`) & node_exporter (`:9100`) bind `127.0.0.1` ✓;
live `ss` **still shows `*:9093` (Alertmanager), `*:3004` (bridge), `*:3100` (Loki)**.
(The webhook-secret plaintext leak formerly folded into this finding was **remediated same day** —
see Audit log; secrets now agenix-only via envsubst placeholders, old value rotated.)
`podman0` is a trusted interface, so any compromised container is a guest: it can **post silences to
Alertmanager** (blinding alerting) and **read all logs via Loki**.
**Fix**: bind Alertmanager + Loki (+bridge) to `127.0.0.1` in `sys/services/monitoring/*`, Jellyfin
`network.bindAddrs = [ "::1" ]`.

**2026-11-11 (wiki sync w/ pushed changes; repo HEAD `98a56d4`):** Alertmanager (`:9093`) now
loopback-bound (`listenAddress = "127.0.0.1"` in `sys/services/monitoring/prometheus/default.nix`) and
Loki (`:3100`) now loopback-bound (`http`/`grpc listen_address = "127.0.0.1"` in
`sys/services/monitoring/loki.nix`, commit `f13b255`) — the P2 bind remediation landed with the pushed
monitoring hardening series. **Still open:** matrix-alertmanager bridge (`:3004`) — owner declined the
bind (2026-11-10) — and Jellyfin (`:8096`) `network.bindAddrs = [ "::1" ]`.

### P3 — Clean/fresh at audit time
- **Traefik 3.7.13**: ahead of all known 2026 advisories (3.7.3 batch etc.); no mTLS/HTTP3 in use.
- **Synapse 1.163.2**: ahead of fixes for PYSEC-2026 CPU-starvation (1.152.1), federation depth (1.127.1), thumbnails/Ghostscript (1.120.1).
- **Vaultwarden 1.37.4, PostgreSQL 16.15, Grafana 13.0.11, nginx, podman 5.8.9, kernel 7.3.1**: no unpatched published advisories found; backports current *as of the flake pin* — which decays until the next `nix flake update`.
- **GGUF caveat** (carries over from the Ollama-era finding): model files can themselves be exploit vectors (memory-leak <0.17.1; CVE-2026-86289 needed a crafted GGUF) — only pull models from trusted sources ([[ideas/translaitor]] and the LFM3 preset).

### P3 — llama.cpp runtime: two open config drifts
1. **Hermes' fallback points at the wrong model id**: `default = "LiquidAI/LFM3-1.7B-GGUF:Q6_K"` matches
   a cache-derived entry, **not the `modelsPreset` alias `LFM3-1.7B`** — both currently `unloaded`, so
   the router would lazy-fetch from HF on demand and the offline-safety-net fallback likely fails
   exactly when it's needed (no network = no fetch). **Fix**: align `model.default` to `LFM3-1.7B`,
   smoke-test one load. (Context concern cleared: `/props` reports `context_length: 262144`.)
2. **`modelsPreset` is a removed option in newer nixpkgs** — the next `nix flake update` past the removal
   breaks evaluation; migrate to per-model `services.llama-cpp.<name>`.

## Action list *(open items only — completed ones are deleted, see Audit log)*

- [ ] P1: Immich → ≥ 3.3.0 and **delete** `permittedInsecurePackages = [ "immich-2.7.5" ]` from `flake.nix` — verified still open 2026-11-07
- [ ] n8n: enable MFA on the owner account; keep n8n updates urgent (auth→RCE advisories all year); LAN/VPN boundary is the wall
- [ ] P2: verify `richdocumentscode` app absent in Nextcloud (KEV-listed, actively exploited)
- [ ] P2: track Jellyfin fix (>12.4) via `nix flake update`
- [ ] P2: bind matrix-alertmanager 3004 + Jellyfin 8096 to loopback (Alertmanager 9093 + Loki 3100
      now bound per repo HEAD `98a56d4` / `f13b255`).
- [ ] P3: llama.cpp — align Hermes fallback `model.default` to preset alias `LFM3-1.7B` + smoke-test load; migrate off removed `modelsPreset` option at next flake update
- [ ] Systemic: keep a manual patch cadence — `nix flake update` + `nix run .#apply` on a schedule (weekly?); [[todo/scheduled-security-audit]] is the reminder mechanism
- [ ] Optional: run `nix-audit-info`/`vulnix` against the system to catch store-level flagged packages automatically

## References

- https://github.com/goauthentik/authentik/security/advisories (batch incl. GHSA-qgqp-xh8r-v73r, GHSA-cxwx-9x59-28qm, GHSA-m9h4-7j9c-55x9, GHSA-h6c5-mpvq-j4jc)
- https://github.com/immich-app/immich/security/advisories/GHSA-72rm-x7xr-v679 (CVE-2026-119328) · https://www.cve.org/CVERecord?id=CVE-2026-119330 · https://www.cve.org/CVERecord?id=CVE-2026-119329
- https://github.com/n8n-io/n8n/security/advisories/GHSA-v364-rw7m-3263 (CVSS 9.9) · https://github.com/advisories/GHSA-HQR4-H3XV-9M3R · https://github.com/n8n-io/n8n/security/advisories/GHSA-c8xv-5998-g76h · https://github.com/n8n-io/n8n/security/advisories/GHSA-mxrg-77hm-89hv
- https://github.com/jellyfin/jellyfin/security/advisories/GHSA-cf3c-8m59-2vhx · https://nvd.nist.gov/vuln/detail/cve-2026-119551
- https://censys.com/advisory/cve-2026-110447/ (Collabora built-in CODE, CISA KEV)
- https://doc.traefik.io/traefik/contributing/security-decisions/ · https://github.com/traefik/traefik/security/advisories
- https://security.snyk.io/vuln/SNYK-PYTHON-MATRIXSYNAPSE-16726260 (Synapse 1.152.1 fixes) · https://kb.cert.org/vuls/id/518910 (Ollama GGUF)
- https://github.com/ryantm/agenix · nixpkgs `permittedInsecurePackages` semantics: https://wiki.nixos.org/w/index.php?title=Security#Known_vulnerabilities_in_packages

## Related

- [[todo/scheduled-security-audit]] — the job that keeps this page current
- [[docs/erebos/secrets]] — credential blast radius if P1 items are exploited
- [[todo/monitoring-alerting]] — silencing risk from unauthenticated Alertmanager
- [[docs/erebos/index]] — the stack audited

## Audit log

- **2026-10-26** — Initial snapshot (manual). P1: Authentik 2026.8.1 advisory batch (fix 2026.8.2), Immich 2.7.5 incl. CVE-2026-119328, ~~n8n internet-exposed without SSO~~. P2: Jellyfin ≤12.4 set; verify `richdocumentscode` (KEV); monitoring ports on 0.0.0.0. Systemic: flake-pin patch cadence. Open follow-ups: P1 items not yet confirmed remediated. *(Ollama 0.40.3 row since superseded — runtime replaced, see below.)*
- **2026-10-26 (same day, corrected)** — n8n finding downgraded P1→P3: **not internet-forwarded** (owner-confirmed); original in-host test misled by NAT hairpinning (200 from inside ≠ public reachability). Built-in owner auth is deliberate (n8n SSO is paid). Residual: enable MFA on owner, keep update cadence high, LAN/VPN boundary is the wall.
- **2026-11-04** — Delta from commit `6ef7e41` (no full re-inventory): Ollama → llama-cpp 0.8.0 with **new unauthenticated `0.0.0.0:8080` bind** (P3 above); `system.autoUpgrade` removed → manual flake-update cadence (pins advanced same day — Authentik/Immich remediation **not yet verified by version probe**); n8n now from `pkgs-unstable`; `hermes.age` access tightened to 0400. P1/P2 status: unchanged pending next inventory. *(Same day, owner correction: llama-cpp unit **purposely stopped** — absence of :8080 is intended, not a silent service death; do not re-flag in future runs. A todo was briefly filed for it and removed per lifecycle.)*
- **2026-11-04 (later, `d5340e3`)** — Bind P3 **remediated**: llama.cpp rebound to `127.0.0.1` ✓. Hermes topology inverted: OpenRouter primary, local LFM3 fallback ("fallback to local inference"). Audit→fix turnaround: ~1 h.
- **2026-11-04 (live re-check run)** — Full-ish re-inventory as requested. **Key finding: the two P1s are NOT fixed** — repo HEAD still `authentikVersion = "2026.8.1"` and running Immich still `2.7.5` (+ `permittedInsecurePackages`): image/Nix pins don't move on `nix flake update`, likely the misunderstanding; runbook in [[docs/erebos/tutorials]] §7. **Fixed:** Prometheus+Immich binds now loopback (live `ss` ✓, commit `b1b690e`), llama-cpp active on loopback, kernel 7.3.0→7.3.1, system rebuild `26.05.20261104.994bb33`. **New:** llama-cpp fallback id mismatch — router up but no model loaded, Hermes targets cache id not preset alias (offline safety-net likely broken). **Still open:** Alertmanager 9093 / Loki 3100 / Jellyfin 8096 binds; Jellyfin >12.4; `richdocumentscode` verify; n8n owner MFA.
- **2026-11-07 (correction + follow-up run)** — Owner challenged the Authentik verdict; they were right for *today*: `8d47326 "upgrading authentik"` (Nov 6) moved main → **2026.8.5** and the live unit pulls that image. The Nov 4 "NOT remediated" call was true at the time (commit didn't exist yet), but a follow-up check was then mis-confirmed by a **stale `raw.githubusercontent.com` CDN blob** — corrected methodology: contents/commits API + live unit ExecStart are the sources of truth. P1 Authentik **CLOSED**; Immich re-probed via contents API + live store path: still 2.7.5 + `permittedInsecurePackages` — **sole open P1**.
- **2026-11-07** — Alerting day. **Remediated:** matrix-alertmanager webhook plaintext leak (owner commit `85c5ebe`): value rotated, now agenix-only — KEY-VALUE file feeds Alertmanager via module envsubst `environmentFile`, store config verified to hold only `` placeholder (grep count of old value = 0); bridge holds raw twin; old value lingers in git history but is voided. Prometheus+node_exporter binds → loopback ✓ (earlier: `b1b690e`/prometheus.nix). 9-rule alert set deployed & live. Also: initrd systemd stage-1 completed (`5f41664`/`034cd93`, verified live), Authentik 2026.8.5 confirmed earlier today, disko device paths corrected (`6973308` prod=/dev/sdd, stage=/dev/vda). **Still open:** binds AM :9093 / bridge :3004 / Loki :3100 (live ss `*`) + Jellyfin 8096; Immich P1; Jellyfin P2; KEV verify P2.
- **2026-11-11** — Wiki sync after owner push (erebos repo HEAD `98a56d4` + hardening series `9b24091`…`98a56d4`): log pipeline now LIVE (Alloy `alloy.nix` store-path config, journal→Loki + traefik/sshd/fail2ban geoip; `host="erebos"`), Loki retention 720h + loopback `127.0.0.1:3100` (`f13b255`), alloy+loki scrape jobs landed (`abc3f49`, `7f50ca8`), Prometheus rules refactored to Nix attrsets (`98a56d4`; 12 rules, promtool check at build), pipeline-health alerts landed (AlloyConfigNotLoaded, LokiDroppingEntries, LogsPipelineStalled), fail2ban sshd jail LIVE (`9b24091`). Config deltas vs 2026-11-07 audit: Alertmanager :9093 + Loki :3100 now loopback-bound (repo) — remediated; **still open:** bridge :3004 (owner declined bind), Jellyfin :8096. Immich P1 sole open P1 unchanged.
