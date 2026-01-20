# pod-synology-vpn

Ansible-guided ProtonVPN setup for Synology DSM with kill switch.

## Overview

This pod provides a documented, repeatable process for setting up ProtonVPN on Synology. It combines automated steps with guided manual steps (DSM has no CLI for VPN management).

**Runs directly on Synology via SSH** - no remote inventory needed.

**Features:**
- Step-by-step guided setup with pauses for manual actions
- Kill switch via iptables (blocks traffic if VPN drops)
- Connection monitoring with DSM notifications
- Status checking playbook
- Clean uninstall

## Prerequisites

1. **ProtonVPN account** - https://protonvpn.com
2. **SSH access to Synology** - Enabled in DSM Control Panel
3. **Ansible installed on Synology** - `pip install ansible`

## Quick Start

```bash
# SSH into Synology
ssh jamie@synology

# 1. Clone this repo
cd /volume1/repos
git clone git@github.com:YOUR_USERNAME/pod-synology-vpn.git
cd pod-synology-vpn

# 2. Update configuration (optional)
# Edit vars/default.yml with your preferences

# 3. Run setup (interactive, prompts for sudo password)
ansible-playbook setup.yml -K
```

## Playbooks

| Playbook | Description |
|----------|-------------|
| `setup.yml` | Full guided setup with pauses for manual steps |
| `status.yml` | Check VPN connection and kill switch status |
| `uninstall.yml` | Remove configuration and restore normal network |

All playbooks require sudo - use `-K` flag to prompt for password:
```bash
ansible-playbook setup.yml -K
ansible-playbook status.yml -K
ansible-playbook uninstall.yml -K
```

## Files Created on Synology

The `setup.yml` playbook creates these files on the Synology:

```
/volume1/vpn-config/
├── protonvpn.ovpn          # OpenVPN config (you provide this)
├── killswitch.sh           # Enable kill switch
├── killswitch-disable.sh   # Disable kill switch
├── vpn-monitor.sh          # Connection monitor (runs via cron)
├── vpn-monitor.log         # Monitor log file
└── .vpn-state              # State tracking for notifications
```

Plus a cron job running every 5 minutes to check VPN status.

## DSM Notifications

The monitor script sends notifications via `synodsmnotify`:
- **VPN Alert** - When connection drops
- **VPN Restored** - When connection recovers

Notifications only fire on state *changes* (not every 5 minutes).

## Username Suffix (NetShield)

DSM UI doesn't accept ProtonVPN username suffixes. The playbook automatically adds them after you create the VPN profile.

Configure in `vars/default.yml`:
```yaml
vpn_username_suffix: "+f2"
```

| Suffix | Feature |
|--------|---------|
| `+f1` | Anti-malware (NetShield) |
| `+f2` | Anti-malware + ad-blocking |
| `+nr` | Moderate NAT |
| `+b:N` | Enforce exit server (N from .ovpn) |

Combine as needed: `+f2+nr`

## Kill Switch

The kill switch blocks ALL outbound internet traffic except:
- Traffic through VPN tunnel (tun0)
- Local network traffic (configurable CIDR)
- VPN connection establishment

**Enable:**
```bash
sudo /volume1/vpn-config/killswitch.sh
```

**Disable:**
```bash
sudo /volume1/vpn-config/killswitch-disable.sh
```

## Manual Steps Required

Due to DSM limitations, these steps require the web UI:

1. **Create VPN Profile** - Control Panel → Network → Network Interface
2. **Connect/Disconnect** - Same location
3. **Delete VPN Profile** - Same location (for uninstall)

The playbook pauses and provides detailed instructions for each manual step.

## Troubleshooting

**VPN won't connect:**
- Verify OpenVPN credentials (different from Proton login!)
- Check .ovpn file was copied correctly
- Try TCP instead of UDP if network is restrictive

**Lost internet after enabling kill switch:**
- VPN probably disconnected
- SSH to Synology and run: `sudo /volume1/vpn-config/killswitch-disable.sh`
- Reconnect VPN in DSM, then re-enable kill switch

**Can't SSH to Synology:**
- Kill switch may be blocking you if on different network
- Access DSM via local network or iDRAC/IPMI if available

## Future Integration

This pod is standalone but designed to integrate with the control plane/data plane architecture. A future VPN gateway service could:
- Provide VPN for multiple services
- Offer API-based connection management
- Integrate with HashiCorp Vault for credentials
