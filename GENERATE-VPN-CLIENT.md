# Generate OpenVPN Client Certificates

Quick guide to create VPN access for team members.

## Prerequisites

- VPN server running: vpn-server-global (35.200.142.203)
- SSH access to VPN server
- Team member's name (use format: firstname.lastname)
- Script location: `~/generate-vpn-client.sh` (already on VPN server)

## Steps (Automated)

### 1. SSH to VPN Server

```bash
gcloud compute ssh vpn-server-global \
  --zone=asia-south1-c \
  --project=kluisz-global-cp
```

### 2. Generate Client Certificate and Config

```bash
# Run the automated script
./generate-vpn-client.sh <username>

# Example:
./generate-vpn-client.sh john.doe
```

**What it does:**
- Generates client certificate
- Signs the certificate
- Creates .ovpn config file with embedded certificates
- Shows download command

**Output:**
```
✅ Client configuration generated successfully!
📁 Config file location:
   ~/client-configs/john.doe/john.doe.ovpn
```

### 3. Download Config File

Exit SSH session, then run on your local machine:

```bash
gcloud compute scp \
  vpn-server-global:~/client-configs/<name>/<name>.ovpn \
  ~/Downloads/<name>-kluisz-global.ovpn \
  --zone=asia-south1-c \
  --project=kluisz-global-cp
```

### 4. Send to Team Member

Send them:
1. The `.ovpn` file
2. Connection instructions (see below)

## Connection Instructions for Team Members

**macOS/Linux:**

```bash
# Install OpenVPN (one-time)
brew install openvpn  # macOS
# OR
sudo apt-get install openvpn  # Ubuntu

# Connect to VPN
sudo openvpn ~/Downloads/<name>-kluisz-global.ovpn
```

**Windows:**

1. Download and install [OpenVPN GUI](https://openvpn.net/community-downloads/)
2. Copy `.ovpn` file to `C:\Program Files\OpenVPN\config\`
3. Right-click OpenVPN GUI → Run as Administrator
4. Right-click system tray icon → Connect

## Verify Connection

After connecting:

```bash
# Check VPN IP (should be 10.8.0.x)
ifconfig tun0  # macOS/Linux
ipconfig       # Windows

# Test access to private VM
ssh user@10.160.0.x  # Replace with actual private IP
```

## Quick Example

Generate certificate for user "sarah.smith":

```bash
# 1. SSH to VPN server
gcloud compute ssh vpn-server-global --zone=asia-south1-c --project=kluisz-global-cp

# 2. Generate certificate (on VPN server)
./generate-vpn-client.sh sarah.smith

# 3. Exit SSH and download (on your local machine)
gcloud compute scp \
  vpn-server-global:~/client-configs/sarah.smith/sarah.smith.ovpn \
  ~/Downloads/sarah.smith-kluisz-global.ovpn \
  --zone=asia-south1-c \
  --project=kluisz-global-cp

# 4. Connect
sudo openvpn ~/Downloads/sarah.smith-kluisz-global.ovpn
```

## Revoke Access

If a team member leaves or certificate is compromised:

```bash
# On VPN server
cd ~/openvpn-ca
./easyrsa revoke <name>
./easyrsa gen-crl

# Update server
sudo cp pki/crl.pem /etc/openvpn/server/
echo "crl-verify crl.pem" | sudo tee -a /etc/openvpn/server/server.conf
sudo systemctl restart openvpn-server@server
```

## Troubleshooting

**Connection fails:**
```bash
# Check OpenVPN server status
gcloud compute ssh vpn-server-global --zone=asia-south1-c
sudo systemctl status openvpn-server@server
sudo tail -f /var/log/openvpn/openvpn.log
```

**Can't access private VMs:**
```bash
# Verify routes while connected to VPN
ip route  # Should show: 10.160.0.0/20 via 10.8.0.1 dev tun0
```

## VPN Server Info

- **Server:** vpn-server-global
- **Public IP:** 35.200.142.203
- **VPN Network:** 10.8.0.0/24
- **VPC Subnets:** 10.160.0.0/20 (and others)
- **Protocol:** UDP 443

## Scripts Comparison

There are two scripts you might see:

1. **`~/generate-vpn-client.sh`** (on VPN server) ✅ **Use this one**
   - Automated one-command certificate generation
   - Properly embeds certificates in .ovpn file
   - Located on VPN server, ready to use

2. **`/tmp/generate-client-config.sh`** (local machine) ❌ Don't use
   - Original setup script with certificate embedding issues
   - Kept for reference only

**Always use `~/generate-vpn-client.sh` on the VPN server.**

---

**Questions?** Check /tmp/VPN-SETUP-GUIDE.md for full documentation.
