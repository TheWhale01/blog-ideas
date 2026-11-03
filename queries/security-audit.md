---
title: Erebos Security Audit
created: 2026-10-26
updated: 2026-10-26
type: query
tags: [erebos, nixos, security, analysis]
sources: [live service inventory (2026-10-26), vendor advisories — see References]
confidence: high
---

# Erebos Security Audit

> Living document, refreshed by the scheduled audit job ([[ideas/Scheduled-Security-Audit]]).
> Newest findings first; per-run history in the Audit log at the bottom.
> Current snapshot: **2026-10-26**.

Point-in-time security audit of Erebos: live version inventory (systemd units, Nix store paths, running service APIs) cross-checked against published advisories as of **2026-10-26**.

## Live inventory (verified on host)

| Component | Running version | Exposure |
|---|---|---|
| Traefik | 3.7.13 | public :80/:443 |
| Authentik | 2026.8.1 (container) | public (IdP for everything) |
| Immich | 2.7.5 (server + machine-learning units) | public via Traefik |
| n8n | 2.39.6 | LAN/VPN only (owner-confirmed 2026-10-26 — router does not forward it; **not** public); built-in owner auth by design — Authentik SSO is paid-only in n8n |
| Nextcloud | 35.0.2 + Collabora `collabora/code` container | public |
| Jellyfin | 12.2.0 | public (LDAP auth) |
| Matrix | Synapse 1.163.2 + MAS | public (federation) |
| Ollama | 0.40.3 | **127.0.0.1 only** ✓ |
| Vaultwarden | 1.37.4 | public (Authentik SSO) |
| Grafana | 13.0.11 | public (Authentik) |
| PostgreSQL | 16.15 | 0.0.0.0:5432, scram, pg_hba scoped to 127.0.0.1/10.88/::1 |
| Podman | 5.8.9 | — |
| kernel / nixpkgs | 7.3.0 / 26.05 pinned rev 2026-10-27 | fresh |

⚠️ **Systemic caveat**: `autoUpgrade` runs `nixos-rebuild` on the *pinned flake.lock* — it does **not** advance nixpkgs or container images. Security fixes land only when you run `nix flake update` + `nix run .#apply` for images. This is the real patch pipeline gap.

## Findings by risk

### P1 — Authentik 2026.8.1 sits exactly in the affected range of the September batch
Advisories published 2026-09-09 list affected `<= 2026.8.1`, patched in **2026.8.2**:
- **GHSA-qgqp-xh8r-v73r** (CVE-2026-94606, 8.9 High): **MFA bypass via recipient override** in the email authenticator — attacker knowing a password gets the OTP code sent to *their* address → full account takeover of the IdP = every SSO app. Only exploitable if the **email authenticator** is enrolled during a flow — if you use only TOTP/WebAuthn the exposure drops, but this is the front door; do not rely on that.
- **GHSA-h6c5-mpvq-j4jc** (High): privilege escalation via delegated group/user management.
- **GHSA-m9h4-7j9c-55x9** (8.1 High): stored credentials (SMTP, outpost tokens, provider configs) readable with *view* permission alone — single-admin box = low practical risk, still patch.
- **GHSA-cxwx-9x59-28qm** (7.5 High): unauthenticated SAML DoS (only if SAML enabled; you run LDAP+OIDC → low).
**Fix**: bump `ghcr.io/goauthentik/server` to ≥2026.8.2 (prefer current stable) in `sys/containers/authentik.nix`, `podman pull`, restart stack; tasks run migrations automatically.

### P1 — Immich 2.7.5: three published CVEs, one unauthenticated
- **CVE-2026-119328** (6.5, fixed 3.3.0): **unauthenticated** `/predict` on the machine-learning service accepts an `options` object → `cache_dir` path injection → arbitrary directory write + `shutil.rmtree()` on chosen paths. Reachability: ML listens on loopback (`127.0.0.1:3003`), so exploitation needs either a path proxied through the immich server or a foothold on the host/podman net — but this is the same podman network as every container.
- **CVE-2026-119330** (fixed 3.2.0): cross-user shared links (authorized user can share another user's asset anonymously).
- **CVE-2026-119329** (4.3, fixed 3.0.0): view-only partner GPS leak of archived assets.
Note: nixpkgs itself flags `immich-2.7.5` via `permittedInsecurePackages` in `flake.nix` — the repo is *declaring* it knowingly vulnerable.
**Fix**: update to **≥ 3.3.0** (check current in nixpkgs 26.05 after `nix flake update`; if the NixOS module lags, move Immich to the official container like the rest of the fleet) and **remove the `permittedInsecurePackages` entry**.

### P3 (corrected 2026-10-26) — n8n: not publicly exposed; built-in auth is a deliberate choice
Original finding said "internet-exposed without Authentik" — **wrong**, per owner: the router/firewall in front of Erebos does not make n8n reachable from the internet (initial test hit NAT hairpinning, which returns 200 from inside the network and is misleading — note this method caveat for future audits). Commit `5e6c83b` removed the Authentik middleware because n8n's SSO feature is **paid-only** — so the built-in owner login is the intended gate, not an oversight.
Residual risk to manage deliberately, since "authenticated user = arbitrary code execution + stored credentials" is n8n's design:
- Keep n8n current with the same urgency as an exposed service — 2026's advisories were all an *authenticated*-to-RCE chain (GHSA-v364-rw7m-3263 CVSS 9.9 et al.).
- Enable MFA/2FA on the n8n owner account (supported in self-hosted/community builds) — single password is the only gate now.
- Confirm the LAN boundary itself is where you want the wall (any device on the Netbird/LAN mesh can reach :5678).

### P2 — Jellyfin 12.2.0 inside "through 12.4" advisory set (2026-10-30)
- **CVE-2026-119549** (7.1 High): SSRF via gzip-SVG profile images (authenticated upload → server fetches internal URLs — on this host that means podman-net pivot to unauthenticated monitoring endpoints, see below).
- **CVE-2026-119551** (7.1): SuggestionsController IDOR (non-admin reads restricted items).
- **CVE-2026-119550** (3.5): playlist ID disclosure.
All require an authenticated (non-admin) account → single-user instance = low practical risk, but patched release (>12.4, GHSA-cf3c-8m59-2vhx) should land via `nix flake update` when available.

### P2 — Nextcloud / Collabora: actively-exploited adjacent CVE
- **CVE-2026-110447** (CVSS 9.2, **CISA KEV 2026-10-23, active exploitation since ~2026-09-10**): unauthenticated OS command injection in the Nextcloud **`richdocumentscode` app** (Built-in CODE Server), fixed in 26.4.303. You run Collabora as a **separate container** (`collabora/code`), which is a different distribution — but verify **the built-in CODE app is not installed/enabled in Nextcloud** (`occ app:list` → if `richdocumentscode` present, `occ app:remove`); its mere presence would put a live-exploited RCE behind only the Nextcloud login.
- Nextcloud 35.0.2: searched advisories affect 29–33.x — no known unpatched NC-35 advisory at audit time; keep the image fresh (it auto-updates daily via the update-containers service).

### P2 — Unauthenticated internal services on `0.0.0.0` (lateral movement surface)
`ss` shows Prometheus 9090, Alertmanager 9093, Loki 3100, Jellyfin 8096, Nextcloud 8881, Authentik 9000 (admin API), Immich 2283, the *arr suite, slskd 5030, flood 3002… bound to **all interfaces**. The nftables default-deny (only 80/443 + trusted interfaces) keeps these off the internet, but `podman0` is trusted: **any compromised container is a guest** — it can read all metrics, query all logs via Loki, **post silences to Alertmanager** (blind your future item-1 alerting), and hit service APIs directly bypassing Traefik/Authentik.
**Fix**: bind what you can to `127.0.0.1` (Prometheus/Loki/Alertmanager options in `sys/services/monitoring/*`; Jellyfin `bind = ::1`), keep Authentik reachable only via its outpost proxy; treat "Firewall + loopback" as the two controls and audit bindings with `ss -ltn` in your periodic lint.

### P3 — Clean/fresh at audit time
- **Traefik 3.7.13**: ahead of all known 2026 advisories (3.7.3 batch etc.); no mTLS/HTTP3 in use.
- **Synapse 1.163.2**: ahead of fixes for PYSEC-2026 CPU-starvation (1.152.1), federation depth (1.127.1), thumbnails/Ghostscript (1.120.1).
- **Ollama 0.40.3**: patched vs GGUF advisories (memory-leak <0.17.1; CVE-2026-86289 needs loading a crafted model) — and **correctly bound to loopback**. Relevant for [[ideas/Translaitor]]: only pull models from trusted sources.
- **Vaultwarden 1.37.4, PostgreSQL 16.15, Grafana 13.0.11, nginx, podman 5.8.9, kernel 7.3.0**: no unpatched published advisories found at audit; nixpkgs pin is 24 h old, so backports are current *as of the pin* — which decays until the next `nix flake update`.

## Action list

- [ ] P1: Authentik container → ≥ 2026.8.2 (`sys/containers/authentik.nix`, pull + restart stack)
- [ ] P1: Immich → ≥ 3.3.0 and **delete** `permittedInsecurePackages = [ "immich-2.7.5" ]` from `flake.nix`
- [x] ~~P1: restore Authentik middleware (or Netbird-only bind) on the n8n router~~ — **corrected**: n8n is not internet-forwarded (owner), built-in auth chosen because SSO is paid; residual actions:
- [ ] Enable MFA on the n8n owner account; treat n8n updates like an exposed service; confirm LAN/VPN boundary is acceptable
- [ ] P2: verify `richdocumentscode` app absent in Nextcloud (KEV-listed, actively exploited)
- [ ] P2: track Jellyfin fix (>12.4) via `nix flake update`
- [ ] P2: re-bind Prometheus/Loki/Alertmanager (+ Jellyfin) to loopback; re-check `ss -ltn` after
- [ ] Systemic: define a patch cadence — `nix flake update` + rebuild on a schedule (weekly?), since `autoUpgrade` alone never moves the pin; candidate for [[todo/index]]
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

- [[ideas/Scheduled-Security-Audit]] — the job that keeps this page current
- [[todo/secrets-management]] — credential blast radius if P1 items are exploited
- [[todo/monitoring-alerting]] — silencing risk from unauthenticated Alertmanager
- [[docs/erebos/index]] — the stack audited

## Audit log

- **2026-10-26** — Initial snapshot (manual). P1: Authentik 2026.8.1 advisory batch (fix 2026.8.2), Immich 2.7.5 incl. CVE-2026-119328, ~~n8n internet-exposed without SSO~~. P2: Jellyfin ≤12.4 set; verify `richdocumentscode` (KEV); monitoring ports on 0.0.0.0. Systemic: flake-pin patch cadence. Open follow-ups: P1 items not yet confirmed remediated.
- **2026-10-26 (same day, corrected)** — n8n finding downgraded P1→P3: **not internet-forwarded** (owner-confirmed); original in-host test misled by NAT hairpinning (200 from inside ≠ public reachability). Built-in owner auth is deliberate (n8n SSO is paid). Residual: enable MFA on owner, keep update cadence high, LAN/VPN boundary is the wall.
