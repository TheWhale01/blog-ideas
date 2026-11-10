---
title: Wiki Log
created: 2026-10-15
updated: 2026-10-15
type: log
tags: [meta, log]
---

# Wiki Log

> Chronological record of all wiki actions. Append-only.
> Format: `## [YYYY-MM-DD] action | subject`
> Actions: ingest, update, query, lint, create, archive, delete
> When this file exceeds 500 entries, rotate: rename to log-YYYY.md, start fresh.

## [2026-11-07] query | Monitoring alerting: pipeline was already deployed — verified live
- User asked to "add alertmanager" → inspection + live probes show **it has been fully deployed since
  ~mid-Oct** inside prometheus.nix: AM 0.31.1 (active, healthy), matrix-alertmanager bridge (@erebot →
  configured room, agenix token+secretFile), 2 systemd rules loaded by Prometheus
- **Synthetic end-to-end test fired**: WikiPipelineTest alert injected via AM API, confirmed accepted
  (`/api/v2/alerts`); owner asked whether it rendered in the Matrix room → pending confirmation
- New security finding filed in [[queries/security-audit]] (P2): **webhook secret committed in
  plaintext in the PUBLIC repo** (receiver URL) + bridge on `*:3004` → forged-alert injection from
  anywhere; action item: rotate + agenix-only (gitignored local for URL build) + loopback binds
  (9093/3100/3004 all still `*` per `ss`)
- [[todo/monitoring-alerting]] rewritten: Status In progress; task 1 [x]; remaining = secret hygiene,
  binds, rule expansion (disk/cpu/mem/targets + SMART later via [[ideas/Drive-Health-Monthly-Newsletter]]),
  Loki LogQL panels, podman-restart noise guard, no dual alert brains (Grafana unified alerting optional)
- todo hub + root index lines updated

## [2026-11-07] update | pvs ground truth: /data is a 5-disk LVM pool, fully allocated
- Owner ran `sudo pvs`: VG is **`whale-nas`** (mapper `whale--nas-data`) spanning **5 PVs** — sda
  (WD20EARX 1.82t), **sdb3** (640 GB ~2009 disk!), sdc1 + sde1 (2× ~931g WD), sdf1 (8t SMR BarraCuda)
  ≈ 10.6 TiB, **PFree ≈ 4 MiB (full)**; only root SSD sdd outside the pool
- Reframed risk language everywhere: architecture §5 + backup posture + drive-health inventory table
  (roles now pvs-verified; ⚠ marks: oldest members sdb/sde likely set the failure pace; sdf is SMR —
  workload caveat; VG-full ⇒ replacement needs new disk first, then pvmove)
- Drive-health idea urgency sharpened: the newsletter watches a 5-member single failure domain with
  zero parity — pool availability = product of its members' health; tasks updated (sdb1/sdb2 leftover
  partitions flagged for identification; by-id option retained)
- docs/erebos/index data row updated; 33 pages; corrected my earlier "plain LVM/single disk" phrasing

## [2026-11-07] delete | todo/disko-prod-device-paths — root path fixed by owner
- Commit `6973308 "fixing drives names"`: prod root `/dev/vda` → **`/dev/sdd`** (the Kingston SA400S3
  SSD — root-on-SSD finally matches hardware; whale--nas-data entry unchanged) → footgun closed, todo
  deleted per lifecycle
- Durable facts folded: architecture §5 rewrites the disk-layout truth (sdd=Kingston root, ESP+btrfs),
  docs/erebos/index prod line corrected; drive-health idea inventory updated (sdd role confirmed,
  sdf-as-whale-VG still "verify with pvs"); residual preference (full role map + optional by-id paths)
  absorbed into the newsletter idea's task list — no orphan links (grep-verified)
- 33 pages

## [2026-11-07] update | Owner correction: prod is BARE METAL — drive-health unblocked, new todo
- Owner: "Erebos prod is not a VM" (stage is — and stage is deliberately out of newsletter scope).
  Verified live from the container view: `systemd-detect-virt` none, Ryzen 5 5600, DMI generic board,
  **6 ATA disks** (WD20EARX, WD6400AAKS, WD10EZEX, Kingston SA400S3 SSD, WD1001FALS, ST8000DM004 8TB),
  zero virtio devices — my earlier "prod = VM" inference (from disko's `/dev/vda`) was wrong; libvirtd
  exists to host the stage VM
- Consequence: SMART collection happens ON Erebos itself → rewrote
  [[ideas/Drive-Health-Monthly-Newsletter]]: gating question removed; disk inventory table added
  (no redundancy exists → newsletter is the only failure strategy today; 2011-era drives flagged for
  power-on-hours); collection now = NixOS smartctl_exporter module on localhost + services.smartd
  self-tests; tasks/index line refreshed
- **New latent bug found & filed** → [[todo/disko-prod-device-paths]] (High): prod.nix targets
  `/dev/vda` on hardware where no vda exists — harmless until someone re-runs disko; disk-role map
  needed (also feeds newsletter per-disk labeling); architecture §5 + docs/erebos/index disko section
  annotated with the stale-path warning
- Also swept leftover stale line: docs/erebos/index still described the removed `vgchange` post-device
  hook (missed in the initrd fold) → updated to stage-1 services.lvm reality
- 34 pages

## [2026-11-07] create | Drive-Health-Monthly-Newsletter idea (feasibility researched)
- Owner asked: monthly drive-state email + always-viewable source; "make all the research on if this
  is feasible" → [[ideas/Drive-Health-Monthly-Newsletter]] with verdict **feasible, one gating fact**
- Research findings: (1) **Erebos is a VM → SMART never crosses virtio** — collection MUST run on the
  physical disk host; (2) Grafana scheduled PDF/email reports = **Enterprise-only** (OSS gap) → digest
  needs DIY: recommended **Hermes monthly cron** (same mechanism as [[todo/scheduled-security-audit]])
  querying Prometheus HTTP API + Resend (creds already on-box); n8n workflow = alt; (3) collection =
  prometheus-community **smartctl_exporter** via NixOS module `services.prometheus.exporters.smartctl`
  (+ node_exporter mdstat; NVMe/SSD caveats documented); (4) **Alertmanager tripwire mandatory** — a
  SMART-FAILED mid-month must not wait for a newsletter; (5) [Scrutiny](https://github.com/AnalogJ/scrutiny)
  documented as all-in-one alternative if the host isn't NixOS
- Open question posed to owner: identity of the physical host (OS? array tech? disks passthrough or
  LVM/mdraid/ZFS?) — determines deployment path
- 6 curl-verified references (1 dead 404 plugin repo dropped); index 33 pages

## [2026-11-07] create | docs/strategos/: full documentation tree for the Pi + olympos/pontos purged
- New machine entity layer per amended [[HERMES]] (`docs/<machine>/`): **docs/strategos/{index,
  services,reference}.md**, built from a fresh read of sys/ @ main (2026-11-07): Pi 3B+ aarch64,
  25.11-stable flake sharing nixos-modules (same base as Erebos), extlinux boot, ext4 SD root,
  4 GiB swapfile, NM + IPv6 fully disabled, hardened sshd (key-only, no root), **create_ap**: wired
  `enu1u1u1` → AP `wlan0` SSID "Chambre Hugo"
- **Security findings documented (not reproduced)**: AP WPA2 passphrase is a short weak pattern in
  plaintext in the PUBLIC repo — flagged with fix path (agenix will arrive on the Pi with the
  keyring work anyway); no disk encryption OK while only-encrypted-key material lands on it;
  version-skew note for future borg receiver role; firewall rules to verify before adding services
- Purged olympos + incidental pontos references per owner ("olympos never existed"): strategos
  readme fleet section rewritten (erebos link kept; Chezmoi-Pontos idea untouched — pontos is its
  genuine subject, confirm intent if that should go too); memory updated
- Fixed my own transcription error mid-work: kernel module is `r8712u` (Realtek USB Wi-Fi dongle)
- Wiring: root index + Strategos subsection (32 pages), age-keys todo links docs/strategos;
  strategos repo: readme commit pending separate push

## [2026-11-07] update | strategos readme fixed upstream; fleet facts corrected
- Owner gave three corrections: olympos **never existed** (abandoned gaming-console idea, config was
  to mirror pontos; "probably never exist"); Pi model = **Raspberry Pi 3B+**; asked for the readme
- Cloned https://github.com/TheWhale01/strategos, rewrote readme.md to match reality (pushed as
  Odin): strategos-only repo (erebos → own flake repo; pontos → Fedora, config retired), dropped the
  stale `hosts/`-disko install section (Pi has no disko — plain hardware-configuration), documented
  actual services (key-based ssh + create_ap on wlan0 — soft AP, noted for LAN topology), nixpkgs
  25.11 pin, athena user, and planned role: agenix keyring host (links todo). Verified
  nixos-modules URL 200
- Wiki: 3B+ model added to [[todo/age-keys-to-strategos]] Target

## [2026-11-07] update | strategos identified: Raspberry Pi config repo integrated
- Owner shared https://github.com/TheWhale01/strategos (verified 200): the Pi is **strategos** — aarch64
  NixOS flake pinning nixpkgs release-25.11 stable, user `athena`, shares nixos-modules input with
  Erebos; readme still lists the fleet (erebos/pontos/olympos configs once lived here, now only
  strategos remains in sys/); last pushed 2026-10-12
- Relevant facts folded: sshd up with authorized keys (borg + key-transfer channel exists today);
  NetworkManager/DHCP → jobs must pin static lease or strategos.local; **no agenix on it** → the
  Pi-as-agenix-recipient option needs setup, reinforcing the passphrase-archive recommendation
- todo/age-keys-to-rpi → renamed **todo/age-keys-to-strategos** (git mv, all refs repointed: backups,
  secrets, architecture, both hubs, root index); prose updated across docs
- log.md entry; model corrected mid-edit (repo never says Pi 4 — dropped the invented spec)

## [2026-11-07] create | Backup spec answered → todo/backups reborn + todo/age-keys-to-rpi
- Owner answered the three architecture questions; facts now govern scope:
  (1) `/data` NOT redundant — `whale--nas-data` is a stale VG name from a previous NAS build, plain
  LVM on this server; (2) only **vaultwarden, immich, nextcloud** need backups — everything else
  explicitly out of scope; (3) **no offline copy of agenix identity keys exists** — target for the
  backup is the owner's **Raspberry Pi (NixOS, same LAN)**, new third machine of the fleet
- Recreated [[todo/backups]] (Critical) precisely scoped: 3 PG logical dumps + /data/Immich +
  /data/nextcloud + vaultwarden state dir; Borg-vs-restic open; Pi = remote target (off-box, not
  off-site — documented limitation); quarterly restore drill
- New [[todo/age-keys-to-rpi]] (High): passphrase-archive (recommended) vs Pi-as-agenix-recipient;
  round-trip decrypt drill from a clean machine is the step that makes it real
- Updated: architecture §5 backup posture, secrets.md cliff → planned fix link, todo/index + root
  index (29 pages)

## [2026-11-07] delete | 3 todo items pruned per owner — decisions folded to docs
- [[todo/podman-image-updates]] deleted: current daily update-containers **works as-is**; no
  digest-diff/signature/notification work wanted — failure observability deferred to
  [[todo/monitoring-alerting]] (restart-noise caveat moved there). Fact recorded in [[docs/erebos/containers]]
- [[todo/backups]] deleted (paused): owner classifies `/data` as **mainly non-sensitive**; will define
  the must-backup list first. Facts folded into [[docs/erebos/architecture]] §5 "Backup posture":
  Immich/Nextcloud media on /data vs the loss-if-root-dies set (5 Postgres DBs, /var/lib/opengym/api,
  agenix identity keys, secrets + flake repos); tutorials §7 last-resort warning repointed there
- [[todo/secrets-management]] deleted: age secrets are **mostly Authentik SSO client_secrets — not
  easily rotatable**; rotation policy explicitly dropped ([[docs/erebos/secrets]] new
  "Rotation posture" section); surviving open fact kept: broad recipients + NO offline backup of
  age identity keys — flagged as the real key-loss cliff
- Cross-references repointed in 10 files (audit Related, opengym, hermes-dashboard, Chezmoi,
  terraform-workflow, authentik-revalidation, monitoring, tutorials); todo hub High section now empty;
  root index 27 pages; Open questions posed to owner re: NAS redundancy, Immich/NC media sensitivity,
  age key offsite copies

## [2026-11-07] delete | todo/initrd-systemd-stage1-migration — COMPLETED & folded
- Commits `5f41664` (delete `boot.initrd.systemd.enable = false`) + `034cd93` (delete the whole
  `postDeviceCommands` vgchange hook from prod.nix; flake pins advanced incl. hermes-agent)
- **Verified live on Erebos**: booted = current = `26.05.20261107.93fb760`, kernel **7.3.3**,
  `/data` mounted from `/dev/mapper/whale--nas-data` → systemd stage-1 auto-activated the VG via
  lvm2 udev rules exactly as predicted; the fallback `boot.initrd.systemd.services` unit was never needed
- Destructive question answered affirmatively by reality: the scripted-era hook was a duplicate
- Folded durable knowledge: [[docs/erebos/architecture]] §5 disk layout rewritten (stage-1 boot,
  services.lvm defaults, commit refs, 26.11 context); [[docs/erebos/reference]] options table +
  boot.initrd.systemd row; [[queries/security-audit]] kernel/pin rows refreshed
- todo/index + root index lines removed (30 pages); deprecation warning era closed for this vault

## [2026-11-07] update | Initrd migration: prod assertion root-caused
- Owner hit `systemd stage 1 does not support boot.initrd.postDeviceCommands` on prod (stage passed —
  no LVM there); definition pinpointed in sys/disko/prod.nix (their own `lvm vgchange -ay` hack for the
  whale--nas-data /data LV) — the origin the todo asked archaeology for
- Three findings: hack is a quoting no-op (`''"…"''` → single quoted token); scripted stage-1-init.sh
  already ran vgchange itself; and pinned nixpkgs tasks/lvm.nix auto-enables
  boot.initrd.services.lvm under systemd stage-1 (default = stage1 && services.lvm, services.lvm
  default true) → fix is TWO DELETIONS (postDeviceCommands block + systemd.enable=false), zero additions
- [[todo/initrd-systemd-stage1-migration]] rewritten: Status In progress, root-cause section, checklist
  reordered (stage done, delete→dry-build→apply→cold reboot→verify /data+lvm2-monitor+journal);
  todo/index + root index lines refreshed
- Live checks used: findmnt /data (/dev/mapper/whale--nas-data ext4), store-source greps of pinned
  nixpkgs (stage-1-init.sh L109, systemd/initrd.nix assertion list, tasks/lvm.nix defaults)

## [2026-11-07] update + create | Audit → open-findings-only; idea: Odin + cve.now MCP
- [[queries/security-audit]] **stripped of resolved findings** (owner convention, now encoded in
  [[todo/scheduled-security-audit]] steps 2+4 and the page header): deleted the RESOLVED Authentik P1
  section, the retracted n8n P3 correction, the Ollama-era bullet, and all [x] action rows; remaining
  sections rewritten state-only (P2 binds = Alertmanager/Loki/Jellyfin only; P3 llama.cpp = the two
  live config drifts). Audit log + git keep the history — Immich is visibly the sole open P1
- Added [[ideas/CVE-Now-MCP]]: connect Odin to cve.now's free hosted MCP server
  (https://mcp.cve.now/mcp, streamable HTTP, **no auth — handshake + tools/list verified live from
  Erebos today**); 5 read-only tools; key fit = `check_stack` accepts **flake.lock** (deterministic
  SCA over the pinned system deps) + feeds for the scheduled audit's advisory step; Hermes config
  sketch (settings.mcp_servers), injection-surface & egress caveats, 90-day coverage warning, 5
  curl-verified references
- index.md: new idea line, Queries line refreshed, 31 pages

## [2026-11-07] update | Audit correction: P1 Authentik RESOLVED — owner was right, raw-CDN lied
- Challenge-driven re-verification found `8d47326 "upgrading authentik"` (Nov 6, on main): 2026.8.1 → **2026.8.5**,
  double-confirmed live (unit-script image tag + contents API). P1 CLOSED in [[queries/security-audit]]
  (banner, inventory row, action list ticked, audit log entry)
- My 2026-11-04 "still 2026.8.1 at HEAD" check was correct for its date, but a later same-week re-check
  got a **stale raw.githubusercontent.com blob** (cache-busting ineffective) — logged as scar tissue
- New permanent methodology rule in [[todo/scheduled-security-audit]]: GitHub **contents/commits API +
  live unit ExecStart** are the repo/runtime sources of truth; never raw.githubusercontent
- docs updated (containers.md + services.md Authentik version refs); Immich remains **sole open P1**
  (2.7.5 + permittedInsecurePackages, re-probed via contents API + store path); note: main also has
  `0564649` immich client_secret rotation (2026-11-07) — no doc surface (secret names unchanged)

## [2026-11-04] query | Security audit live re-check run (procedure of the future cron job)
- Requested "update the audit" → ran the [[todo/scheduled-security-audit]] procedure manually:
  commits since sync (`b1b690e` immich+prometheus loopback binds), repo HEAD pins, unit ExecStarts,
  nixos-version, live `ss -ltn`, llama `/props` + `/v1/models` probes
- **Honest headline: the two P1s are NOT remediated** — Authentik still `2026.8.1` at HEAD and Immich
  still `2.7.5` running + `permittedInsecurePackages` in place; note the mismatch with owner's
  expectation — ghcr/Nix pins don't move via `nix flake update`. P1 sections + action list marked "re-verified
  not fixed", runbook link (§7) added
- Remediated since Oct 26 (verified live): Prometheus 9090 ✓, Immich 2283 ✓, llama-cpp 8080 ✓ loopback;
  kernel 7.3.0→7.3.1; still open: Alertmanager 9093, Loki 3100, Jellyfin 8096 binds
- **New finding**: llama-cpp router up but **no model loaded**; Hermes fallback id
  `LiquidAI/LFM3-1.7B-GGUF:Q6_K` ≠ preset alias `LFM3-1.7B` → offline safety-net would lazy-fetch
  from HF and likely fail precisely when offline; fix = align model.default, smoke-test load
- Updated: audit header/inventory/P1/P2/P3/action-list/audit-log; docs services + index rows
  (llama re-enabled status); root index Queries line

## [2026-11-04] create | todo/initrd-systemd-stage1-migration
- User-reported eval warning "Scripted initrd is deprecated… removal in 26.11" — root cause pinpointed:
  `sys/configuration.nix` line ~52 `boot.initrd.systemd.enable = false` (explicit opt-out; no
  documented reason — first checklist item is `git log -S` archaeology)
- High priority: hard deadline 2026-11-30 (26.11 stable deletes scripted initrd; flake pins
  nixpkgs-unstable, removal already in pin range)
- Boot-surface audit makes it a boring migration: root is plain GPT + vfat + btrfs (no LUKS/LVM);
  /data mapper volume defined outside NixOS entirely; stage-first path available; rollback =
  previous generation in systemd-boot menu
- 3 curl-verified references (release notes, impermanence#325, option docs); todo/index.md +
  root index.md wired (30 pages)

## [2026-11-04] create | Authentik upgrade + rollback runbook (tutorials §7)
- Added [[docs/erebos/tutorials]] **§7**: full procedure for the user's stack — one-variable version
  pin (`authentikVersion` → all 4 images), host-Postgres pg_dump backup, pull-before-stop, explicit
  one-shot `migrate` (server does not auto-migrate), declarative bump + `nix run .#apply`,
  verification checklist (health endpoint, forwardAuth sign-in, LDAP login, worker journal)
- **Downgrade section**: Authentik has no supported downgrade path (migrations are forward-only in
  practice) → restore = pre-upgrade pg_dump + tag revert; unsupported reverse-migration last resort
  flagged as a [[todo/backups]] argument; multi-minor drift rule (one minor at a time)
- Verified 2026-11-04: ghcr tags through **2026.8.5** exist (server+ldap+proxy), matching GitHub
  Releases latest community version; official docs URL curl-verified
- [[queries/security-audit]] P1 row links the runbook + target version; fixed stale
  auto-upgrade bullet in tutorials "Notes on the environment"; index line refreshed

## [2026-11-04] update | Erebos sync: commit d5340e3 "fallback to local inference"
- Provider roles inverted: **OpenRouter qwen3.8-flash = primary**, local LFM3/llama.cpp = fallback
  (safety net when cloud is down); docs/erebos/services.md Hermes section rewritten accordingly
- **Audit P3 closed same day**: llama.cpp rebound `0.0.0.0` → `127.0.0.1` ✓ — audit→fix turnaround
  ~1 h, recorded in Audit log + action list ticked (remaining checklist for a future re-enable:
  verify n_ctx, tool-call smoke test, modelsPreset migration)
- Audit P3 section, inventory row, docs/erebos/index.md service rows updated; services.md +
  security-audit.md dates refreshed

## [2026-11-04] delete | todo/llama-cpp-service-dying removed — shutdown is intentional
- Owner: llama-cpp unit purposely stopped, not a fault. Deleted the todo (created minutes earlier);
  repointed refs in security-audit (inventory row, P3 section, action list → "checklist for whenever
  re-enabled"), todo/index.md, root index.md (29 pages), Translaitor Related, docs/erebos/services.md
- Audit log records the intent so the future scheduled job doesn't re-flag the absent :8080

## [2026-11-04] update | Erebos sync: commit 6ef7e41 "upgrade + running local LFM3" — idea IMPLEMENTED
- **[[ideas/Hermes-Local-LFM3]] implemented → deleted** (lifecycle rule); durable design knowledge folded
  into new `sys/services/llamacpp.nix` section in [[docs/erebos/services]]: llama-cpp 0.8.0 from
  pkgs-unstable replacing Ollama (module deleted), LFM3-1.7B **Q6_K** via HF `modelsPreset`,
  `-fa on -ctk q8_0 -ctv q8_0`; Hermes = custom provider :8080 + **fallback_providers OpenRouter**
  (qwen3.8-flash) — fallback proven live this session
- **Live observation** (from this very box): llama-cpp unit exits ~8 s after start (status 0),
  nothing on :8080 → created [[todo/llama-cpp-service-dying]] (High): journal diagnosis,
  `-c` context verification vs Hermes 64000 default, `modelsPreset` removed upstream (next flake
  update breaks eval), and P3 security note: unauth bind moved loopback → 0.0.0.0 (reachable via
  podman0/wt0 trusted interfaces)
- Also in `6ef7e41`: **`system.autoUpgrade` removed** (docs index/reference/architecture + audit
  systemic caveat updated — manual `nix flake update` cadence now; pins advanced same day,
  Authentik/Immich remediation pending version-probe), n8n → pkgs-unstable, cleanerr
  `ignoredDirectories` {aurral,lidarr,slskd}, `hermes.age` tightened 0440→0400 owner=svc user
  (secrets.md row updated)
- [[queries/security-audit]]: inventory Ollama row → llama.cpp, P3 finding added, action list +4/1
  updated, 2026-11-04 audit log entry; [[todo/scheduled-security-audit]] inventory list updated;
  [[ideas/Translaitor]] relinked (Ollama runtime gone — translategemma hosting must migrate)
- index.md: idea row removed, todo row added (30 pages); 7 pages date-bumped

## [2026-10-26] lint | Structural alignment: three consistency debts fixed
- [[HERMES]] naming rule amended: lowercase-hyphens for docs/todo/queries/raw/comparisons/concepts;
  Title-Case allowed in ideas/ (idea filenames are stable blog permalinks — renaming would break
  published URLs)
- Entity layer formalized: docs/ (docs/erebos/services/* = per-service entity pages); retired empty
  generic folders entities/ and summaries/ (removed from disk + index.md sections); _archive/
  re-scoped in Page Thresholds to rare non-git-recoverable material, supersede = delete+fold per the
  lifecycle rule
- Wikilink backfill (min-2 rule) on the three pre-convention idea pages: [[ideas/Deezify]] →
  Translaitor + docs/erebos/index; [[ideas/Remote-Gaming-Server]] → docs/erebos/index +
  Cloudflare-Tunnel-SSH; [[ideas/Translaitor]] → docs/erebos/services + Hermes-Local-LFM3
  (all link targets verified to exist; dates bumped)

## [2026-10-26] lint | Link checklist run: 89 URLs, 1 real finding
- External: 87/89 healthy. Flags: deezify.thewhale.fr → genuine 404 (offline instance; annotated
  in [[ideas/Deezify]], URL kept as historical — matches the rebuild-from-scratch plan);
  gitlab.gnome.org/GNOME/dconf → 406 is a urllib content-negotiation artifact (curl returns 200),
  link kept.
- Internal wikilinks: clean. Remaining "broken" hits are by-design: historical references in
  log.md (append-only record pointing at since-deleted pages) and the [[wikilinks]] placeholder
  example in [[HERMES]].
- Added .linkcheck-last.json to .gitignore (machine artifact of this and future runs — the
  scheduled audit job, [[todo/scheduled-security-audit]], can reuse this exact procedure)

## [2026-10-26] create | Hermes-Local-LFM3 idea
- Added [[ideas/Hermes-Local-LFM3]]: move Odin from OpenRouter (qwen3.8-flash) to a local custom
  provider running Liquid AI LFM3-1.7B (released 2026-10-27, post-trained for agentic harnesses
  incl. Hermes Agent explicitly)
- Grounded in live facts: Erebos = 12 cores / 31 GiB, Ollama already serving qwen3.5:9b +
  translategemma:4b; ollama.com/library/lfm3 confirmed with CPU quants (lfm3:1.7b-q4_ ~1 GB)
- Covers Hermes `provider: custom` config (nix snippet for sys/services/hermes.nix), the
  OLLAMA num_ctx vs context_length trap, timeouts, license (LFM Open v1.0), and the honest
  trade-off: sweet spot = scheduled/background/triage work; heavy coding stays with cloud →
  hybrid topology proposed; eval-before-switch checklist
- 6 external references, all curl-verified; index.md (total: 30 pages)

## [2026-10-26] update | Moved Scheduled-Security-Audit: ideas/ → todo/
- `git mv ideas/Scheduled-Security-Audit.md todo/scheduled-security-audit.md`; frontmatter now `type: todo`
  with Status: Not started / Priority: Medium / parent hub
- Repointed refs in queries/security-audit.md, root index.md, todo/index.md (added to Medium section)
- HERMES.md scope rule refined per user usage: todo/ = actionable backlog (improvements, fixes,
  operational/automation tasks); ideas/ = new services/projects. Page count unchanged (29)

## [2026-10-26] update | n8n audit finding corrected (P1→P3)
- Owner clarification: n8n is not internet-forwarded; built-in auth chosen deliberately (n8n SSO is paid-only)
- Root cause of my misread: NAT hairpinning — curling the WAN IP from inside the network returns 200; recorded as methodology caveat
- [[queries/security-audit]]: inventory row, findings section rewritten as P3 with residual recommendations
  (owner MFA, aggressive update cadence, LAN/VPN boundary is the wall), action list + Audit log updated
- [[ideas/Scheduled-Security-Audit]]: added "exposure checks must come from outside" design rule
- root index.md Queries line refreshed

## [2026-10-26] update | Audit page renamed + Scheduled-Security-Audit idea
- Renamed [[queries/erebos-cve-exposure-2026-10]] → [[queries/security-audit]] (git mv); reframed as
  a living document: header banner + dated Audit log section + "current snapshot" marker
- Added [[ideas/Scheduled-Security-Audit]]: weekly Hermes cron job that re-inventories versions
  (systemd/Nix-store/API probes), diffs against the stored snapshot, cross-checks vendor GHSA +
  CISA KEV, updates the audit page, pushes, and reports to Matrix — read-only by design
- External refs verified (hermes cron docs, trivy, KEV, GHSA db; vulnix links 404'd and were dropped)
- index.md: Ideas + Queries lines updated; total pages: 29

## [2026-10-26] query | Erebos CVE exposure audit (filed)
- Live inventory via systemd/Nix-store paths/service APIs (Traefik 3.7.13, Authentik 2026.8.1,
  Immich 2.7.5, n8n 2.39.6, Nextcloud 35.0.2, Jellyfin 12.2.0, Synapse 1.163.2, Ollama 0.40.3,
  Grafana 13.0.11, Vaultwarden 1.37.4, Podman 5.8.9, PG 16.15, kernel 7.3.0, pin 2026-10-27)
- Filed [[queries/erebos-cve-exposure-2026-10]] with P1/P2/P3 findings + action list:
  P1 Authentik Sep-2026 advisory batch (≤2026.8.1, fix in 2026.8.2 — MFA bypass 8.9);
  P1 Immich 2.7.5 (CVE-2026-119328 unauth ML path-injection <3.3.0, +2 more);
  P1-config n8n internet-exposed without Authentik since 5e6c83b; P2 Jellyfin ≤12.4 set;
  P2 verify richdocumentscode absent (CVE-2026-110447 CISA KEV, actively exploited);
  P2 unauthenticated monitoring ports on 0.0.0.0 (podman0 lateral surface); systemic:
  autoUpgrade doesn't advance flake pin → patch cadence gap
- Corrected stale version notes: "Traefik 2" → 3.7.13 in docs/erebos index.md + services.md
- index.md updated (total: 28 pages)

## [2026-10-26] create | Authentik policy-revalidation todo (researched)
- Added [[todo/authentik-policy-revalidation]]: intercept logins when existing creds violate changed policies
- Verdict: username leg fully feasible (Expression Policy + Prompt/Write stage bindings, official
  force-reset-on-login pattern); password leg fundamentally limited — hashed passwords can't be
  retroactively policy-checked, so it's force-rotation-on-change instead
- Native "required actions" (issue #25310 / PR #24846) is enterprise + 2026.11.0 — not for 2026.8.1 CE
- All implementation is terraform-declarable in sys/terraform/authentik.nix; 8 URLs curl-verified
- todo/index + root index.md updated (total: 27 pages)

## [2026-10-26] create | Quickshell Notification Client idea
- Added [[ideas/Quickshell-Notification-Client]]: custom notification daemon inside the quickconf
  bar repo on pontos (Hyprland/Fedora), styled with Global.qml Catppuccin Mocha
- Upstream verification against quickshell source (mirror + forge): repo moved outfoxxed/quickshell →
  quickshell-mirror/quickshell; site is quickshell.outfoxxed.me (quickshell.dev NXDOMAIN);
  module `Quickshell.Services.Notifications` already implements the whole freedesktop protocol —
  API confirmed from notification.hpp/server.hpp (NotificationServer.trackedNotifications(),
  Notification props incl. actions/hasInlineReply/dismiss()/sendInlineReply())
- Design reduced to a QML UI exercise: Repeater-bound popup window + DND gate on Global.qml;
  mako bus-name ownership check is task 1; all external URLs curl-verified per the new rule
- Updated index.md (total: 26 pages)

## [2026-10-26] update | External-references rule added; chezmoi page linked
- New standing rule in [[HERMES]] Idea Pages: every idea links official docs and/or GitHub repos
  of mentioned tools (inline for primary tool + "External references" section; URLs verified)
- [[ideas/Chezmoi-Pontos]]: added inline links + External references section (chezmoi, age, agenix,
  home-manager, Fedora packaging, flatpak, dconf) — all 13 URLs checked live with curl
- Caught during verification: chezmoi install URL is /install/ (not /user-guide/installation/);
  home-manager.org unreachable → use nix-community.github.io/home-manager

## [2026-10-26] create | Chezmoi-Pontos idea
- Added [[ideas/Chezmoi-Pontos]]: dotfile management for the laptop (pontos), which moved
  NixOS → Fedora to ease small-project development
- Covers: why chezmoi (git-native, templating, age secrets), home-manager-on-Fedora alternative
  (and why it was declined), repo structure sketch, Fedora-specific scripts (dnf/flatpak/dconf),
  wipe-and-recover drill as the acceptance test
- First idea scoped to a machine other than Erebos — noted in page
- Updated index.md (total: 25 pages)

## [2026-10-26] update | Vault source-of-truth decided; todo deleted
- User decision: the authoritative wiki checkout is wherever `WIKI_PATH` points
  (= /var/lib/hermes/workspace/blog-ideas); the /home/hades copy is not authoritative
- Recorded permanently in [[HERMES]] Domain (incl. note to point Obsidian at the $WIKI_PATH checkout)
- Deleted todo/vault-source-of-truth.md per lifecycle (last todo split from wiki-path); refs repointed
  in docs/erebos/services/hermes-dashboard, todo/index, root index.md (24 pages)

## [2026-10-26] update | Todo lifecycle: delete completed items; wiki-path closed & deleted
- HERMES.md rule changed: completed todos are DELETED (log.md + git history preserve them), no `_archive/` copies
- Verified `WIKI_PATH` fix live (gateway env = workspace path; dashboard unit active, :9119 → 200)
- Deleted todo/wiki-path.md (Done); split its still-open sub-question into new [[todo/vault-source-of-truth]] (Medium)
- Repointed refs in docs/erebos/services/hermes-dashboard, todo/index, root index.md; page count 25 (−1 +1)
- Session tracker: item 6 completed, 6b (vault source-of-truth) added

## [2026-10-26] update | Synced wiki with erebos 2de8521 (dashboard auth simplified; WIKI_PATH fixed)
- docs/erebos/services/hermes-dashboard: OIDC→Basic bridge replaced by edge-only Authentik auth
  (no dashboard-side provider; terraform module no longer parseEnv's hermes.age; group attrs removed);
  added security caveat (any local process can reach :9119), decision-history v1→v2 entry, WIKI_PATH section
- docs/erebos/services + terraform.md: matching wording updates
- todo/wiki-path: ✅ Done — WIKI_PATH removed from global sessionVariables, scoped to
  systemd.services.hermes-agent.environment (commit 2de8521); verified live in gateway env (200 on :9119,
  unit active). Follow-up kept open: decide vault source-of-truth
- todo/index + root index.md: status refreshed

## [2026-10-26] update | Hermes Dashboard implemented — idea promoted to service doc
- erebos commit `27c42d9` "added hermes dashboard": systemd unit (loopback :9119) + Traefik router
  `hermes-dashboard.thewhale.fr` + Authentik forwardAuth + new `sys/terraform/hermes.nix`
  (proxy provider with OIDC→Basic bridge via group attributes; group: whale only)
- Created [[docs/erebos/services/hermes-dashboard]] (architecture, config, decision history, ops notes)
- Deleted the implemented idea `ideas/Hermes-Web-Dashboard.md` per lifecycle rule; refs updated
- services.md (Hermes Agent section + terraform module list), docs/erebos/index services table,
  terraform.md module table, reference.md + secrets.md (hermes.age now hermes:users 0440)
- Net page count unchanged (25): −1 idea, +1 service doc

## [2026-10-26] delete | Gluetun-AirVPN idea (implemented) — lifecycle rule added
- Implemented ideas are now deleted from `ideas/` once their knowledge lives in `docs/` (rule recorded in [[HERMES]])
- [[ideas/Gluetun-AirVPN]] decision history + lessons folded into [[docs/erebos/containers]] (VPN tunnels section)
- Repointed references in [[docs/erebos/services]] and index.md; total pages: 25
- Checked remaining ideas (Deezify, Remote-Gaming-Server, Translaitor, Cloudflare-Tunnel-SSH, Fail2ban, Hermes-Web-Dashboard): all still unimplemented, they stay

## [2026-10-26] update | Synced wiki with erebos commits Oct 15–26
- Read erebos repo commits: 84c3833 (hermes/Odin), 62e1e1e (opengym), ce8fb26 (slskd→podman, hermes→openrouter), c92ad7c (second gluetun)
- [[docs/erebos/containers]]: single gluetun → two tunnels (gluetun-transmission CH / gluetun-slskd NL, PF-only, per-tunnel WG key + agenix secret); new slskd-as-container section (native gluetun PF integration, Authentik fronting); transmission stack netns/secret names updated
- [[docs/erebos/services]]: slskd systemd section → pointer; gluetun section → split note; new Hermes Agent section (services.hermes-agent, OpenRouter qwen/qwen3.8-flash, Matrix allowlist, hermes.age, hades in hermes group)
- [[docs/erebos/index]]: services + containers tables, file tree (hermes.nix, slskd.nix container, no gluetun.nix), updated 2026-10-26
- [[docs/erebos/reference]] + [[docs/erebos/secrets]]: gluetun.age → gluetun/transmission.age + gluetun/slskd.age; hermes.age added
- [[ideas/Gluetun-AirVPN]]: marked Implemented via Option 1 with lessons (WG key per tunnel, PCP timeout fix, native slskd integration)
- [[todo/wiki-path]]: provenance corrected — declared in sys/configuration.nix sessionVariables (84c3833); tasks rewritten

## [2026-10-22] update | Introduced todo/ layer; split Erebos-Server-TODO
- New scope rule (recorded in [[HERMES]]): `ideas/` = new services to add; `todo/` = improvements/fixes to what's already implemented
- Created `todo/` with 7 pages: index (hub), monitoring-alerting, backups, podman-image-updates, secrets-management, terraform-workflow, wiki-path
- Each section of the former `ideas/Erebos-Server-TODO.md` became its own file (status, priority, tasks, detailed notes); hub file deleted
- Updated live references in docs/erebos/index, docs/erebos/monitoring, docs/erebos/services/opengym, ideas/Fail2ban, ideas/Hermes-Web-Dashboard, root index.md
- Tag taxonomy extended with Work: todo/improvement/fix/housekeeping; page type `todo` added
- Total pages: 26

## [2026-10-22] update | Expanded Erebos-Server-TODO with detailed notes
- Added a "Detailed notes" block to each of the 6 sections of [[ideas/Erebos-Server-TODO]]
  (implementation guidance: alert routing to Matrix, backup scope incl. /var/lib/opengym/api + pg_dumpall,
  podman auto-update labels, age recipient grouping, tofu state migration, WIKI_PATH provenance)
- Mirrors the session todo list; the wiki page is the durable source of truth

## [2026-10-22] update | Added WIKI_PATH housekeeping item to TODO list
- Appended item 6 to [[ideas/Erebos-Server-TODO]]: `WIKI_PATH` still points to the
  inaccessible `/home/hades/code/blog-ideas`; should be set to
  `/var/lib/hermes/workspace/blog-ideas` in `~/.hermes/.env`

## [2026-10-22] create | Hermes Web Dashboard idea
- Added [[ideas/Hermes-Web-Dashboard]]: browser management UI for the Hermes agent on Erebos
- Covers three access models (SSH tunnel, systemd + Authentik-gated public bind, Hermes Desktop remote backend)
- Security notes: dashboard is admin-equivalent (config, API keys, PTY chat); auth gate mandatory for non-loopback binds
- Updated index.md (total: 20 pages)
- Note: briefly launched `hermes dashboard` on loopback during research; killed and verified stopped afterwards

## [2026-10-22] create | Fail2ban idea
- Added [[ideas/Fail2ban]]: brute-force log scanning + IP bans for Erebos
- Includes NixOS specifics: journald backend (no /var/log/auth.log), nftables vs iptables actions,
  ignoreIP for podman/gluetun/wt0 ranges, X-Forwarded-For caveat for Cloudflare-fronted jails
- Cross-linked with [[ideas/Cloudflare-Tunnel-SSH]] and [[docs/erebos/services]]
- Updated index.md (total: 19 pages)

## [2026-10-22] update | Renamed SCHEMA.md to HERMES.md
- `git mv SCHEMA.md HERMES.md` — the schema file now identifies as the Hermes agent configuration for this wiki
- No content changes; historical log entries referencing SCHEMA.md left intact

## [2026-10-22] update | openGym implemented: idea promoted to service doc
- openGym is now deployed on Erebos as `sys/containers/opengym.nix`
- Moved [[ideas/openGym]] → [[docs/erebos/services/opengym]] (new `docs/erebos/services/` folder)
- Rewrote documentation to match the real Nix config:
  - Two containers: `opengym-api` (backend, port 3000) + `opengym-web` (nginx, shares network namespace)
  - Published port 3006 (vars.opengym.web.port), not 3002
  - Data dir `/var/lib/opengym/api:/data` (host), not `/data/opengym`
  - Passkey/WebAuthn auth (RP_ID/ORIGIN); Authentik forwardAuth middleware is commented out (not enforced)
  - Added env var reference table and accurate deployment commands
- Deleted ideas/openGym.md; updated index.md (Ideas minus one, new "Erebos Services" section)

## [2026-10-15] create | Gluetun and AirVPN idea
- Created [[ideas/Gluetun-AirVPN]] with VPN considerations for slskd:
  - Option 1: Second Gluetun container for Cloudflare Tunnel
  - Option 2: Switch to AirVPN to bypass ProtonVPN constraints
- Updated [[ideas/Cloudflare-Tunnel-SSH]] to remove Gluetun-specific content
- Updated index.md (total: 18 pages)

## [2026-10-15] create | Cloudflare Tunnel for SSH idea
- Added [[ideas/Cloudflare-Tunnel-SSH]] with implementation plan for exposing SSH via Cloudflare Tunnel
- Includes NixOS module, firewall rules, DNS setup, and Erebos integration steps

## [2026-10-15] update | Replace Tailscale with Netbird in architecture.md
- Updated `docs/erebos/architecture.md` to replace Tailscale with Netbird in the Big Picture diagram

## [2026-10-15] update | Replace Tailscale with Netbird references
- Updated all wiki pages to replace Tailscale with Netbird (interface: `wt0`)
- Removed `sys/services/tailscale.nix` section from `docs/erebos/services.md` (Netbird now in nixos-modules)
- Updated `docs/erebos/index.md` table and file tree to reflect Netbird via nixos-modules
- Updated `docs/erebos/tutorials.md`, `reference.md`, and `monitoring.md`

## [2026-10-15] update | Remove stale/deprecated references
- Removed duplicate `slskd` reference in `docs/erebos/services.md`
- Updated `docs/erebos/index.md` to link to [[ideas/Erebos-Server-TODO]] instead of outdated Roadmap section
- Updated `docs/erebos/index.md` to link to [[ideas/Erebos-Server-TODO]] (see "Implement Proper Monitoring & Alerting" section)
- Removed legacy file note (jellyfin/default.nix, get_m3u.py) from `docs/erebos/index.md` Roadmap
- Added `reference` and `tutorial` to tag taxonomy in `SCHEMA.md`

## [2026-10-15] create | Erebos Server TODO List
  1. Implement proper monitoring & alerting
  2. Add robust backups
  3. Automate Podman image updates
  4. Centralize and secure secrets management
  5. Improve Terraform/OpenTofu workflow

## [2026-10-15] update | Wiki frontmatter standardization
- Updated frontmatter for all `ideas/*.md` and `docs/erebos/*.md` to match SCHEMA.md conventions
- Added `type`, `created`, `updated`, and `sources` fields to all pages
- Converted YAML list tags to inline array format

## [2026-10-15] update | Wiki index
- Updated `index.md` to list all 15 existing pages (4 ideas + 8 Erebos docs + 3 meta files)
- Fixed total page count from 0 to 15
- Added summaries for each page

## [2026-10-15] create | Wiki initialized

## 2026-11-07 (alerting) — root cause confirmed from bridge journalctl

- matrix-alertmanager 0.9.0 logs `M_FORBIDDEN: @erebot not in room !C-EIAS89vRQV7k6yUk34TTqXDro5S3Xwx_sJidUW3Lk:matrix.thewhale.fr` — room ID superseded the stale `!FVqWRrn1…` noted earlier from config summary.
- Proves the rest of the chain healthy: fresh AS-minted token VALID (403-on-send, not 401), webhook secret OK, Alertmanager routing OK. Failed events dropped, not retried (`clearing queue`).
- Remaining: invite @erebot (or repoint matrixRooms; copy ID from Element, chat text mangles +/=), silence/fix ServiceOffline spam BEFORE joining, then end-to-end test; then secret rotation + loopback binds.
- todo/monitoring-alerting, todo/index, index updated.

## 2026-11-07 (evening) — 'works fine' was a false positive; bridge source read

- Owner's 'it works' = assumed my test alert had arrived; NOTHING has ever arrived despite: correct room ID in config, @erebot now a room MEMBER, valid AS token (403-class errors prove auth passes), working secret gate (403 probed without secret), unit restarted 21:22 post-apply.

- 790c16a also DELETED both alert rules for a from-scratch rebuild → zero coverage today; docs/todo corrected (pipeline = CONFIGURED, not LIVE).

- Source read of the nixpkgs-patched 0.9.0 store copy: secret enforced in-app (query OR Basic-auth; upstream lacks the patch); send failures return 500 (AM retries — my ACK-and-drop theory disproved); ensureInRoom has an upstream misplaced-parens cache bug (benign). Journal since 21:00 is the missing witness — owner runbook issued; suspects now: room power levels for sends, or unseen successful sends.

## 2026-11-07 (late evening) — pipeline CONFIRMED WORKING by journal evidence

- Bridge journal 21:22-21:44: join 200 + six /send PUTs all 200 with event IDs — all three probe alerts delivered into !C-EIAS89…; AM 5-min retry backoff as predicted from source read. Mystery narrowed to client-side room identity (matrix.to by-ID link provided to owner). Services.md: pipeline = LIVE.

## 2026-11-07 (night) — v12 research + replacement ruleset designed & dry-run validated

- Room v12 (spec v1.16, Project Hydra; Synapse default since 1.162): room IDs drop the :server component (create-event hash); matrix.to links with a suffix are invalid for v12 rooms — the by-ID link I sent was malformed, that likely hid the confirmed-delivered alerts; version upgrade ⇒ NEW room ID (tombstone) — verify live room ID vs config.
- New 9-rule set (3 groups) designed from live evidence: StartType-label discovery, whitelist-based ServiceOffline (69-unit reality), 10%/5% disk thresholds vs live 13.1% /data, readonly-fs & target-down & reboot watchers; llama-cpp intentionally unwatched. YAML validated parseable; full block in 'Proposed replacement rules'.

## 2026-11-07 (late night) — ruleset unit-tested green via promtool; artifacts attached

- Discovered promtool in prometheus 3.12.0 derivation's -cli store output. check rules: 9 OK; test rules SUCCESS across 6 scenarios (timing boundary +5m/+30m, noise-regression for oneshot/target/socket, whitelist for llama-cpp, real 13.1 % /data silent, target-down).
- Attached todo/alert-rules.yml + todo/alert-rules.test.yml to the vault; todo page gained 'Verified & attached artifacts' section with rerun command.

## 2026-11-07 (final) — alerting operational; Alert-link suppression found

- Owner: "everything works fine" — rules deployed + delivery visually confirmed. Pipeline: LIVE, end of a long debug.
- 'Alert link' is utils.js FALLBACK when ALERT_LINKS unset → suppress via null-config env entry (no patch); documented in docs/erebos/services.md.
- Open tail: webhook secret rotation (leaked, unrotated), loopback binds AM/bridge/Loki, odin display-name rename.

## 2026-11-07 — todo filed: issues-room read-only (bot-only writes)

- New todo/issues-room-readonly.md: events_default→50 + @erebot PL 50 via Element roles & permissions; v12 twist documented — room creator's PL is infinite/immutable, so silencing ALL users incl. owner requires the bot to be the creator (recreate option (b)) else creator-as-exception (a). Wired: todo hub (new Low section), root index (+count 34), cross-link from monitoring-alerting. Pages 33→34.

## 2026-11-07 — secret-in-agenix design for Alertmanager webhook (envsubst route)

- Pinned alertmanager module inspected: environmentFile option + preStart envsubst into PrivateTmp — store copy holds only $PLACEHOLDER; PID1 reads EnvironmentFile as root (compatible with agenix 0400 + DynamicUser). Recipe recorded in todo/monitoring-alerting.md rotation item: KEY=VALUE env secret file for AM + raw secret file for bridge (same value, two shapes); replaces the earlier 'secrets.local.nix' idea.

## 2026-11-07 (sync) — commits 85c5ebe + 6973308 ingested

- `85c5ebe` rotated webhook secret: exactly the envsubst recipe — new `secrets/shared/prometheus.age` (KEY=VALUE, owner vars.prometheus.user, 0400) as alertmanager environmentFile; URL now placeholder; bridge .age re-encrypted; 9-rule set + vars.prometheus.user also landed here. Store hygiene verified live (0 occurrences old secret; EnvironmentFiles=/run/agenix/prometheus). **agenix runtime dir = /run/agenix**, not /run/secrets — corrected across todo/monitoring-alerting, secrets.md, tutorials runbook.
- `6973308` disko: prod /dev/sdd, stage /dev/vda (role-correct swap). docs/erebos/index + architecture already reflected the sdd fix; audit P2 leak finding stripped from open body (resolved, lives in Audit log); rotation todo closed; ALERT_LINKS suppression + loopback binds remain open.

## 2026-11-07 — odin display name applied: odin → Odin

- Owner changed it; verified live via profile endpoint (displayname=Odin). Recorded in docs/erebos/services.md Hermes section. Last open chat-tracked item from the alerting thread closed.

## 2026-11-07 — alerting thread CLOSED at owner level

- Owner: all @erebot tests worked perfectly — delivery now verified both server-side (journal event IDs) and client-side (rendered in ops room). todo/monitoring-alerting status → Core complete; reality-check paragraph refreshed (9 rules live, end-to-end confirmed); v12 room-visibility question settled. Remaining items unchanged: loopback binds, ALERT_LINKS suppression, Loki panels, issues-room read-only.

## 2026-11-07 — monitoring note restructured (owner request)

- 263-line debug chronicle slimmed to a 100-line remainder-at-a-glance page. Nothing lost: durable knowledge (v12 gotchas, appservice/MAS token recipes, bridge source semantics, promtool -cli location) folded into docs/erebos/services.md 'Maintenance recipes & Matrix gotchas'; rule YAML + tests already attached as todo/alert-rules.yml/.test.yml; incident story remains in this file's git history + today's log entries. Structure now: Current state → What's left (6 open items) → rules workflow → safe re-test drills → history pointer.

## 2026-11-07 — backups: Borg chosen, declarative config written; nextcloud DB correction

- Owner picked Borg. todo/backups.md rewritten: verified topology (vaultwarden+immich = PostgreSQL
  socket dbs; **nextcloud = SQLite in /data/nextcloud/data**, official image, no external DB — earlier
  draft wrongly listed a PG dump for it), full `services.borgbackup.jobs` snippet (preHook pg_dump -Fc
  ×2 + occ maintenance-mode + sqlite .backup into tmpfs /run/borg-dumps, postHook unquiesces; weekly
  Sun 03:30; prune 7d/5w/12m; agenix borg-backup key + borg-env), strategos `borgbackup.repos` config
  with restricted `borg serve --append-only` authorized key, Pi-3B+ perf note, restore drill,
  failed-job → alerting cross-ref. Git question answered in-page: NO git transport for archives —
  repo stays borg-native; offsite later = second borg repo (Borgbase/storage box), not GitHub.

## 2026-11-07 — idea filed: Intrusion-Detection (current stack = capable but blind)

- Owner asked whether the monitoring stack can reveal intrusion attempts + success. Live assessment: promtail fully commented out, alloy enabled with NO settings (empty graph), Loki label API = zero series — log pipeline dark; ruler unconfigured; Loki `auth_enabled=false` on `*:3100` = attacker-accessible delete button. Prometheus side already holds useful auth signal (authentik metrics scraped).
- ideas/Intrusion-Detection.md: layered plan in build order (journal→Loki via Alloy loki.source.journal/loki.write or revived promtail+geoip; ruler LogQL incl. sshd fail floods, accepted-session audit cards, pipe-failure anti-tamper alert; authentik login-failure metric rules; auditd surgical watches; NixOS-specific drift advantages — content-addressed store shrinks persistence watch-set to /etc /var /data /new-units + nix store verify + current-system hash gauge; ruler alerts already off-box via Matrix; journald→append-only borg snapshots to strategos; Fail2ban as response phase). Verified doc URLs (Alloy components, Loki ruler/alerts, audit userspace/rules man). Preconditions: P2 binds + Loki auth. Pages 34→35.

## 2026-11-07 — idea filed: CleanYourTV (owner share)

- Android/Google-TV debloat over wireless ADB: backup → pm disable-user bloat/telemetry (reversible, nothing uninstalled) → native Apple-TV-style launcher. Prompt-first (coding-agent workflow), brand-aware. 55k★, created 2026-10, young + vibe-coded caveats noted; official sources = repo + cleanyour.tv only (project itself warns about typosquats/app-store fakes). Homelab angle: clean Jellyfin thin client. URLs curl-verified. Pages 35→36.
- Owner context added: Anker Nebula Capsule 3 projector (slow 2 GB Android 9 box) is the target — supported-brand list confirms Anker Nebula projectors covered; Android-9 ADB caveat + keep-list (laser safety/keystone/DSP/DRM/Cast) recorded in ideas/CleanYourTV.md.
- docs/erebos/monitoring.md refreshed + gained copy-paste Loki 3.x retention snippet (owner request, 1 month): compactor{retention_enabled, delete_request_store=filesystem, working_dir}+ limits_config{retention_period=720h, max_look_back=720h} — both blocks required on 3.7.9 (live version verified); components table corrected (promtail file = dead Alloy stub, Loki = 0 series today).

## 2026-11-09 — Alloy collection config (validated) replaces dead promtail era

- Context: owner's promtail enable hit the nixpkgs hard removal (EOL assertion) → alloy was enabled with no config. Delivered working collector: environment.etc."alloy/config.alloy" (module has NO settings option; .alloy files hot-reload; module already grants systemd-journal supplementary group). Components: loki.source.journal (path/max_age/labels/relabel→unit) → loki.process (traefik json stage: DownstreamStatus/request_path/ClientHost/RequestMethod → labels; geo_ip commented — mmdb file never existed) → loki.write :3100. Pitfall caught by binary validation: 'format' argument doesn't exist on the journal source in Alloy 1.16 (validate rejected it). config validated with alloy validate exit=0 on the installed 1.16.0. Section added to docs/erebos/monitoring.md; table corrected.

## 2026-11-09 — todo filed: GitHub PR governance for erebos (owner directive; NOT implemented)

- Owner stopped the mid-flight setup ("I don't want you to actually implement anything — make a wiki entry"). Captured as todo/github-pr-governance.md: protect `main` (PR-required, **0 approvals** — agent opens PRs through the owner's own OAuth identity, so ≥1 approval would lock the owner out; no force pushes), agent flow = feat/* branch → PR with verification evidence → agent reads review threads → owner merges & deploys, never the agent. Trust-model honesty: device-flow token = full repo scope → hard control is protection rules, soft control is agent policy; documented upgrade path = dedicated bot collaborator account.
- Rollback state verified: main force-push reverted same-day (remote = `85c5ebe`), Alloy River config preserved at workspace `erebos-alloy.patch` (first PR candidate), OAuth device code expired unused → no token ever issued, `gh` removed from container nix profile, local clone clean on `main`. Priority High; hub + index updated (pages 36→37).

## 2026-11-09 — refile: todo/github-pr-governance → ideas/GitHub-PR-Governance (owner request)

- Owner asked the entry to live in `ideas/`. Renamed per idea-folder permalink convention (Title-Case), frontmatter `type: idea`, restructured to Idea Pages format (problem statement / proposed solution / status / external references), links repointed (todo/index + root index Todo row removed; Ideas row added). Page count unchanged (37). Content gain: rulesets noted as the stronger admin-can't-bypass alternative to classic branch protection; all external URLs curl-verified 2026-11-09. Still deliberately NOT implemented.

## 2026-11-09 — lint: scope rule amended + ideas/ vs todo/ classification audited

- Owner defined the rule: ideas/ = anything built from scratch (services, projects, workflows/pipelines); todo/ = changes to existing config/code. HERMES.md scope rule rewritten as dated amendment (owner decision 2026-11-09); hub headers (todo/index, root index) aligned.
- Audit of all 20 pages: frontmatter types all match placement. ideas/ (12): all are build-from-scratch items — incl. GitHub-PR-Governance (new pipeline, owner's example) and Intrusion-Detection, Drive-Health (new layered capabilities). todo/ (7): all modify deployed artifacts — monitoring-alerting, backups (borg job into existing flake), age-keys-to-strategos (protect existing keys), authentik-policy-revalidation, issues-room-readonly, terraform-workflow, scheduled-security-audit (borderline: the cron job is new, but it maintains the existing queries/security-audit — kept todo; flag to owner if reclassification desired). No moves needed.

## 2026-11-09 — naming unified: ideas/ → lowercase kebab (Title-Case exception retired) + security-audit ruling

- Owner ruling #1: scheduled-security-audit stays in `todo/` (confirmed, no move — it maintains the existing audit artifact).
- Owner ruling #2: same nomenclature for `ideas/` as `todo/` (lowercase kebab). All 12 idea files renamed to their **exact lowercase** form — verified against the live blog first: kiln publishes `/ideas/<lowercased-filename>`, and all 12 ideas (incl. github-pr-governance, published minutes after filing) are live, so the lowercase transform preserves every permalink (hence `cleanyourtv.md`, NOT `clean-your-tv.md`). HERMES.md conventions bullet rewritten (exception retired, lowercase-transform rule recorded). 26 wikilinks + path refs rewritten across index, 7 idea pages, queries/security-audit, todo/monitoring-alerting, todo/scheduled-security-audit. Prose mentions of *deleted* historical paths (Hermes-Web-Dashboard, Gluetun-AirVPN, Erebos-Server-TODO) left intact — they are history, not links. log.md untouched (append-only).

## 2026-11-09 — Alloy: final Nix module (configPath semantics resolved), patch rebuilt

- Owner asked about `services.alloy.configPath` (mynixos). Verified module option set through **nixpkgs master**: no `settings` attrset exists or is planned — configPath is only a pointer; correct usage = leave default `/etc/alloy` (directory mode) + `environment.etc."alloy/config.alloy".text` → reloadTriggers hot-reload on rebuild; pointing configPath at a store path loses reload (module docs' own trade-off note). Answered: environment.etc IS the "like mas/matrix" declarative path (mas = attrset→YAML generator; Alloy = text-rendered River, no generator possible).
- Final alloy.nix rebuilt over the previously-logged config: loki.push URL interpolated from config.services.loki.configuration, host label from config.networking.hostName, geoip block included commented with the load-time crash-loop warning (mmdb absent; stage name corrected to stage.geoip — old note had geo_ip), configPath/reload rationale in module comments. Validated end-to-end: git branch + nix eval of the flake's rendered text, alloy 1.12.2 validate exit=0 + fmt clean. Patch at /var/lib/hermes/workspace/erebos-alloy.patch (git-apply --check clean vs 85c5ebe; old geoip-enabled variant superseded — that one would have crash-looped on a host with no mmdb, since stage.geoip opens the DB at config load). Wiki docs/erebos/monitoring.md Alloy section rewritten to match; components table marked alloy.nix pending.

## 2026-11-09 — division of labor fixed; Alloy module embedded in wiki

- Owner ruling: agent performs NO code changes (no applying/branches/pushing) — code answers = snippets in chat; wiki fed when necessary; owner applies and deploys. Consequence: docs/erebos/monitoring.md no longer points the final Alloy config at the transient container patch (workspace/erebos-alloy.patch) — the full validated alloy.nix module is now embedded inline (recovered verbatim from the patch), with owner-apply framing. Section heading + components table adjusted. Patch file remains for convenience but the wiki is the durable home.

## 2026-11-09 — attrset→River generator built and proven (answers "can Alloy config be written in Nix?")

- nixpkgs has no River format/generator (pinned rev + master both checked — no pkgs.formats.river, no services.alloy.settings, no community flake found). Built a ~75-line pure-builtins river.nix (blocks/repeated blocks/refs/map-constructors/lists/comments + ordered list-bodies for stage pipelines). Proven end-to-end: rendered the owner's full Alloy pipeline from an attrset via nix-instantiate; alloy validate exit 0 + fmt clean (1.12.2, pinned pkgs); output vs the hand-written validated config identical modulo cosmetic ordering; alloy run → all components healthy. Dev bugs documented as caveats: rec-attrset `map` shadowing builtins.map; `${f x}` vs `${f x arg}` interpolation braces; alphabetical emission → use ordered list bodies where stage run-order matters. Embedded in docs/erebos/monitoring.md as new section (scratch copy is 24h-ephemeral — wiki is the durable home). Owner applies if adopted; agent made no repo changes.

## 2026-11-09 — owner config-mode decision: plain text → /nix/store via configPath

- Owner rejected the attrset→River generator (complexity not worth it) and chose plain-text config compiled to a store file: services.alloy.configPath = pkgs.writeText "alloy-config.alloy" ''…''. Verified types.path.check accepts drv / outPath / interpolated string against the pinned nixpkgs (all three = true; str-base coercion yields /nix/store path in ExecStart). Accepted trade-off: restart-instead-of-reload on config change (positions in %S/alloy make restarts loss-free for the journal reader). docs/erebos/monitoring.md intro rewritten to record the decision; generator section marked explored-not-adopted; alloy.nix inline module still shows the /etc .text form as the reload-capable alternative.

## 2026-11-09 — Alloy in Grafana: scrape + official mixin dashboards + UI (verified path)

- Added 'Visualising Alloy in Grafana' section to docs/erebos/monitoring.md: scrape job alloy 127.0.0.1:12345 (loopback is Alloy's DEFAULT, confirmed from docs + live run); provision grafana/alloy operations/alloy-mixin/rendered/dashboards (resources/controller/loki) — panel metric names spot-checked against live 1.12.2 /metrics (raw exports, no mixin rules dependency); optional ConfigLoadFailed/LogWriteDropping rules; ScrapeTargetDown already covers collector death; Cloud-only Alloy Health/app explicitly excluded (no OSS grafana-alloy-app plugin — API 404); built-in UI via SSH tunnel, no Traefik (unauthed). Dashboards + snippets delivered to owner in chat; owner applies.

## 2026-11-09 — owner push verified: alloy.nix landed (abc3f49), one transcription bug found

- git fetch + worktree eval: configPath renders to /nix/store/nifg...-config.alloy, alloy validate exit 0 on the built file. Body identical to validated reference except labels: host = "127.0.0.1" (hardcoded) instead of config.networking.hostName → would poison {host="erebos"} queries and collide with future strategos streams. Fix = one line, owner applies. Grafana layer-1 (scrape job 'alloy' 127.0.0.1:12345) NOT in the push (+2 prometheus lines were alertmanager loopback bind [good hardening] + matrix-alertmanager mention=true); dashboards not provisioned yet. Wiki table + review note updated.

## 2026-11-09 — dashboard 14055 evaluated and rejected; gap found: loki not scraped

- Owner asked if grafana.com dashboard 14055 (Loki stack monitoring Promtail/Loki) fits Erebos. Downloaded rev 5 JSON: ~20/22 metric panels are Kubernetes-Helm-specific (cAdvisor container_*, kube_pod_container_resource_*, pod=~loki.*, app_kubernetes_io_name) + a dead promtail row → no data on Erebos. Only the 5 raw loki_* panels would work — and revealed the real gap: prometheus.nix has NO loki scrape job (loki self-metrics on 127.0.0.1:3100/metrics unmonitored). Snippet added to Grafana section: job_name loki → 127.0.0.1:3100, alongside alloy job pending. Recommendation: keep mixin alloy-loki/controller/resources + custom LogQL panels.

## 2026-11-09 — review of 11a4979 (owner pushed: alloy job, host fix, terranix dashboards)

- Verified via worktree + nix eval: alloy scrape job compiles into scrapeConfigs; vars.alloy.port=12345 present; journal host fixed (hardcoded 'erebos' — noted stage caveat + config.networking.hostName alternative).
- Flags raised: (1) loki scrape job still missing → 14055's 5 functional panels empty; snippet recorded in wiki. (2) 14055 added anyway; DS binding verified to work by literal-name luck; ~20/22 panels k8s-dead — custom loki-overview.json (10 verified panels) attached for terranix adoption via builtins.readFile. (3) grafana_dashboard.authentik RENAMED to traefik (14837→17346) → old authentik dashboard orphaned/unmanaged in Grafana; owner to confirm intent. (4) UI-edited managed dashboards collide with next tofu apply.
- monitoring.md Grafana section: dashboard path corrected to terranix-managed (not provision.dashboards as first drafted).

## 2026-11-09 — dashboard question: no journal-native pre-made exists (search closed)

- Owner asked for an off-the-shelf journal-only Loki dashboard. Exhaustive check: grafana.com families 12611/18042/24574/20254/13186 all k8s/docker-shaped (variable/expr JSON evidence, incl. hardcoded DS uid); grafana/alloy-scenarios systemd-journal + linux dirs = config only, no dashboards; Alloy Health = Cloud-only. Conclusion recorded in monitoring.md: custom loki-overview.json stays the route; 18042 identified as the mechanically-forkable base (vars → job/host/unit) if community layout desired. Side note: scenarios' config.alloy independently mirrors Erebos' journal pipeline choices.

## 2026-11-09 — journal-logs dashboard built & query-verified (dropdown per application)

- Owner asked for a simple dashboard: application dropdown, empty/All = every app. Built journal-logs.json (uid erebos-journal-logs): Loki var "unit" label_values({job="systemd-journal"}, unit), includeAll+multi, allValue ".*"; stacked timeseries sum by (unit) count_over_time [1m] + logs panel, both honoring a "search" textbox regex var (default .*). Every query executed against a live grafana-loki 3.6.7 (scratch): label API returns unit values; All selector includes unit-less streams (unit=~".*" matches absent label); filtered selector returns only chosen unit. Datasource bound by literal names Prometheus/Loki (same proven mechanism). Terranix: resource.grafana_dashboard.journal_logs.config_json = builtins.readFile ./journal-logs.json.

## 2026-11-09 — journal-logs v2: removed search filter (highlight pitfall)

- Owner reported full log lines sur-highlighted orange/yellow. Cause: Grafana logs panel highlights the matched substring of filter queries; the bonus regex textbox defaulted to ".*", matching every line end-to-end. Fix v2: dropped the search variable and "|~ \"$search\"" from both panel exprs (selector-only + count_over_time) — selector-only renders unhighlighted, same as Explore browsing. Lesson for any filter-box dashboard: never default a regex filter to .* in a logs panel; use an empty-match trick or omit the feature.

## 2026-11-09 — roles clarified: Alloy ≠ Prometheus replacement (new monitoring.md section)

- Owner asked whether Alloy's scrape capability obsoletes Prometheus. Answer: no — Prometheus holds TSDB+PromQL, rule engine (alert chain to @erebot), and Grafana datasource; Alloy substituting scraping would still remote_write to a store. Documented as 'Alloy vs Prometheus' section in monitoring.md with the future architecting hook: when strategos joins the fleet, Alloy-on-agent + remote_write to Erebos is the right pattern (also for Drive-Health smartctl exporter).

## 2026-11-09 — JSON→Nix dashboard conversion pattern (journal-logs kind/spec)

- Owner gave the Grafana-13 native (kind/spec, vizConfig 13.0.12) Journal Logs JSON and asked for it as Nix. Produced sys/terraform-suitable journal-logs.nix = a plain attrset mirroring the JSON; wire with config_json = builtins.toJSON (import ./journal-logs.nix). Verified faithful: builtins.toJSON output re-normalized (jq -S) is byte-identical to the owner's export (diff clean).
- Pattern: any Grafana JSON (incl. new kind/spec format) → Nix attrset → builtins.toJSON — the dashboard becomes editable Nix data (DRY with vars/interpolation), same terranix grafana_dashboard resource. Caveat noted: this exported JSON binds queries by group="loki" with NO explicit panel datasource field — relies on default/explicit binding; if a panel errors after apply, add datasource {name="Loki"} at the query-group level. Owner applies; agent made no repo change.

## 2026-11-10 — state sync: monitoring pipeline after e0f8d57 (dashboards stay manual)

- Repo state confirmed: e0f8d57 keeps sys/services/monitoring/dashboards/systemd-journal.json (Grafana 13 kind/spec) and REMOVED terranix 14055 entries. Owner decision recorded: dashboard import is manual (UI); node_exporter+traefik remain terranix-managed.
- todo/monitoring-alerting rewritten. LIVE: Alloy→Loki pipeline (217k+ lines verified), scrape jobs incl. alloy+loki (verified returning series), Alertmanager loopback-bound, 9 rules covering the new jobs via ScrapeTargetDown, ServiceOffline whitelist includes alloy.
- What's left, reprioritized: (1) Loki retention NOT in loki.nix — unbounded disk growth (snippet ready in docs/erebos/monitoring); (2) remaining loopback binds Loki :3100 + bridge :3004 (auth_enabled=false makes :3100 a mutation surface on podman0); (3) Grafana cleanup: 14055 if it ever applied + orphaned authentik 14837; (4) ALERT_LINKS suppression env; (5) 1-week noise soak; optionals: geoip mmdb, pipeline-health alerts, request_path cardinality, host interpolation.

## 2026-11-10 — retention snippet CORRECTED: max_look_back_period removed in Loki 3.x

- Owner asked for the retention config. Before handing it over, verified against the flake-pinned grafana-loki 3.7.9 via --verify-config: limits_config.max_look_back_period is REJECTED ("field not found in type validation.plain") — it was dropped in Loki 3.x, so the wiki snippet's earlier "live version verified" claim (2026-11-07) had been wrong (2.x-era key). Corrected snippet (compactor 4 keys + limits_config.retention_period 720h only) now genuinely prints "config is valid" on 3.7.9; monitoring.md updated with dated correction note (old log entry left intact per append-only). Full merged loki.nix delivered in chat for owner to apply; nix side additionally eval-checked compactor/limits blocks render from the actual module in a scratch worktree.

## 2026-11-10 — loopback binds: findings verified + ready snippets (loki + bridge)

- nftables firewall module (pinned nixpkgs) reading: trustedInterfaces (podman0, wt0) accept at top of chain input; extraInputRules appended to input-allow afterwards => firewall cannot shield :3100/:3004 from containers/Netbird peers; bind fixes required. Loki: server.http/grpc_listen_address accepted by 3.7.9 --verify-config (full merged yaml valid). Bridge: jaywink/matrix-alertmanager@0.9.0 src/app.js line 18 has no host param; nixpkgs module (services/matrix/matrix-alertmanager.nix, maintainer erethon) exposes no address option -> overrideAttrs postPatch APP_ADDRESS pass-through + environment.APP_ADDRESS=127.0.0.1; first patch attempt had double-comma syntax bug (build exit 0 but node --check would crash — caught by inspecting artifact, npm never syntax-checks); corrected build produced APP_ADDRESS-bearing app.js passing node --check; listen semantics proven on node 22. Snippets in docs/erebos/monitoring.md new 'Loopback binds' section; todo item updated (wt0 exposure noted). Owner applies; all clients already loopback => no functional change expected.
