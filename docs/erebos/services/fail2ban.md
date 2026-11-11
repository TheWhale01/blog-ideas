---
title: Fail2ban
type: reference
created: 2026-11-10
updated: 2026-11-10
tags: [nixos, erebos, services, security]
sources: [erebos:sys/services/fail2ban.nix (9b24091), "https://github.com/fail2ban/fail2ban", nixpkgs-b40629e-services.fail2ban-module, fail2ban-1.1.0-config/filter.d/sshd.conf]
---

# Fail2ban on Erebos — Service Documentation

**LIVE since 2026-11-10** (commit `9b24091 added fail2ban on SSH`): v1.1.0 from nixpkgs, journald
(systemd) backend, sshd jail only. Verified: full-config eval of merged jails + live journal shows
`Starting Fail2ban v1.1.0` / `Server ready` / fresh sqlite DB; visible in Loki as
`{unit="fail2ban.service"}` with zero bans so far — correct for the default mode (see semantics).

## Landed configuration (`sys/services/fail2ban.nix`)

```nix
{ ... }:

{
  services.fail2ban = {
    enable = true;
    ignoreIP = [
      "10.88.0.0/16"          # podman
      "10.2.0.0/16"           # gluetun
      "192.168.1.0/24"        # LAN
    ];
    bantime-increment = {
      enable = true;
      maxtime = "48h";
      rndtime = "1h";
    };
    jails.sshd.settings = {
      maxretry = 5;
      findtime = "10m";
      bantime  = "1h";
    };
  };
}
```

Merged result verified via `nix eval` on the full config: `DEFAULT` = `backend=systemd`,
`banaction=iptables-multiport` (auto: nixpkgs module picks nftables action only if
`networking.nftables.enable`; `packageFirewall` follows `networking.firewall.package` — so bans
land in the same kernel-nft tables family as the NixOS firewall), `ignoreip` includes `127.0.0.1/8 ::1`
+ the three ranges above; `sshd` jail: `port="22"` (auto-derived from `services.openssh.ports`),
5 failures/10 min → 1h ban, escalating ×2 per repeat up to 48h with 1h jitter.

**Side effect (intentional, from the nixpkgs module):** `services.openssh.settings.LogLevel` becomes
`"VERBOSE"` via `mkDefault` — more sshd lines in journald (and thus Loki). Not set in `openssh.nix`.

## sshd filter semantics (fail2ban 1.1.0, `mode = normal` default)

Read from the pinned filter (`config/filter.d/sshd.conf` in the nixpkgs source tarball):

**Counted as failures (normal mode):** username-touching events — `Failed password` (impossible
here: `PasswordAuthentication=false`), `Invalid user X from <IP>`, `User X ... not allowed because
not listed in AllowUsers` (fires for everyone but `hades`), `maximum authentication attempts
exceeded`, `Disconnecting: Too many authentication failures` (catches sustained key-stuffing),
PAM `authentication failure`.

**NOT counted (normal mode):** anonymous `Connection closed/reset ... [preauth]` — these match the
`mdre-normal-other` regex but are wrapped `<F-NOFAIL>` → match-but-not-failure. This is exactly the
current Vultr prober pool traffic pattern (no username sent) → zero bans expected from them today.
Also never counted: `Failed publickey for hades` (default `publickey=nofail` — protects key logins
from self-lockout) and the `WARNING: connection from ... post-quantum` lines (no pattern matches).

**Consequence:** the jail as configured guards username enumeration and sustained stuffing only;
anonymous preauth noise passes. To ban those too: `mode = ddos|aggressive` counts `[preauth]`
closes/resets — but then interrupted legitimate handshakes count as well, so
**fill `ignoreIP` with the home IPv6 prefix (and Netbird `wt0` subnet) BEFORE switching** — see
[[todo/fail2ban-tuning]].

## Observing it

```bash
fail2ban-client status sshd            # banned IPs, fail counts
iptables -nL f2b-sshd; ip6tables -nL f2b-sshd
journalctl -u fail2ban --since today
```
Loki: `{unit="fail2ban.service"}`. The ban-geo pipeline **landed with this same commit** (`9b24091`):
a `stage.match` on `{unit="fail2ban.service"}` in alloy.nix extracts `Ban <IP>` → geoip →
`geoip_country_name` label (`Unban` lines don't match the case-sensitive `Ban ` pattern, so un-bans
never double-count). The geoip dashboard carries a third geomap panel, **"Banned by Country (SSH)"**:
`sum by (geoip_country_name) (count_over_time({unit="fail2ban.service", geoip_country_name=~".+"}[$__range]))`.
It shows No data until the FIRST ban happens — expected, not a failure (see semantics above).
Deterministic end-to-end smoke test (safe, reversible):
`sudo fail2ban-client set sshd banip 8.8.8.8` → within ~30s the panel gains a United States tile
(`{unit="fail2ban.service", geoip_country_name="United States"}` in Loki) →
`sudo fail2ban-client set sshd unbanip 8.8.8.8`.
Semantics: the panel counts **ban events**, not distinct IPs — a returning offender re-banned
monthly counts repeatedly; Loki can't distinct-count an unlabeled field, and promoting `client_ip`
to a label to enable that is the cardinality anti-pattern we avoided. Alert seed for
[[ideas/intrusion-detection]].

## Related
- [[docs/erebos/monitoring]] — journal→Loki pipeline (fail2ban logs ride it)
- [[todo/fail2ban-tuning]] — open decisions (aggressive mode + ignoreIP, web jails)
- [[ideas/intrusion-detection]] — detection layers this pairs with
