---
title: Fail2ban on Erebos
created: 2026-10-22
updated: 2026-11-10
type: idea
tags: [nixos, erebos, security, infrastructure]
sources: [https://github.com/fail2ban/fail2ban]
---

# Fail2ban on Erebos

**Status**: validated snippet delivered to owner 2026-11-10 (evaluated against pinned nixpkgs
`b40629e`; pending `nixos-rebuild switch` + live ban observation)

[Fail2ban](https://github.com/fail2ban/fail2ban) is a daemon that scans logs for repeated authentication failures and bans the offending source IPs by injecting firewall rules (reject/drop for a configurable time). Mature project (since 2011), IPv6-capable since v0.10.

## Why on Erebos

- **Public attack surface**: Erebos exposes TCP 80/443 to the whole internet and SSH on port 22. Even with `PasswordAuthentication = false` (see [[docs/erebos/services]]), sshd still gets a constant stream of brute-force probing — fail2ban suppresses the noise, saves log volume, and stops attacker enumeration of valid usernames.
- **Web services**: Nextcloud, Authentik, Vaultwarden, and the Matrix homeserver all log failed auth events. Repeat offenders can be banned at the edge instead of relying only on app-level rate limits.
- **Defense in depth**: complements Cloudflare (see [[ideas/cloudflare-tunnel-ssh]]) and Netbird — it protects anything reached *directly* on the host IP, where Cloudflare's rules don't apply.

## Realistic expectations

Fail2ban reduces the *rate* of incorrect authentication attempts; it does not replace strong authentication. Its value is mostly: log noise reduction, throttling credential-stuffing, and deterring casual scanners. Application-level lockouts (Authentik has its own) remain the primary control.

## NixOS integration

`services.fail2ban` exists in nixpkgs. Two Erebos-specific adaptations:

1. **journald backend**: NixOS has no `/var/log/auth.log`; sshd (and most services) log to journald, so jails must use `backend = systemd` with `journalmatch`.
2. **Don't ban yourself**: exclude internal ranges on `ignoreIP` — podman (`10.88.0.0/16`), gluetun tunnels (`10.2.0.0/16` etc.), Netbird (`wt0` subnet), LAN, and your own admin IPs. Tailscale/Netbird exit ranges matter especially for the sshd jail.

### Final module (validated 2026-11-10 against pinned nixpkgs `b40629e`)

`sys/services/fail2ban.nix` + add `./fail2ban.nix` to `sys/services/default.nix` imports:

```nix
{ ... }:

{
  services.fail2ban = {
    enable = true;

    # never ban ourselves; 127.0.0.1/8 and ::1 are added by the module automatically.
    # ADJUST: LAN CIDR + Netbird (wt0) subnet + home IPv6 prefix are placeholders below.
    ignoreIP = [
      "10.88.0.0/16"          # podman
      "10.2.0.0/16"           # gluetun
      "192.168.1.0/24"        # LAN placeholder
      "2a02:842a:3540::/64"   # home IPv6 prefix placeholder (dynamic — check ip -6 addr on pontos)
    ];

    # repeat offenders escalate: 1h -> 2h -> 4h ... capped at 48h, jittered 1h
    bantime-increment = {
      enable = true;
      maxtime = "48h";
      rndtime = "1h";
    };

    jails.sshd.settings = {
      maxretry = 5;   # generous — key-only sshd, offenders are preauth fuzzers
      findtime = "10m";
      bantime  = "1h";
    };
  };
}
```

Rendered-jail verification (merged `services.fail2ban.jails` from `eval-config`):
`DEFAULT` = `backend=systemd`, `banaction=iptables-multiport`, `ignoreip` merged correctly;
`sshd` = `port="22"` (auto from `services.openssh.ports`), overrides applied.

Notes:
- **Firewall-backend task RESOLVED by the module itself**: `banaction` defaults to
  `iptables-multiport` (auto `nftables-multiport` if `networking.nftables.enable`), and
  `packageFirewall` follows `networking.firewall.package` — bans land in the same nft kernel
  backend as the NixOS firewall. Erebos uses the default → no action needed.
- Enabling fail2ban silently raises `services.openssh.settings.LogLevel` to `VERBOSE`
  (module `mkDefault`, not set in `openssh.nix`) — more sshd lines in journald, which now also
  means more lines into Loki via the journal source. Expected, not a bug.
- journald backend means NO `/var/log/auth.log` dependency; the sshd jail reads the journal
  directly (`backend=systemd` is the module's DEFAULT). IPv6 attackers are bannable
  (ip6tables side of the action).
- Behind Cloudflare, Traefik sees *Cloudflare's* IPs in the remote address — the web jails must parse the real client IP (X-Forwarded-For handling in the filter), otherwise bans would hit Cloudflare ranges. Consider this jail family optional/harder; sshd is the clear win. (Today nextcloud.thewhale.fr et al. are direct — Loki access logs already show real client IPs.)
- `fail2ban` writes its own events to journald (`logtarget=SYSLOG`) → they flow into Loki as
  `{unit="fail2ban.service"}` for free. Optional synergy: add a `stage.match` like the sshd
  geoip branch (`stage.regex { expression = "Ban (?P<client_ip>[0-9a-fA-F.:]+)" }` → geoip →
  country label) and get a "countries we've banned" map + alert seed for
  [[ideas/intrusion-detection]].
- Watch for a week with `fail2ban-client status sshd` / `{unit="fail2ban.service"} |= "Ban"`
  before adding aggressive web jails.

## Tasks

- [x] Decide firewall backend alignment — resolved: nixpkgs module auto-aligns (see Notes)
- [ ] Implement sshd jail with journald backend + ignoreIP list (podman, gluetun, wt0, LAN) — snippet delivered, owner to apply + fill real LAN/Netbird/home-IPv6 values
- [ ] Verify first real ban (Vultr preauth prober pool already hits port 22 every few minutes — check `iptables -nL f2b-sshd` + `fail2ban-client status sshd` a day after deploy)
- [ ] Prototype one web-facing jail (nextcloud or authentik) with real-client-IP parsing
- [ ] Re-evaluate the sshd jail once [[ideas/cloudflare-tunnel-ssh]] removes public port 22 (may keep as belt-and-suspenders or scope to Netbird-only access)

## Related

- [[docs/erebos/services]] — current SSH/firewall posture
- [[ideas/cloudflare-tunnel-ssh]] — complementary network-hiding strategy
- [[todo/index]] — security hardening backlog
