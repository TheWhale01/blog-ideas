---
title: Fail2ban tuning
created: 2026-11-10
updated: 2026-11-10
type: todo
tags: [erebos, security, fail2ban, nixos]
sources: [erebos:sys/services/fail2ban.nix, [[docs/erebos/services/fail2ban]]]
---

# Fail2ban tuning (follow-ups on the live sshd jail)

## 1. Decide: ban anonymous preauth fuzzers? (`mode` upgrade)

Current sshd jail (`mode=normal`) ignores `Connection closed/reset ... [preauth]` lines — the
recurring Vultr prober pool is never banned. Upgrading to
`jails.sshd.settings.filter = "sshd[mode=aggressive]";` counts them.

**Preconditions before switching:**
- Add home IPv6 prefix to `ignoreIP` — owner logins arrive over IPv6 (`2a02:842a:3540:4201:…` —
  a /64 NOT covered by today's list; verify current prefix on pontos with `ip -6 addr`).
- Add Netbird `wt0` subnet to `ignoreIP` if SSH is used over it.
- Rationale: aggressive mode counts interrupted *legitimate* handshakes; without ignoreIP the
  owner can self-ban (bantime grows to 48h; recovery via Netbird/console only).

## 2. Prototype one web jail (traefik JSON, journald backend)

Traefik access JSON is already in journald (`{unit="traefik.service"}`, real client IPs — services
are direct, not behind Cloudflare). A jail would use `backend=systemd` +
`journalmatch = { _SYSTEMD_UNIT = "traefik.service" }` and a custom filter anchoring the JSON
fields, e.g. for auth-endpoint floods: failregex capturing `"ClientHost":"<HOST>"` combined with
`"DownstreamStatus":(401|403)` and login paths (authentik `/api/v3/…/login`, vaultwarden
`/accounts/…`, nextcloud `/login/v2`-style — pick ONE target first; probe real paths in Loki before
writing the filter). Watch Cloudflare caveat if [[ideas/cloudflare-tunnel-ssh]]-style proxying ever
fronts the services.

## 3. Re-evaluate after port-22 removal
If [[ideas/cloudflare-tunnel-ssh]] lands, scope the sshd jail to Netbird-only or keep as
belt-and-suspenders.

## History
- 2026-11-10: split out when the fail2ban idea went LIVE (`9b24091`) and folded into [[docs/erebos/services/fail2ban]].
