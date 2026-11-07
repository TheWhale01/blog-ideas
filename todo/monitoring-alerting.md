---
title: Implement Proper Monitoring & Alerting
created: 2026-10-15
updated: 2026-11-07
type: todo
tags: [nixos, erebos, todo, improvement, infrastructure]
sources: ["live inspection 2026-11-07: alertmanager 0.31.1 + matrix-alertmanager active"]
---

# Implement Proper Monitoring & Alerting

- **Status**: In progress — discovered already 80% deployed (2026-11-07)
- **Priority**: Critical
- **Parent**: [[todo/index]]

## Reality check (was written assuming nothing existed)

`sys/services/monitoring/prometheus.nix` already contains a **complete alerting pipeline**, live and
healthy (`/-/healthy` = OK):

- `services.prometheus.alertmanager` — enabled, v0.31.1, port 9093, `openFirewall = false`
- routing → receiver `erebot` → `webhook_configs` → `services.matrix-alertmanager` (port 3004),
  homeserver `matrix.thewhale.fr`, `tokenFile = erebot.age`, `secretFile = matrix-alertmanager-webhook.age`,
  posting into a configured room ID
- Prometheus side: 2 alert rules already active (`ServiceCrashed` 1m/critical, `ServiceOffline` 3m/warning)

Synthetic end-to-end test fired 2026-11-07 (alert `WikiPipelineTest` injected via AM API, visible in
`/api/v2/alerts`) — **awaiting owner confirmation it rendered in the Matrix room**.

## Remaining work

- [x] ~~Fire one synthetic alert end-to-end~~ attempted 2026-11-07 — **pipeline is BROKEN at the
      bridge→Matrix leg**: direct bridge POST with the correct webhook envelope (`receiver: erebot`)
      returns `{"result":"error"}` + HTTP 500, i.e. the room for `erebot` IS matched but the Matrix
      send fails. Strongest hypothesis (2026-11-07): **@erebot likely does not exist** — MAS is the
      identity store (`enable_registration=false`, passwords off) and the profile probe returned
      `No row found (profiles)` (inconclusive: an existing user with no displayname set gives the
      same answer; positive controls @whale/@odin resolved fine). The webhook secret itself is valid
      (bridge accepted POSTs without 401). `manage register-user` below resolves both cases: it
      errors cleanly if the user already exists, and the token issuance then fixes the dead-token case.
- [x] ~~Create the bot in MAS~~ (attempted; superseded — @erebot turned out to be an **appservice**
      user, see the route-A resolution below. MAS recipe kept for reference/other bots — NOT
      Authentik, Matrix users live in MAS; binary from `systemctl show mas -p ExecStart` →
      mas-cli 1.28.0):
      ```bash
      MAS=/nix/store/4m2bp2m0rpiy96vfa3xiv3iyw2scgmk5-matrix-authentication-service-1.28.0/bin/mas-cli
      CFG="--config /run/secrets/mas --config /nix/store/0xlj62hlal43kzq5810kbwv629kgsj5d-mas-config.yaml"
      sudo -u mas $MAS $CFG manage register-user --yes erebot --display-name "Erebot"
      sudo -u mas $MAS $CFG manage issue-compatibility-token erebot matrix-alertmanager
      ```
      (compat-token = the documented long-lived headless-bot route; NO admin privileges flag; token
      output → `sudo agenix -e` into `secrets/shared/matrix/erebot.age` as a single line → rebuild →
      invite `@erebot:matrix.thewhale.fr` to room `!FVqWRrn1…` in Element — matrix-webhook
      auto-joins invites → re-fire test alert)
- [x] ~~create via MAS~~ **blocked 2026-11-07**: `manage register-user --yes erebot` →
      `Error: Username not available on homeserver` — Synapse reports the localpart taken: an
      **`@erebot` exists as a Synapse-LOCAL user outside MAS** (likely from the original webhook
      setup via the registration shared secret — also explains the 500 at delivery: old/invalid
      token in erebot.age for an account MAS doesn't own). Verify:
      `sudo -u postgres psql -d matrix-synapse -Atc "SELECT name,deactivated,is_guest FROM users WHERE name LIKE '%erebot%';"`
- [x] ~~Choose remediation route~~ → **A, via the appservice** (owner confirmed 2026-11-07:
      `@erebot` exists **as an application-service user**, registered through
      `secrets/${env}/matrix/appservice.age` → `/run/secrets/matrix-appservice`). MAS is not in the
      picture for this account at all — and can't be (exclusive AS namespace reserves the localpart).
      Fresh-token recipe (no admin, no MAS):
      ```bash
      sudo cat /run/secrets/matrix-appservice            # as_token
      curl -X POST http://127.0.0.1:8008/_matrix/client/v3/login \
        -H "Authorization: Bearer <as_token>" -H 'Content-Type: application/json' \
        -d '{"type":"m.login.application_service","identifier":{"type":"server","user":"erebot"}}'
      ```
      ⚠️ must hit **localhost:8008 (Synapse direct)** — Traefik's `mas-compat` router proxies all
      public `/login` to MAS, which rejects `m.login.application_service`. Token → `agenix -e`
      into `secrets/shared/matrix/erebot.age` → rebuild. AS virtual users have no profile row
      until they set one — explains the earlier 404 probe; display name can be PUT with the new
      token. (The `.env PermissionDenied` warning during mas-cli runs = cosmetic; cwd unreadable
      to the mas user.)
      NOTE on as_token ≠ access token: per spec, `as_token` is only valid on AS-API endpoints
      (login, namespace queries) — Synapse accepting it on arbitrary client endpoints is an
      unguaranteed implementation detail; always mint the syt_ token first.
- [x] ~~Diagnose delivery 500~~ **RESOLVED by journalctl 2026-11-07 19:49**: bridge logs
      `M_FORBIDDEN: User @erebot:matrix.thewhale.fr not in room` — the configured room is
      **`!C-EIAS89vRQV7k6yUk34TTqXDro5S3Xwx_sJidUW3Lk:matrix.thewhale.fr`** (supersedes the earlier
      `!FVqWRrn1…` ID noted from the config summary — stale). This 403 was ALSO the cause of the
      direct-bridge HTTP 500 earlier. Chain fully proven healthy otherwise: token now VALID
      (403-from-send ≠ 401 M_UNKNOWN_TOKEN), webhook secret OK, Alertmanager routing OK.
      Failed events are DROPPED (`clearing queue`), not retried.
      Fix: invite @erebot to that room in Element (restart bridge if no auto-join), or repoint
      `matrixRooms` (room ID must be copied from Element — chat/log text mangles `+/=` as `_`).
      ⚠️ silence/fix the queued ServiceOffline spam BEFORE the bot joins, it lands instantly.
      Versions observed: matrix-alertmanager 0.9.0 (js-sdk), Synapse 1.164.2.
- [ ] Then re-test: inject via `http://127.0.0.1:9093/api/v2/alerts` (curl in the chat thread), expect
      message in the alerts room within ~40 s
- [ ] ⚠️ **Fix rule spam before enabling delivery**: live AM currently holds firing `ServiceOffline`
      alerts for `systemd-update-done.service` (oneshot), `poweroff.target` (target) and
      `virtlxcd-admin.socket` (socket) — the `unit-include` regex in node_exporter extraFlags is not
      filtering as intended, and the rule expression ignores unit type. Fix:
      `expr: node_systemd_unit_state{name=~".+\\.service", type!~"oneshot|oneshot-reload", state="active"} == 0`
      (and audit why `--collector.systemd.unit-include` isn't effective in the running node_exporter)
- [ ] Rotate the leaked webhook secret + remove it from git: currently committed plaintext in the
      **public** repo (`prometheus.nix` receiver URL) and *working* — anyone can forge alerts. New
      secret lives agenix-only; build the URL from a gitignored local file
      (`sys/secrets.local.nix`, 0600, optional import)
- [ ] Bind loopback (closes audit P2 remainder): AM 9093, bridge 3004, Loki 3100 — live `ss` still `*`
- [ ] Expand rules beyond systemd state
- [ ] Loki LogQL error panels
- [ ] Noise guard + confirm single alert brain (Grafana unified alerting stays optional)

## Related

- [[docs/erebos/monitoring]] — the stack
- [[queries/security-audit]] — P2 binds + new secret-exposure finding
- [[ideas/Drive-Health-Monthly-Newsletter]] — future SMART alert source into this pipeline
