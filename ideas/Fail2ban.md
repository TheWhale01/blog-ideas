---
title: Fail2ban on Erebos
created: 2026-10-22
updated: 2026-10-22
type: idea
tags: [nixos, erebos, security, infrastructure]
sources: [https://github.com/fail2ban/fail2ban]
---

# Fail2ban on Erebos

**Status**: unimplemented

[Fail2ban](https://github.com/fail2ban/fail2ban) is a daemon that scans logs for repeated authentication failures and bans the offending source IPs by injecting firewall rules (reject/drop for a configurable time). Mature project (since 2011), IPv6-capable since v0.10.

## Why on Erebos

- **Public attack surface**: Erebos exposes TCP 80/443 to the whole internet and SSH on port 22. Even with `PasswordAuthentication = false` (see [[docs/erebos/services]]), sshd still gets a constant stream of brute-force probing — fail2ban suppresses the noise, saves log volume, and stops attacker enumeration of valid usernames.
- **Web services**: Nextcloud, Authentik, Vaultwarden, and the Matrix homeserver all log failed auth events. Repeat offenders can be banned at the edge instead of relying only on app-level rate limits.
- **Defense in depth**: complements Cloudflare (see [[ideas/Cloudflare-Tunnel-SSH]]) and Netbird — it protects anything reached *directly* on the host IP, where Cloudflare's rules don't apply.

## Realistic expectations

Fail2ban reduces the *rate* of incorrect authentication attempts; it does not replace strong authentication. Its value is mostly: log noise reduction, throttling credential-stuffing, and deterring casual scanners. Application-level lockouts (Authentik has its own) remain the primary control.

## NixOS integration

`services.fail2ban` exists in nixpkgs. Two Erebos-specific adaptations:

1. **journald backend**: NixOS has no `/var/log/auth.log`; sshd (and most services) log to journald, so jails must use `backend = systemd` with `journalmatch`.
2. **Don't ban yourself**: exclude internal ranges on `ignoreIP` — podman (`10.88.0.0/16`), gluetun tunnels (`10.2.0.0/16` etc.), Netbird (`wt0` subnet), LAN, and your own admin IPs. Tailscale/Netbird exit ranges matter especially for the sshd jail.

### Example module (`sys/services/fail2ban.nix`)

```nix
{ config, ... }:

let
  vars = config.vars;
in
{
  services.fail2ban = {
    enable = true;
    jails = {
      sshd.settings = {
        enabled = true;
        backend = "systemd";
        port = "ssh";
        maxretry = 5;
        findtime = "10m";
        bantime = "1h";
        # never lock out internal/VPN networks
        ignoreIP = [
          "127.0.0.0/8"
          "10.88.0.0/16"     # podman
          "10.2.0.0/16"      # gluetun tunnels
          # + your Netbird (wt0) subnet and LAN ranges
        ];
      };

      # Traefik/nginx access logs (journald), custom filter for 401/403 floods
      # nextcloud / authentik / matrix jails can follow once log formats are
      # confirmed — journald journalmatch per unit, e.g.:
      #   journalmatch = { _SYSTEMD_UNIT = "nextcloud.service" };
    };
  };
}
```

Notes:
- Check which firewall backend Erebos runs: with **nftables** set `banaction = nftables` (or the `nftables-multiport` action); with iptables the default `iptables-multiport` works.
- Behind Cloudflare, Traefik sees *Cloudflare's* IPs in the remote address — the web jails must parse the real client IP (X-Forwarded-For handling in the filter), otherwise bans would hit Cloudflare ranges. Consider this jail family optional/harder; sshd is the clear win.
- Test with `fail2ban-client status sshd` and monitor `journalctl -u fail2ban` for a week in `bantime.increment = true` slow-ban mode before aggressive settings.

## Tasks

- [ ] Decide firewall backend alignment (iptables vs nftables action)
- [ ] Implement sshd jail with journald backend + ignoreIP list (podman, gluetun, wt0, LAN)
- [ ] Prototype one web-facing jail (nextcloud or authentik) with real-client-IP parsing
- [ ] Re-evaluate the sshd jail once [[ideas/Cloudflare-Tunnel-SSH]] removes public port 22 (may keep as belt-and-suspenders or scope to Netbird-only access)

## Related

- [[docs/erebos/services]] — current SSH/firewall posture
- [[ideas/Cloudflare-Tunnel-SSH]] — complementary network-hiding strategy
- [[todo/index]] — security hardening backlog
