---
title: Cloudflare Tunnel for SSH
created: 2026-10-15
updated: 2026-10-15
type: idea
tags: [nixos, erebos, cloudflare, ssh, tunnel, security, networking]
sources: []
---

# Cloudflare Tunnel for SSH

Expose SSH access to Erebos via Cloudflare Tunnel, eliminating the need to open port 22 publicly while maintaining secure remote access.

## Why Cloudflare Tunnel?

- **No open ports**: SSH is exposed only through Cloudflare's network, not directly on the server's public IP.
- **Zero Trust**: Leverage Cloudflare's authentication (e.g., Access policies) or keep it simple with SSH keys.
- **DDoS protection**: Cloudflare absorbs attacks before they reach your server.
- **IP flexibility**: Access your server from anywhere without worrying about dynamic IPs or ISP restrictions.

## Implementation

### 1. Add `cloudflared` to NixOS

Create a new module at `sys/services/cloudflared.nix`:

```nix
{ config, pkgs, lib, ... }:

let
  tunnelName = "erebos-ssh";
  credentialsFile = "/var/lib/cloudflared/${tunnelName}.json";
  cloudflare = pkgs.cloudflared;
  cloudflaredUser = "cloudflared";
  cloudflaredGroup = "cloudflared";
  sshPort = 22;
  tunnelDomain = "ssh.thewhale.fr";
  tunnelId = "YOUR_TUNNEL_ID";  # Replace with actual tunnel ID from Cloudflare
in {
  users.groups.${cloudflaredGroup} = {};
  users.users.${cloudflaredUser} = {
    isSystemUser = true;
    group = cloudflaredGroup;
  };

  systemd.services.cloudflared-tunnel = {
    description = "Cloudflare Tunnel for SSH";
    wantedBy = [ "multi-user.target" ];
    after = [ "network-online.target" ];
    serviceConfig = {
      ExecStart = "${cloudflare}/bin/cloudflared tunnel --config ${credentialsFile} run ${tunnelName}";
      Restart = "always";
      User = cloudflaredUser;
      Group = cloudflaredGroup;
      # Ensure the credentials file exists and is readable
      ExecStartPre = "${pkgs.coreutils}/bin/mkdir -p /var/lib/cloudflared && ${pkgs.coreutils}/bin/chown ${cloudflaredUser}:${cloudflaredGroup} /var/lib/cloudflared";
    };
  };

  # Optional: Create the credentials directory at build time
  system.activationScripts.setupCloudflaredDir = ''
    mkdir -p /var/lib/cloudflared
    chown ${cloudflaredUser}:${cloudflaredGroup} /var/lib/cloudflared
  '';
}
```

### 2. Generate Tunnel Credentials

On a machine with `cloudflared` installed (or temporarily on Erebos):

```bash
# Install cloudflared temporarily (if not already available)
nix-shell -p cloudflared

# Authenticate with Cloudflare
cloudflared tunnel login

# Create a new tunnel
cloudflared tunnel create erebos-ssh

# Copy the credentials file to Erebos
scp ~/.cloudflared/<tunnel-id>.json hades@erebos:/var/lib/cloudflared/erebos-ssh.json
chown cloudflared:cloudflared /var/lib/cloudflared/erebos-ssh.json
chmod 600 /var/lib/cloudflared/erebos-ssh.json
```

### 3. Configure the Tunnel

Create a tunnel configuration file at `/var/lib/cloudflared/config.yml`:

```yaml
# /var/lib/cloudflared/config.yml
tunnel: erebos-ssh
credentials-file: /var/lib/cloudflared/erebos-ssh.json

routing:
  rules:
    - service: ssh://localhost:22
      ingress:
        - hostname: ssh.thewhale.fr
          service: ssh://localhost:22
        - service: http_status:404  # Catch-all rule
```

### 4. Update Firewall Rules

In `sys/configuration.nix` or a firewall module:

```nix
networking.firewall = {
  allowedTCPPorts = [ 80 443 ];  # Remove 22 from public ports
  extraCommands = ''
    # Allow SSH only from Cloudflare IPs (optional, if you want to keep port 22 open as a fallback)
    ${lib.concatMapStringsSep "\n" (ipRange: "iptables -A INPUT -p tcp --dport 22 -s ${ipRange} -j ACCEPT") [
      "173.245.48.0/20"
      "103.21.244.0/22"
      "103.22.200.0/22"
      "103.31.4.0/22"
      "141.101.64.0/18"
      "108.162.192.0/18"
      "190.93.240.0/20"
      "188.114.96.0/20"
      "197.234.240.0/22"
      "198.41.128.0/17"
      "162.158.0.0/15"
      "104.16.0.0/13"
      "104.24.0.0/14"
      "172.64.0.0/13"
      "131.0.72.0/22"
    ]}
  '';
};
```

### 5. DNS Configuration

In your Cloudflare DNS settings:
- Add a CNAME record for `ssh.thewhale.fr` pointing to `<tunnel-id>.cfargotunnel.com`
  (replace `<tunnel-id>` with the ID from `cloudflared tunnel info erebos-ssh`)

Or use a DNS-only setup with the tunnel's assigned subdomain.

### 6. Test the Tunnel

```bash
# Check tunnel status
cloudflared tunnel info erebos-ssh

# Test SSH access
ssh hades@ssh.thewhale.fr -p 22
```

### 7. (Optional) Add Cloudflare Access Policies

For additional security, set up Cloudflare Access:
1. Go to Cloudflare Zero Trust dashboard.
2. Create a new Access application for `ssh.thewhale.fr`.
3. Add policies (e.g., require email authentication or SAML).
4. SSH will then require both the SSH key **and** Cloudflare authentication.

## Erebos Integration

1. Add the `cloudflared.nix` module to `sys/services/default.nix`:
   ```nix
   imports = [
     ./cloudflared.nix
     # ... other services
   ];
   ```

2. Rebuild Erebos:
   ```bash
   sudo nixos-rebuild switch --flake .#erebos
   ```

3. Start and enable the tunnel:
   ```bash
   sudo systemctl enable --now cloudflared-tunnel
   ```

## Notes

- **Credentials Security**: The tunnel credentials file (`erebos-ssh.json`) is sensitive. Restrict permissions to the `cloudflared` user only.
- **High Availability**: Consider running `cloudflared` as a systemd service with `Restart=always` to ensure the tunnel reconnects if interrupted.
- **Monitoring**: Add a health check for the tunnel (e.g., `systemd` watchdog or a simple HTTP endpoint via `cloudflared` metrics).
- **Fallback**: If you remove SSH from public ports entirely, ensure you have console access (e.g., via a KVM or serial console) in case of tunnel issues.

## Resources

- [Cloudflare Tunnel Documentation](https://developers.cloudflare.com/cloudflare-one/connections/connect-apps/)
- [Cloudflare IP Ranges](https://www.cloudflare.com/ips/)
- [NixOS cloudflared package](https://search.nixos.org/packages?channel=23.11&show=cloudflared)
