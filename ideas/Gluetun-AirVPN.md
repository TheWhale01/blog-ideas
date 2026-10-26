---
title: Gluetun and AirVPN for slskd
created: 2026-10-15
updated: 2026-10-26
type: idea
tags: [nixos, erebos, gluetun, vpn, slskd, airvpn, protonvpn, networking]
sources: []
---

# Gluetun and AirVPN for slskd

**Status: ✅ Implemented (2026-10-26) via Option 1** — two dedicated Gluetun containers, staying on ProtonVPN. AirVPN was not needed. Details: [[docs/erebos/containers]] (VPN tunnels section). Key lessons from the implementation:

- Each tunnel has its **own Proton WireGuard device config** (one WG key = one tunnel) and its own agenix secret (`gluetun/transmission.age`, `gluetun/slskd.age`).
- slskd was converted to a container (`sys/containers/slskd.nix`) sharing the `gluetun-slskd` netns; its **native gluetun integration** (`SLSKD_VPN=true`, `SLSKD_VPN_GLUETUN_URL=http://127.0.0.1:8000`, `SLSKD_VPN_PORT_FORWARDING=true`) auto-syncs the listening port — no port-sync sidecar needed.
- The initial `PCP gateway i/o timeout` on the slskd tunnel was fixed by `PORT_FORWARD_ONLY=on` + choosing a PF-reliable country (Netherlands) instead of a random one.
- slskd's web UI is now behind **Authentik forwardAuth** (it previously relied on `web.authentication disabled`).

The rest of this page is kept as the original decision context.

Since **slskd** requires a VPN connection (currently using ProtonVPN via Gluetun), and you may need to add additional services that also require VPN connectivity, this document outlines options for managing VPN containers and providers.

## Current Setup

- **slskd** is configured to use a Gluetun container with ProtonVPN (WireGuard).
- Gluetun handles VPN connectivity, port forwarding, and network routing for slskd.

## Options for Additional VPN Services

### Option 1: Second Gluetun Container

If you need to add another service (e.g., Cloudflare Tunnel) that also requires a VPN connection, you can create a second Gluetun container.

- **Pros**: Isolates slskd traffic from other services.
- **Cons**: Higher resource usage, complexity in managing two VPN containers.

#### Implementation
1. Create a new Gluetun container (e.g., `gluetun-cloudflared`) with the same ProtonVPN configuration.
2. Configure the new service (e.g., `cloudflared`) to use the new container's network namespace.

Example NixOS container configuration:
```nix
# In sys/containers/gluetun-cloudflared.nix
{
  image = "qmcgaw/gluetun:latest";
  extraContainerConfig = {
    CapAdd = [ "NET_ADMIN" "NET_RAW" ];
    Devices = [ "/dev/net/tun" ];
    Environment = {
      VPN_SERVICE_PROVIDER = "protonvpn";
      VPN_TYPE = "wireguard";
      SERVER_COUNTRIES = "Switzerland";
      VPN_PORT_FORWARDING = "on";
    };
  };
}
```

### Option 2: Use AirVPN (Recommended)

[AirVPN](https://airvpn.org/) is a privacy-focused VPN provider that may offer more flexibility for your use case, especially if you need to bypass ProtonVPN's constraints (e.g., port forwarding limitations).

- **Pros**:
  - Strong privacy focus and no logging policy.
  - Supports OpenVPN and WireGuard.
  - Allows multiple devices/connections, which could simplify your setup.
  - May avoid ProtonVPN's constraints (e.g., port forwarding limitations).

- **Cons**:
  - Requires switching VPN providers for slskd.
  - Configuration changes needed for Gluetun.

#### AirVPN Setup with Gluetun

1. Update your Gluetun configuration to use AirVPN:
   ```nix
   # In sys/containers/gluetun.nix
   {
     image = "qmcgaw/gluetun:latest";
     extraContainerConfig = {
       CapAdd = [ "NET_ADMIN" "NET_RAW" ];
       Devices = [ "/dev/net/tun" ];
       Environment = {
         VPN_SERVICE_PROVIDER = "airvpn";
         VPN_TYPE = "openvpn";
         # AirVPN-specific settings
         OPENVPN_USER = "your_airvpn_username";
         OPENVPN_PASSWORD = "your_airvpn_password";
         SERVER_COUNTRIES = "France";  # Or your preferred country
       };
     };
   }
   ```

2. Ensure your Gluetun container has the necessary capabilities and environment variables for AirVPN.

#### Why AirVPN?
- **Port Forwarding**: AirVPN supports port forwarding, which is essential for slskd.
- **Flexibility**: Easier to configure for multiple services (e.g., slskd + other VPN-dependent services).
- **Privacy**: Strong no-logging policy, which aligns with self-hosting principles.

For more details, see [AirVPN's official site](https://airvpn.org/).
