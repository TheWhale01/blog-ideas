---
title: Monitoring
type: reference
created: 2026-10-15
updated: 2026-11-09
tags: [nixos, erebos, monitoring, observability]
sources: []
---

# Erebos — Monitoring Stack

The monitoring stack lives in `sys/services/monitoring/` and is composed by `sys/services/monitoring/default.nix`.

## Alloy vs Prometheus — who does what (owner Q 2026-11-09)

Alloy *can* scrape metrics (`prometheus.scrape`), which tempts "drop Prometheus" — but Prometheus
on Erebos holds three roles, one substitutable at best: **(1) scraper** — Alloy could take it over,
but Alloy has no TSDB/PromQL: scraped data must remote_write to *something*, so Prometheus (or a
new Mimir/VM) stays; on a single node this adds a hop, removes nothing. **(2) rule engine** —
every alert (ServiceCrashed, DiskSpaceLow, ScrapeTargetDown, …) is Prometheus rules → Alertmanager
→ matrix-alertmanager → @erebot; Alloy evaluates no rules. **(3) Grafana's queryable datasource** —
all PromQL panels hit the Prometheus API. Current canonical split: Alloy = log pipeline, Prometheus
= metrics TSDB + rules. The Alloy-as-agent pattern genuinely pays off when the fleet grows: run
Alloy on strategos scraping journal + node metrics and remote_write/loki.push TO Erebos'
Prometheus/Loki — Erebos stays the central query/alert/visualize tier, strategos runs no TSDB.
That's the shape for the Drive-Health (smartctl on bare metal) and multi-host log plans.

## Components

| File | Tool | Role |
|---|---|---|
| `prometheus.nix` | Prometheus | Time-series metrics and alerting rules |
| `grafana.nix` | Grafana | Dashboards, visualization, alerting UI |
| `loki.nix` | Loki 3.7.9 | Log aggregation + storage (filesystem chunks; **collector-independent**) |
| `alloy.nix` (landed `abc3f49`; store-path mode `configPath = pkgs.writeText`) | **Alloy** (promtail removed from nixpkgs — EOL, module assertion) | Log shipping. Repo file is still `promtail.nix`, `enable`d with **no config** → Loki zero series until the patch is applied. Final collector config: next section (validated with `nix eval` + `alloy validate`/`fmt`). |

## Log collection — Alloy Nix module (validated 2026-11-09; owner applies)

The `services.alloy` module (checked through nixpkgs master) has **no `settings` attrset** —
River is a DSL with no generator, unlike the attrset-driven `matrix-synapse`/`mas` configs.
**Owner decision 2026-11-09: plain-text config written to a store file via `configPath`** —
`pkgs.writeText` + `services.alloy.configPath = alloyConf` (type-coercion verified: `types.path`
accepts the derivation directly, `.outPath`, or interpolation). Consequence, accepted knowingly:
config changes **restart** Alloy instead of hot-reloading (the module's `reloadTriggers` only
follow `environment.etc."alloy/*.alloy"` files, and every edit changes the store hash embedded in
ExecStart). Journal collection is not harmed by restarts: reader positions live in the state dir
(`%S/alloy`), so the service resumes where it stopped. The `/etc` variant below stays documented
as the reload-capable alternative; pick one, don't mix (component names must be unique across
whatever `/etc/alloy/*.alloy` files still exist). The module also grants the service the
`systemd-journal` supplementary group automatically, and **fails the whole initial load** if a
referenced file (e.g. the geoip mmdb) is missing — hence the crash-loop warnings below.

The validated module below landed as `sys/services/monitoring/alloy.nix` in `abc3f49`.
Application remains **owner-driven** — the agent delivers snippets, not changes.
**Review note 2026-11-09**: the pushed copy has `host = "127.0.0.1"` in the journal labels
(transcription slip — should be `host = "${config.networking.hostName}"` → `erebos`); everything
else is byte-equivalent to the validated config (`alloy validate` OK on the built store file). It renders Alloy's River config from Nix the way `mas.nix`
renders YAML — values interpolated from module options rather than hardcoded:

- `loki.write "default"` push URL ← `config.services.loki.configuration` (currently
  `http://127.0.0.1:3100/loki/api/v1/push`)
- journal `host` label ← `config.networking.hostName`
- pipeline: `loki.source.journal "system"` (`path = "/var/log/journal"`, `max_age = "12h"`) →
  `discovery.relabel "journal"` promotes `__journal__systemd_unit` → `unit` →
  `loki.process "system"` (`stage.match` on `{unit="traefik.service"}`: `stage.json` extracts
  `status`/`request_path`/`client_ip`/`method`, `stage.labels` adds status/request_path/method) →
  `loki.write`. Geoip `stage.geoip` is included **commented out**: it opens its mmdb at config
  load and `/var/lib/geoip/GeoLite2-City.mmdb` does not exist today — uncommenting without the
  file bricks the service (crash loop, verified experimentally with `alloy run`).

```nix
{ config, ... }:

let
  lokiCfg = config.services.loki.configuration;
in
{
  services.alloy = {
    enable = true;
    # configPath stays at its default (/etc/alloy, a directory). That is the
    # reload-capable mode: the module adds every environment.etc."alloy/*.alloy"
    # file to systemd.services.alloy.reloadTriggers, so nixos-rebuild switch
    # hot-reloads Alloy (SIGHUP) instead of restarting it. Pointing configPath
    # at a store path (e.g. "${pkgs.writeText ...}") works but loses reload —
    # the module docs call this out explicitly.
  };

  # Alloy speaks River (its own DSL). Even current nixpkgs master has NO
  # structured `settings` option for services.alloy (unlike matrix-synapse /
  # the mas.yaml we generate), so environment.etc is the supported way to keep
  # the config declarative — same spirit as mas.nix, text-rendered instead of
  # attrset-rendered.
  environment.etc."alloy/config.alloy".text = ''
    loki.write "default" {
      endpoint {
        url = "http://${lokiCfg.common.instance_addr}:${toString lokiCfg.server.http_listen_port}/loki/api/v1/push"
      }
    }

    // Promote __journal__systemd_unit (internal label, dropped by default) to a real `unit` label.
    discovery.relabel "journal" {
      targets = []

      rule {
        source_labels = ["__journal__systemd_unit"]
        target_label  = "unit"
      }
    }

    loki.source.journal "system" {
      path          = "/var/log/journal"
      max_age       = "12h"
      labels = {
        job  = "systemd-journal",
        host = "${config.networking.hostName}",
      }
      relabel_rules = discovery.relabel.journal.rules
      forward_to    = [loki.process.system.receiver]
    }

    loki.process "system" {
      forward_to = [loki.write.default.receiver]

      // Traefik JSON access logs -> structured labels (accessLog.format=json is set in traefik.nix)
      stage.match {
        selector = "{unit=\"traefik.service\"}"

        stage.json {
          expressions = {
            status       = "DownstreamStatus",
            request_path = "request_path",
            client_ip    = "ClientHost",
            method       = "RequestMethod",
          }
        }

        stage.labels {
          values = {
            status       = "",
            request_path = "",
            method       = "",
          }
        }

        // Optional geoip — requires /var/lib/geoip/GeoLite2-City.mmdb, which does NOT exist today.
        // stage.geoip opens its DB at config load, so a missing file fails the whole initial load
        // (crash loop, not just an unhealthy component). Provision it first (MaxMind license key
        // needed), make it readable by the dynamic alloy user, then uncomment:
        // stage.geoip {
        //   source  = "client_ip"
        //   db      = "/var/lib/geoip/GeoLite2-City.mmdb"
        //   db_type = "city"
        // }
      }
    }
  '';
}
```

Validation recipe (what this module passed): `nix eval` of the rendered text from the actual
flake config, then `$A validate file.alloy` + `$A fmt` (exit 0 + silence), where
`A=/nix/store/*-grafana-alloy-*/bin/alloy`; component names must be unique across all
`/etc/alloy/*.alloy` files.

Notes:
- `format` is **not** a valid argument on `loki.source.journal` in Alloy 1.16 (a leftover from the
  promtail draft — `alloy validate` rejects it); journal lines flow as text and `stage.json` parses
  Traefik's messages fine.
- Re-validate any edit before applying: `$A validate file.alloy` where
  `A=/nix/store/*-grafana-alloy-*/bin/alloy` (exit 0 + silence = OK).
- Once this lands, the retention snippet above finally has something to prune; first load backfills
  only the last `max_age` (12 h) of journal — expect a burst on first start, quiet after.
- **Params pitfall (2026-11-10):** this Loki build rejects `now`/`-10m` for `start`/`end`
  (`strconv.ParseInt` error) — use RFC3339 or ns-integers:
  `--data-urlencode "start=$(date -u -d '10 min ago' +%FT%TZ)" --data-urlencode "end=$(date -u +%FT%TZ)"`.
  Geoip is LIVE since 2026-11-10 16:29 (`stage.geoip` + `stage.structured_metadata` in the running
  config, geo_country="FR" filter proven against real traffic; metadata is filterable, not echoed
  in query_range arrays).
- Queries to smoke-test: `{job="systemd-journal"} |= "Failed password"` (sshd), 
  `{unit="traefik.service", status=~"5.."}` (edge errors), `{host="erebos"} |~ "error"`.

## SSHD GeoIP pipeline + 17877 dashboard findings (2026-11-10)

**A — traefik country label (owner pushed, LIVE).** `geoip_country_name = ""` added to the
traefik `stage.labels`. Live label index now carries `geoip_country_name` with bounded values
(France/Ireland/Norway/Portugal/Singapore/United States). `Access by Country` panel (country-mode
GeoMap on `sum by (geoip_country_name)(count_over_time(...))`) shows data.

**B — sshd branch (LIVE since 2026-11-10 evening; owner pushed + rebuilt).** Validated first with
`alloy validate` rc=0 against the production 1.16.0 binary on the full rendered config. First
post-deploy entry — the owner's own login from IPv6 `2a02:842a:…` — carries
`geoip_country_name="France"` in Loki; city decoded as Triel-sur-Seine. Sibling `stage.match`
inside `loki.process "system"` in `sys/services/monitoring/alloy.nix`, inserted after the traefik
match's closing `}`:

    stage.match {
      selector = "{unit=\"sshd.service\"}"

      stage.regex {
        expression = "(?:from|by) (?P<client_ip>[0-9a-fA-F.:]+)"
      }

      stage.geoip {
        source  = "client_ip"
        db      = "/var/lib/geoip/GeoLite2-City.mmdb"
        db_type = "city"
      }

      stage.structured_metadata {
        values = {
          geo_country = "geoip_country_code",
          geo_city    = "geoip_city_name",
        }
      }

      stage.labels {
        values = {
          geoip_country_name = "",
        }
      }
    }

Regex covers `from <IP>` (Accepted/Failed password) and `by <IP>` (Connection closed by <IP>).
After `nixos-rebuild switch` + alloy restart, verify with a FRESH ssh login — pre-existing entries
keep their old label sets (labels only apply to newly ingested lines):
`{unit="sshd.service", geoip_country_name=~".+"}`.

**Semantics findings (source read: grafana/loki `clients/pkg/logentry/stages/geoip.go`):**
- `stage.geoip` with missing/unparsable source IP is a **no-op — the entry passes through**
  (process() returns early; Run always forwards). IP-less sshd lines (pam_unix session, bare kex
  errors) are NOT dropped by the sshd branch. The old crash-loop warning concerned only a missing
  mmdb at config load.
- On the running Alloy 1.16.0 the geo fields populate the **extracted map only**: live
  label-values check shows `geoip_location_latitude/_longitude` and `geoip_city_name` are NOT in
  the label index; only explicitly promoted fields become labels. No per-coordinate stream
  explosion on Erebos.

**Dashboard 17877 "GeoIP" anatomy.** It was built for `{job="iptables", geoip_country_name=~".+"}` —
kernel firewall packet-LOG streams with the country promoted as a LABEL; Erebos emits no such
stream (NixOS firewall logs nothing). Only "Access by Country" (an aggregation query) can be made
to work: selector sed to `unit="traefik.service"` + country label (step A, done). The
"IP Locations"/"Heatmap"/"(SSH)" panels are raw-log **point-layer** GeoMaps needing lat/lon per
entry, which in Loki means promoting `geoip_location_latitude/longitude` as labels = one stream
per visitor coordinate, unbounded cardinality — **do not promote**. Replace them with country-mode
GeoMap/table panels on
`sum by (geoip_country_name) (count_over_time({unit="traefik.service", geoip_country_name=~".+"}[$__range]))`
and, after B lands, a twin query on `{unit="sshd.service", geoip_country_name=~".+"}`. A genuine
attacker world-map needs firewall LOG rules feeding journald (see intrusion-detection idea) —
different source, same pipeline shape.

**Landed state (commit 1966884 `added GeoIP on traefik and sshd`).** The sshd branch merged exactly
as delivered. Owner followed the advice above: the vendored fork `sys/terraform/dashboards/geoip.json`
(title GeoIP, uid `pV2rGDZ4z`) now contains ONLY the two aggregation country-GeoMap panels —
`sum by (geoip_country_name) (count_over_time({unit="traefik.service", geoip_country_name=~".+"}[$__range]))`
and its `{unit="sshd.service", ...}` twin ("Access by Country" / "Access by Country (SSH)").
Live verification 1h after deploy: sshd `France=2, United States=2` (the Vultr prober pool arrived —
first hostile entry on the SSH map), traefik 11 countries with France≈6.5k (desktop sync).

## Dashboards-as-code: terranix Grafana provisioning (owner-landed 2026-11-10)

`sys/terraform/grafana.nix` is a **terranix** module (Nix → Terraform) — Terraform *files* are never
committed; the terraform CLI is just the executor of the generated config. Content:

- `provider.grafana` → `https://grafana.<domain>`, auth from agenix identity file
  `/run/agenix/grafana-terraform` (a Grafana API key secret, `[REDACTED]` here).
- Catalog dashboards stay fetched LIVE at apply time: `data.http` → grafana.com API downloads of
  1860 (node exporter) and 17346 (traefik) → `grafana_dashboard` resources. `data.http.geoip` still
  fetches 17877 but is now **unused** (the resource reads the local fork instead) — dead wiring,
  remove or re-point deliberately.
- Datasource-binding trick (kills the `${DS_*}` "Skip import" no-data bug at the root):
  traefik.json uses terraform's own `replace(..., "$${DS_PROMETHEUS}", vars.prometheus.uid)`;
  the geoip fork is substituted **at Nix-eval time** with
  `builtins.replaceStrings [ "\${DS_LOKI}" ] [ (toString vars.loki.uid) ]` over
  `builtins.readFile ./dashboards/geoip.json`. Both render concrete UIDs into `config_json`.
- `systemd_journal` resource is COMMENTED OUT: the committed `dashboards/systemd-journal.json` is in
  the new Grafana unified-dashboard **kind/spec (v2)** format (UI export) and the terraform provider's
  `grafana_dashboard` path POSTs the legacy API → "issue with v2 API" commit note.
  **Resolution (2026-11-10, delivered, owner applies):** convert the dashboard back to the CLASSIC
  schema, mirroring the working geoip fork exactly: top-level `__inputs` (name `DS_LOKI`), every panel
  target + templating var + annotation datasource bound to `{"type":"loki","uid":"${DS_LOKI}"}`,
  `id/gnetId` null keys, `schemaVersion: 39`, fixed `uid: erebos-journal-logs`. Then the resource line
  must use the same substitution as geoip:
  `systemd_journal.config_json = "${builtins.replaceStrings [ "\${DS_LOKI}" ] [ "${toString vars.loki.uid}" ] (builtins.readFile ./dashboards/systemd-journal.json)}";`.
  Queries verified live before delivery (41 series on the timeseries expr, 55 units via label_values).
  After apply, DELETE the older manually-imported copy in the UI (random uid) — Terraform manages
  only the `erebos-journal-logs` uid.
- Same module also defines the Authentik OAuth2 app/groups for Grafana (pre-existing).

## Visualising Alloy in Grafana (setup guide, verified 2026-11-09)

Three layers, no Cloud required:

1. **Scrape Alloy's self-metrics.** Alloy's HTTP server (metrics, built-in UI, live debugging)
   defaults to **127.0.0.1:12345** (verified from `alloy run` docs + a live scrape; the nixpkgs
   module passes no listen-addr override, so loopback is already the posture). Add to
   `prometheus.nix` `scrapeConfigs`, matching the existing job style:
   ```nix
   { job_name = "alloy";
     static_configs = [{ targets = [ "127.0.0.1:12345" ]; }]; }
   ```
   The existing `ScrapeTargetDown` (`up == 0`) rule then covers collector death automatically.
2. **Dashboards (owner choice: managed via terranix/grafana-provider, not `provision.dashboards`).**
   `sys/terraform/grafana.nix` `resource.grafana_dashboard.*` — dashboards pulled with `data.http`
   from grafana.com or `builtins.readFile` of a repo JSON.
Status after `11a4979`: `alloy` scrape job
   + `vars.alloy.port` ✅ landed (verified `nix eval`: jobs = prometheus, node_exporter, authentik,
   traefik, alloy); journal `host` label fixed to `erebos` (hardcoded — on erebos-stage the same
   config would tag stage logs `host=erebos`; `config.networking.hostName` avoids that); dashboard
   **14055 added despite k8s-specific verdict** — DS binding works by luck (literal names
   "Prometheus"/"Loki" == provisioned DS names), but ~20/22 panels stay No-data; the 5 real
   loki_* panels need the **still-missing loki scrape job**:
   ```nix
   { job_name = "loki";
     static_configs = [{ targets = [ "127.0.0.1:${toString config.services.loki.configuration.server.http_listen_port}" ]; }]; }
   ```
   Also in `11a4979`: the `authentik` dashboard resource was **renamed** to `traefik` (14837 →
   17346) — the previously managed authentik dashboard is now orphaned in Grafana; to keep both,
   add a second `grafana_dashboard.authentik` entry instead of having renamed. Recommended
   replacement for 14055: minimal custom `loki-overview.json` (10 panels: distributor
   lines/bytes, ingester streams + append failures, alloy journal-read + write-dropped, two LogQL
   log panels; metric names verified against live Alloy /metrics) attached in chat 2026-11-09,
   wire with `config_json = builtins.readFile ./loki-overview.json;`. Note: UI edits to
   provider-managed dashboards collide on next `tofu apply`.
   Original mixin route (`grafana/alloy` → `operations/alloy-mixin/rendered/dashboards/`
   ships ready JSON. For this pipeline: `alloy-resources` (CPU/mem/restarts), `alloy-controller`
   (component health + evaluation time), `alloy-loki` (journal lines read, write latency, dropped
   entries). All panel exprs use raw exported metric names (spot-checked against a live
   `alloy 1.12.2 /metrics`: `alloy_component_controller_running_components`,
   `loki_source_journal_target_lines_total`, `loki_write_request_duration_seconds`,
   `alloy_build_info`, …) — **no mixin recording rules needed**. Datasource variables are
   `type=datasource` so they bind to the provisioned Prometheus/Loki at render; `cluster`/`namespace`
   stay empty on a single node (harmless). Commit the JSONs to the repo and add to `grafana.nix`:
   ```nix
   services.grafana.provision.dashboards.settings.providers = [{
     name = "alloy";
     options.path = ./grafana-dashboards;   # dir containing the three *.json
   }];
   ```
3. **Optional pipeline alerts** (append to the `monitoring_self` rules group):
   ```yaml
   - alert: AlloyConfigLoadFailed
     expr: alloy_config_last_load_successful == 0
   - alert: AlloyLogWriteDropping
     expr: increase(loki_write_dropped_entries_total[5m]) > 0
   ```

**Journal-native dashboard search (2026-11-09, exhaustive):** grafana.com has **no** pure
systemd-journal Loki dashboard — every logging-dashboard family is container/k8s-shaped, JSON-verified:
12611 (hardcoded DS uid 000000042 + `container/pod/stream` vars), 18042 v2 (same vars),
24574 v3 (`container_name/service_name/instance`), 20254 + 13186 (`label_values(kube_pod_info,
namespace)` / workload vars). Official grafana/alloy-scenarios `systemd-journal` + `linux` ship
config only — **no dashboards** (their config.alloy matches Erebos' pipeline, useful as upstream
cross-check). Alloy Health dashboards = Grafana Cloud only. Hence the custom `loki-overview.json`
(terranix `builtins.readFile`). If a community layout is ever wanted, closest is 18042 with its
four variables swapped to `job/host/unit` selectors — a mechanical JSON fork.

**Also rejected: dashboard 14055 "Loki stack monitoring (Promtail, Loki)"** (owner asked 2026-11-09):
JSON inspection shows ~20 of 22 metric panels target the **Loki-Stack Helm chart on Kubernetes**
(`container_memory_working_set_bytes`, `kube_pod_container_resource_*`, `pod=~"loki.*"`,
`app_kubernetes_io_name`) plus a dead **promtail** row — they'd all render "No data" on Erebos.
Usable instead: a `loki` scrape job (currently absent! Loki's self-metrics on `127.0.0.1:3100/metrics`
are invisible to Prometheus):
```nix
{ job_name = "loki";
  static_configs = [{ targets = [ "127.0.0.1:3100" ]; }]; }
```
which feeds the 5 genuine `loki_distributor_lines_received_total` / `loki_ingester_memory_streams` /
`loki_distributor_bytes_received_total` / append-failures / dropped-samples style panels, alongside
hand-built LogQL log panels for the actual log views.

Not applicable: the "Alloy Health" integration and the Alloy app tile are **Grafana Cloud**
features; there is no `grafana-alloy-app` plugin in the OSS catalog (checked grafana.com API —
NotFound). The built-in Alloy UI (component graph, health, profiling) is best accessed ad hoc:
`ssh -L 15000:127.0.0.1:12345 hades@erebos` → http://localhost:15000 — don't expose it through
Traefik; it has no auth.

## Attrset-driven River generation (custom `pkgs.formats`-style generator)

> **Status: explored, NOT adopted** (owner chose plain text + store path 2026-11-09).
> Kept as reference — the caveats and proof record remain valid for any future `pkgs.formats.river`.


Since nixpkgs has no River generator (verified against the pinned rev **and** master as of
2026-11-09), the Alloy body above can instead be written as pure Nix data and compiled to
River at eval time — the same authoring model as `mas.nix` (attrset → yaml-format) and synapse's
`settings`. Two files in `sys/services/monitoring/`:

**`river.nix`** (~75 lines, no dependencies beyond `builtins`):

```nix
# river.nix — an attrset → River (Grafana Alloy DSL) generator, pkgs.formats-style.
#
# Conventions:
#   - top level: { "loki.write"."default" = BODY; } → loki.write "default" { ... }
#   - BODY attrset: keys emitted in alphabetical order (River is declarative, so
#     component argument order doesn't matter — EXCEPT stage.* blocks inside
#     loki.process, which RUN IN FILE ORDER. For ordered bodies pass a LIST of
#     single-key attrsets instead — emitted verbatim in list order).
#   - plain nested attrset  → unlabeled block (  endpoint { ... }  )
#   - non-empty list of plain attrsets → repeated block ( rule { } rule { } )
#   - river.list / river.map / river.ref → explicit attribute forms:
#       list  → key = [ v, v ]      (for lists of strings/refs)
#       map   → key = { k = v, .. } (River map constructors: labels, expressions, values)
#       ref   → bare expression passthrough (component references like
#               discovery.relabel.journal.rules — NOT a quoted string!)
#   - key "comment" = "text" anywhere → // text line
rec {
  ref = v: { __river = "ref"; inherit v; };
  map = v: { __river = "map"; inherit v; };
  list = v: { __river = "list"; inherit v; };
  # body: ordered LIST of single-key attrsets as a block's body (for stage.*
  # pipelines where file order matters, nested inside a component block)
  body = v: { __river = "body"; inherit v; };

  isRv = v: builtins.isAttrs v && builtins.hasAttr "__river" v;
  plain = v: builtins.isAttrs v && !(isRv v);

  indent = n: builtins.concatStringsSep "" (builtins.genList (_: " ") n);

  q = s: ''"${builtins.replaceStrings [ "\\" "\"" ] [ "\\\\" "\\\"" ] s}"'';

  val = v:
    if isRv v then
      if v.__river == "ref" then v.v
      else if v.__river == "map" then mapCtor v.v
      else if v.__river == "list" then listLit v.v
      else throw "river: unknown marker"
    else if builtins.isString v then q v
    else if builtins.isInt v || builtins.isBool v || builtins.isFloat v then toString v
    else if builtins.isList v then listLit v
    else throw "river: unsupported value of type ${builtins.typeOf v}";

  listLit = vs: "[ " + builtins.concatStringsSep ", " (builtins.map val vs) + " ]";

  mapCtor = m:
    "{ " + builtins.concatStringsSep ", "
      (builtins.map (k: "${k} = ${val m.${k}}") (builtins.attrNames m))
    + " }";

  emitAttr = ind: k: v:
    if k == "comment" then "${indent ind}// ${v}\n"
    else if isRv v && v.__river == "body" then "${indent ind}${k} {\n${emitOrderedList (ind + 2) v.v}${indent ind}}\n"
    else if plain v then "${indent ind}${k} {\n${emitBodyLines (ind + 2) v}${indent ind}}\n"
    else if builtins.isList v && v != [ ] && builtins.all plain v then
      builtins.concatStringsSep ""
        (builtins.map (it: "${indent ind}${k} {\n${emitBodyLines (ind + 2) it}${indent ind}}\n") v)
    else "${indent ind}${k} = ${val v}\n";

  emitBodyLines = ind: m:
    builtins.concatStringsSep ""
      (builtins.map (k: emitAttr ind k m.${k}) (builtins.attrNames m));

  # ordered body: list of single-key attrsets, emitted in list order
  emitOrderedList = ind: items:
    builtins.concatStringsSep ""
      (builtins.map (it: builtins.concatStringsSep ""
        (builtins.map (k: emitAttr ind k it.${k}) (builtins.attrNames it))) items);

  generate = cfg:
    builtins.concatStringsSep "" (builtins.map (t:
      builtins.concatStringsSep "" (builtins.map (l:
        let body = cfg.${t}.${l}; in
        ''${t} "${l}" {''
        + "\n"
        + (if builtins.isList body then emitOrderedList 2 body else emitBodyLines 2 body)
        + "}\n\n"
      ) (builtins.attrNames cfg.${t}))
    ) (builtins.attrNames cfg));
}
```

**`alloy.nix`** using it:

```nix
{ config, ... }:
let
  river = import ./river.nix;
  r = river.ref;
  mp = river.map;
  lokiCfg = config.services.loki.configuration;

  alloyRiver = {
    "loki.write"."default".endpoint.url =
      "http://${lokiCfg.common.instance_addr}:${toString lokiCfg.server.http_listen_port}/loki/api/v1/push";

    "discovery.relabel"."journal" = {
      targets = [ ];
      rule = {
        source_labels = [ "__journal__systemd_unit" ];
        target_label = "unit";
      };
    };

    "loki.source.journal"."system" = {
      path = "/var/log/journal";
      max_age = "12h";
      labels = mp { job = "systemd-journal"; host = config.networking.hostName; };
      relabel_rules = r "discovery.relabel.journal.rules";
      forward_to = [ (r "loki.process.system.receiver") ];
    };

    # ordered list-body: forward_to -> comment -> stage.match, emitted in list order
    "loki.process"."system" = [
      { forward_to = [ (r "loki.write.default.receiver") ]; }
      { comment = "Traefik JSON access logs -> structured labels (accessLog.format=json in traefik.nix)"; }
      { "stage.match" = {
          selector = "{unit=\"traefik.service\"}";
          "stage.json".expressions = mp {
            status = "DownstreamStatus"; request_path = "request_path";
            client_ip = "ClientHost"; method = "RequestMethod";
          };
          "stage.labels".values = mp { status = ""; request_path = ""; method = ""; };
          comment = "geoip: stage.geoip opens its mmdb at config LOAD — file absent on Erebos, crash loop if enabled";
        };
      }
    ];
  };
in
{
  services.alloy.enable = true;   # configPath stays default /etc/alloy → hot-reload
  environment.etc."alloy/config.alloy".text = river.generate alloyRiver;
}
```

Semantics: plain nested attrsets = unlabeled blocks; non-empty list of attrsets = repeated blocks;
`river.ref` = bare expression passthrough (component references — never quoted); `river.map` =
`key = { k = v, … }` constructor; `river.list` forces attribute-list form over block repetition;
`comment = "…"` emits `// …`. Top-level: `"component.type"."label" = BODY`.

Caveats (honest limits):
- Attrset bodies emit keys **alphabetically**. River is order-independent for component args, but
  `stage.*` blocks inside one body RUN IN FILE ORDER — a future `stage.geoip` would sort before
  `stage.json`. Use list-bodies (or `river.body` for nested blocks) wherever stage order matters.
- `ref` passes raw text; typos surface only at `alloy validate`/load, not at Nix eval.
- Keep the validation habit regardless: `nix eval` the rendered text + `alloy validate` + `alloy fmt`.

Test record 2026-11-09: rendered this exact attrset with `nix-instantiate --eval --raw` against
the generator; `alloy validate` exit 0 (Alloy 1.12.2 from the flake's pinned nixpkgs); output
differs from the hand-written validated config only cosmetically (component/attr order, inline
maps); `alloy run` on the generated file → "All Alloy components are healthy."
Bugs caught while building it: exported `map` marker shadows `builtins.map` inside a `rec` set
(use `builtins.map` internally), and interpolated function applications need the argument **inside**
the `${ … }` braces.

## Loki retention — 1 month (paste into `loki.nix`, inside `services.loki.configuration`)

Loki 3.x needs **both** blocks — the period alone does nothing without an enabled compactor, and
the compactor refuses to start in 3.x without `delete_request_store`:

```nix
      compactor = {
        retention_enabled = true;
        delete_request_store = "filesystem";   # matches common.storage.filesystem
        working_directory = "${config.services.loki.dataDir}/compactor";
        compaction_interval = "10m";
      };
      limits_config = {
        retention_period = "720h";              # 1 month
      };
```

> **Corrected 2026-11-10**: an earlier revision of this snippet included
> `limits_config.max_look_back_period` — that field was **removed in Loki 3.x**
> (`loki --verify-config` on 3.7.9 rejects the key outright: "field max_look_back_period
> not found in type validation.plain"). The snippet above is re-verified against the live
> binary version: `msg="config is valid"`.

Verify after apply: `curl -s 127.0.0.1:3100/metrics | grep -m1 loki_build_info` (service alive) and
`curl -s 127.0.0.1:3100/metrics | grep loki_compactor` (compactor working). Existing chunks are
deleted lazily at the next compaction cycle once they age past 720 h — retention starts eating the
backlog only once logs actually flow, which needs the collector fix above first.

## Loopback binds (2026-11-10, findings + snippets)

`networking.firewall.trustedInterfaces = [ "podman0" "wt0" ]` **accepts ALL ports from those
interfaces at the top of nft's `input` chain** (module verified: the trusted accept precedes any
per-port logic; `extraInputRules` lands later in `input-allow` and cannot fix this). So anything
on the podman network AND every Netbird peer could reach Loki `:3100` (auth off ⇒ delete API)
and the alert bridge `:3004`. The fix must be at the bind level:

**Loki** — config supports it; `--verify-config`-validated against the flake's 3.7.9:
```nix
      server = {
        http_listen_port = 3100;
        http_listen_address = "127.0.0.1";
        grpc_listen_address = "127.0.0.1";
      };
```
Safe: all clients (Alloy push, Grafana DS, Prometheus scrape, alertmanager→bridge) already
use 127.0.0.1. If the fleet ever ships logs from strategos, revisit (dedicated bind or tunnel).

**matrix-alertmanager** — upstream Node app had no host knob (`src/app.js`:
`app.listen(process.env.APP_PORT, () => {})`; nixpkgs module exposes only `port`), so bind
via a small package patch + env var (built + `node --check` verified 2026-11-10; behavior
identical upstream when `APP_ADDRESS` is unset):
```nix
  services.matrix-alertmanager.package = pkgs.matrix-alertmanager.overrideAttrs (final: prev: {
    postPatch = (prev.postPatch or "") + ''
      substituteInPlace src/app.js --replace-fail "app.listen(process.env.APP_PORT" "app.listen(process.env.APP_PORT, process.env.APP_ADDRESS"
    '';
  });
  systemd.services.matrix-alertmanager.environment.APP_ADDRESS = "127.0.0.1";
```
(`pkgs` arrives automatically in modules; upstream PR + module `listenAddress` option is the
clean long-term path. `grafana.nix` note: adding a serviceConfig.Environment would also work —
the `environment.APP_ADDRESS` form merges per-key into the module's env attrset.)

Post-rebuild check: `ss -tln | grep -E ':(3100|3004|9095)\b'` ⇒ all 127.0.0.1; ops-room alerts
still flowing (they ride loopback anyway).

## Integration

The monitoring stack is wired into the NixOS config through:

- `sys/services/monitoring/default.nix` (imports all four modules);
- `sys/services/default.nix` (the monitoring module is imported here);
- Traefik dynamic config (Grafana UI exposed at `grafana.thewhale.fr`, behind Authentik).

## TODO

For current status and tasks, see [[todo/monitoring-alerting]].

## External Monitoring

Because Erebos is a server, it also relies on:

- **Netbird** (via nixos-modules) — VPN for safe remote management;
- **Traefik's** own `metrics.prometheus` endpoint (exposed on `127.0.0.1:8200`).

These are all part of the overall monitoring and observability picture.
