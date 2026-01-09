# OpenVPN Setup Guide for New GCP Project

Complete guide to set up OpenVPN server in any GCP project from scratch. Follow these steps to enable secure VPN access to your private VMs.

## Overview

This setup creates:
- OpenVPN server on a dedicated VM
- VPN client network (10.8.0.0/24)
- Firewall rules for VPN access
- Routes for VPN clients
- Automated client certificate generation

**Time required:** 20-30 minutes

---

## Prerequisites

Before starting, ensure you have:

- [ ] GCP project with billing enabled
- [ ] `gcloud` CLI installed and configured
- [ ] Project owner or editor permissions
- [ ] SSH access to GCP VMs

---

## Step 1: Set Project Variables

```bash
# Set these variables for your project
PROJECT_ID="your-project-id"
REGION="asia-south1"              # Change to your preferred region
ZONE="asia-south1-c"              # Change to your preferred zone
VPN_VM_NAME="vpn-server"
VPC_NETWORK="default"             # Change if using custom VPC
MACHINE_TYPE="n2d-standard-2"     # Adjust based on needs
```

Set active project:

```bash
gcloud config set project ${PROJECT_ID}
```

---

## Step 2: Create VPN Server VM

```bash
# Find latest Ubuntu 24.04 image
IMAGE=$(gcloud compute images list \
  --project=ubuntu-os-cloud \
  --filter="family=ubuntu-2404-lts-amd64" \
  --format="value(name)" \
  --limit=1)

echo "Using image: ${IMAGE}"

# Create VM with IP forwarding enabled
gcloud compute instances create ${VPN_VM_NAME} \
  --project=${PROJECT_ID} \
  --zone=${ZONE} \
  --machine-type=${MACHINE_TYPE} \
  --network-interface=network-tier=PREMIUM,stack-type=IPV4_ONLY,subnet=${VPC_NETWORK} \
  --can-ip-forward \
  --no-service-account \
  --no-scopes \
  --maintenance-policy=MIGRATE \
  --provisioning-model=STANDARD \
  --tags=vpn-server,vpn-gw \
  --create-disk=auto-delete=yes,boot=yes,device-name=${VPN_VM_NAME},image=projects/ubuntu-os-cloud/global/images/${IMAGE},mode=rw,size=20,type=pd-balanced \
  --no-shielded-secure-boot \
  --shielded-vtpm \
  --shielded-integrity-monitoring \
  --labels=purpose=vpn-server \
  --reservation-affinity=any

echo "✅ VM created successfully"
```

---

## Step 3: Reserve and Assign Static IP

```bash
# Reserve static external IP
gcloud compute addresses create ${VPN_VM_NAME}-ip \
  --project=${PROJECT_ID} \
  --region=${REGION}

# Get the reserved IP
VPN_EXTERNAL_IP=$(gcloud compute addresses describe ${VPN_VM_NAME}-ip \
  --region=${REGION} \
  --format="value(address)")

echo "✅ Reserved IP: ${VPN_EXTERNAL_IP}"

# Remove default ephemeral IP
gcloud compute instances delete-access-config ${VPN_VM_NAME} \
  --project=${PROJECT_ID} \
  --zone=${ZONE} \
  --access-config-name="External NAT"

# Assign static IP to VM
gcloud compute instances add-access-config ${VPN_VM_NAME} \
  --project=${PROJECT_ID} \
  --zone=${ZONE} \
  --access-config-name="External NAT" \
  --address=${VPN_EXTERNAL_IP}

echo "✅ Static IP assigned to VM"
```

---

## Step 4: Create Firewall Rules

```bash
# Allow OpenVPN traffic (UDP 443)
gcloud compute firewall-rules create allow-openvpn-udp \
  --project=${PROJECT_ID} \
  --network=${VPC_NETWORK} \
  --direction=INGRESS \
  --priority=1000 \
  --source-ranges=0.0.0.0/0 \
  --action=ALLOW \
  --rules=udp:443 \
  --target-tags=vpn-server \
  --description="Allow OpenVPN UDP 443 from anywhere"

# Allow VPN gateway egress
gcloud compute firewall-rules create allow-vpn-gw-egress \
  --project=${PROJECT_ID} \
  --network=${VPC_NETWORK} \
  --direction=EGRESS \
  --priority=1000 \
  --destination-ranges=0.0.0.0/0 \
  --action=ALLOW \
  --rules=all \
  --target-tags=vpn-gw \
  --description="Allow VPN gateway to reach all destinations"

echo "✅ Firewall rules created"
```

---

## Step 5: Create Route for VPN Clients

```bash
gcloud compute routes create vpn-clients-route \
  --project=${PROJECT_ID} \
  --network=${VPC_NETWORK} \
  --destination-range=10.8.0.0/24 \
  --next-hop-instance=${VPN_VM_NAME} \
  --next-hop-instance-zone=${ZONE} \
  --priority=900 \
  --tags=vpn-gw

echo "✅ Route created for VPN clients"
```

---

## Step 6: Get VPC Subnet CIDR

You need to know your VPC subnet CIDR to configure routing:

```bash
# List all subnets in your VPC
gcloud compute networks subnets list \
  --network=${VPC_NETWORK} \
  --project=${PROJECT_ID}

# Note down the IP ranges (e.g., 10.160.0.0/20)
# You'll need this in Step 8
```

**Save the subnet CIDR ranges - you'll need them next!**

---

## Step 7: Install OpenVPN Server

### 7.1: SSH to VPN Server

```bash
gcloud compute ssh ${VPN_VM_NAME} \
  --zone=${ZONE} \
  --project=${PROJECT_ID}
```

### 7.2: Run Installation Script

Copy and paste this entire script into your SSH session:

```bash
#!/bin/bash
set -e

echo "🔧 Installing OpenVPN server..."

# Update system
sudo apt-get update
sudo apt-get upgrade -y

# Install OpenVPN and Easy-RSA
sudo apt-get install -y openvpn easy-rsa iptables-persistent

# Set up Easy-RSA for certificate management
echo "🔐 Setting up PKI infrastructure..."
make-cadir ~/openvpn-ca
cd ~/openvpn-ca

# Configure Easy-RSA vars
cat > vars <<EOF
set_var EASYRSA_REQ_COUNTRY    "IN"
set_var EASYRSA_REQ_PROVINCE   "Karnataka"
set_var EASYRSA_REQ_CITY       "Bangalore"
set_var EASYRSA_REQ_ORG        "MyCompany"
set_var EASYRSA_REQ_EMAIL      "admin@example.com"
set_var EASYRSA_REQ_OU         "IT"
set_var EASYRSA_ALGO           "ec"
set_var EASYRSA_DIGEST         "sha512"
EOF

# Initialize PKI
./easyrsa init-pki
./easyrsa --batch build-ca nopass

# Generate server certificate
./easyrsa --batch gen-req server nopass
./easyrsa --batch sign-req server server

# Generate Diffie-Hellman parameters (takes 2-5 minutes)
echo "⏳ Generating DH parameters (this takes 2-5 minutes)..."
./easyrsa gen-dh

# Generate TLS auth key
openvpn --genkey secret ta.key

# Copy certificates to OpenVPN directory
sudo cp pki/ca.crt /etc/openvpn/server/
sudo cp pki/issued/server.crt /etc/openvpn/server/
sudo cp pki/private/server.key /etc/openvpn/server/
sudo cp ta.key /etc/openvpn/server/
sudo cp pki/dh.pem /etc/openvpn/server/

echo "✅ Certificates created and copied"
```

**Wait for the script to complete (takes 2-5 minutes for DH generation).**

---

## Step 8: Configure OpenVPN Server

Still in SSH session, create the server configuration:

```bash
# Get your VPC subnet CIDR from Step 6
# Replace 10.160.0.0/255.255.240.0 with your actual subnet

sudo tee /etc/openvpn/server/server.conf > /dev/null <<EOF
# OpenVPN server configuration
port 443
proto udp
dev tun

# SSL/TLS parameters
ca ca.crt
cert server.crt
key server.key
dh dh.pem

# Network configuration
server 10.8.0.0 255.255.255.0
ifconfig-pool-persist /var/log/openvpn/ipp.txt

# Push routes to clients - CHANGE THIS to your VPC CIDR
push "route 10.160.0.0 255.255.240.0"

# If you have multiple subnets, add more push routes:
# push "route 10.161.0.0 255.255.240.0"
# push "route 10.162.0.0 255.255.240.0"

# Client configuration
keepalive 10 120
tls-auth ta.key 0
cipher AES-256-GCM
auth SHA256

# User/group
user nobody
group nogroup

# Persistence
persist-key
persist-tun

# Logging
status /var/log/openvpn/openvpn-status.log
log-append /var/log/openvpn/openvpn.log
verb 3

# Performance
sndbuf 0
rcvbuf 0

# Explicit exit notification
explicit-exit-notify 1
EOF

echo "✅ OpenVPN server configured"
```

**Important:** Update the `push "route ..."` line with your actual VPC subnet CIDR!

---

## Step 9: Enable IP Forwarding and NAT

```bash
# Create log directory
sudo mkdir -p /var/log/openvpn

# Enable IP forwarding
sudo sysctl -w net.ipv4.ip_forward=1
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf

# Configure NAT with iptables
PRIMARY_IFACE=$(ip route | grep default | awk '{print $5}')
echo "Primary interface: ${PRIMARY_IFACE}"

sudo iptables -t nat -A POSTROUTING -s 10.8.0.0/24 -o ${PRIMARY_IFACE} -j MASQUERADE
sudo iptables -A FORWARD -i tun0 -o ${PRIMARY_IFACE} -j ACCEPT
sudo iptables -A FORWARD -i ${PRIMARY_IFACE} -o tun0 -j ACCEPT

# Save iptables rules
echo iptables-persistent iptables-persistent/autosave_v4 boolean true | sudo debconf-set-selections
echo iptables-persistent iptables-persistent/autosave_v6 boolean true | sudo debconf-set-selections
sudo netfilter-persistent save

echo "✅ IP forwarding and NAT configured"
```

---

## Step 10: Start OpenVPN Server

```bash
# Enable and start OpenVPN
sudo systemctl enable openvpn-server@server
sudo systemctl start openvpn-server@server

# Check status
sudo systemctl status openvpn-server@server

# Should show: active (running)
```

**Verify it shows "active (running)" in green.**

---

## Step 11: Create Client Certificate Generation Script

Still in SSH session:

```bash
cat > ~/generate-vpn-client.sh << 'EOF'
#!/bin/bash
# Generate OpenVPN client certificate and config file
# Usage: ./generate-vpn-client.sh <username>

set -e

if [ -z "$1" ]; then
    echo "Usage: $0 <username>"
    echo "Example: $0 john.doe"
    exit 1
fi

CLIENT_NAME=$1
VPN_SERVER_IP=$(curl -s ifconfig.me)

echo "🔐 Generating client certificate for: ${CLIENT_NAME}"

cd ~/openvpn-ca

# Generate client certificate
echo "📝 Creating certificate request..."
./easyrsa --batch gen-req ${CLIENT_NAME} nopass

echo "✍️  Signing certificate..."
./easyrsa --batch sign-req client ${CLIENT_NAME}

# Create client config directory
mkdir -p ~/client-configs/${CLIENT_NAME}

echo "📄 Creating OpenVPN config file..."

# Create the config file with embedded certificates
cat > ~/client-configs/${CLIENT_NAME}/${CLIENT_NAME}.ovpn << "OUTER_EOF"
##############################################
# OpenVPN Client Configuration
##############################################

client
dev tun
proto udp
remote VPN_SERVER_IP_PLACEHOLDER 443
resolv-retry infinite
nobind
persist-key
persist-tun
remote-cert-tls server
cipher AES-256-GCM
auth SHA256
key-direction 1
verb 3

<ca>
OUTER_EOF

# Replace placeholder with actual IP
sed -i "s/VPN_SERVER_IP_PLACEHOLDER/${VPN_SERVER_IP}/" ~/client-configs/${CLIENT_NAME}/${CLIENT_NAME}.ovpn

# Append CA certificate
cat pki/ca.crt >> ~/client-configs/${CLIENT_NAME}/${CLIENT_NAME}.ovpn

cat >> ~/client-configs/${CLIENT_NAME}/${CLIENT_NAME}.ovpn << "OUTER_EOF"
</ca>

<cert>
OUTER_EOF

# Append client certificate
cat pki/issued/${CLIENT_NAME}.crt >> ~/client-configs/${CLIENT_NAME}/${CLIENT_NAME}.ovpn

cat >> ~/client-configs/${CLIENT_NAME}/${CLIENT_NAME}.ovpn << "OUTER_EOF"
</cert>

<key>
OUTER_EOF

# Append client private key
cat pki/private/${CLIENT_NAME}.key >> ~/client-configs/${CLIENT_NAME}/${CLIENT_NAME}.ovpn

cat >> ~/client-configs/${CLIENT_NAME}/${CLIENT_NAME}.ovpn << "OUTER_EOF"
</key>

<tls-auth>
OUTER_EOF

# Append TLS auth key
cat ta.key >> ~/client-configs/${CLIENT_NAME}/${CLIENT_NAME}.ovpn

echo "</tls-auth>" >> ~/client-configs/${CLIENT_NAME}/${CLIENT_NAME}.ovpn

echo ""
echo "✅ Client configuration generated successfully!"
echo ""
echo "📁 Config file location:"
echo "   ~/client-configs/${CLIENT_NAME}/${CLIENT_NAME}.ovpn"
echo ""
echo "📥 Download to your local machine:"
echo "   gcloud compute scp ${VPN_VM_NAME}:~/client-configs/${CLIENT_NAME}/${CLIENT_NAME}.ovpn \\"
echo "     ~/Downloads/${CLIENT_NAME}.ovpn \\"
echo "     --zone=${ZONE} --project=${PROJECT_ID}"
echo ""
echo "🔌 Connect using:"
echo "   sudo openvpn ~/Downloads/${CLIENT_NAME}.ovpn"
echo ""
EOF

chmod +x ~/generate-vpn-client.sh

echo "✅ Client generation script created"
```

**Note:** Update `${ZONE}` and `${PROJECT_ID}` in the script output with your actual values, or manually edit after generation.

---

## Step 12: Generate Your First Client Certificate

```bash
# Generate certificate (replace with your name)
./generate-vpn-client.sh your.name

# Example:
./generate-vpn-client.sh ravi.ranjan
```

Exit the SSH session.

---

## Step 13: Download Client Config

On your **local machine**:

```bash
# Download the .ovpn file
gcloud compute scp ${VPN_VM_NAME}:~/client-configs/your.name/your.name.ovpn \
  ~/Downloads/your.name.ovpn \
  --zone=${ZONE} \
  --project=${PROJECT_ID}
```

---

## Step 14: Connect to VPN

### macOS/Linux:

```bash
# Install OpenVPN (one-time)
brew install openvpn  # macOS
# OR
sudo apt-get install openvpn  # Ubuntu

# Connect to VPN
sudo openvpn ~/Downloads/your.name.ovpn
```

You should see:
```
Initialization Sequence Completed
```

### Windows:

1. Install [OpenVPN GUI](https://openvpn.net/community-downloads/)
2. Copy `.ovpn` file to `C:\Program Files\OpenVPN\config\`
3. Right-click OpenVPN GUI → Run as Administrator
4. Right-click system tray icon → Connect

---

## Step 15: Test VPN Connection

Open a **new terminal** (keep VPN connected in the first one):

```bash
# Check your VPN IP (should be 10.8.0.x)
ifconfig tun0  # macOS/Linux
ipconfig       # Windows

# Create a test VM without public IP
gcloud compute instances create test-vm \
  --zone=${ZONE} \
  --machine-type=e2-micro \
  --network-interface=subnet=${VPC_NETWORK},no-address \
  --project=${PROJECT_ID}

# Get its private IP
TEST_VM_IP=$(gcloud compute instances describe test-vm \
  --zone=${ZONE} \
  --project=${PROJECT_ID} \
  --format="value(networkInterfaces[0].networkIP)")

echo "Test VM private IP: ${TEST_VM_IP}"

# SSH using private IP (while connected to VPN)
ssh ${TEST_VM_IP}
```

If you can SSH to the private IP, **VPN is working!** ✅

---

## Generate Additional Client Certificates

For team members:

```bash
# SSH to VPN server
gcloud compute ssh ${VPN_VM_NAME} --zone=${ZONE} --project=${PROJECT_ID}

# Generate certificate
./generate-vpn-client.sh <team-member-name>

# Exit and download
gcloud compute scp ${VPN_VM_NAME}:~/client-configs/<name>/<name>.ovpn \
  ~/Downloads/<name>.ovpn \
  --zone=${ZONE} \
  --project=${PROJECT_ID}
```

---

## Troubleshooting

### VPN won't connect

```bash
# SSH to VPN server
gcloud compute ssh ${VPN_VM_NAME} --zone=${ZONE} --project=${PROJECT_ID}

# Check OpenVPN status
sudo systemctl status openvpn-server@server

# Check logs
sudo tail -f /var/log/openvpn/openvpn.log

# Verify firewall rule
gcloud compute firewall-rules list --project=${PROJECT_ID} --filter="name=allow-openvpn-udp"
```

### Can't access private VMs

```bash
# Check routes (on your laptop while connected to VPN)
ip route | grep tun0

# Should see:
# 10.160.0.0/20 via 10.8.0.1 dev tun0  (or your VPC CIDR)

# SSH to VPN server and check IP forwarding
gcloud compute ssh ${VPN_VM_NAME} --zone=${ZONE} --project=${PROJECT_ID}
sysctl net.ipv4.ip_forward  # Should be 1

# Check NAT rules
sudo iptables -t nat -L -n -v
```

### Certificate generation fails

```bash
# SSH to VPN server
cd ~/openvpn-ca

# Check PKI is initialized
ls pki/

# Regenerate if needed
./easyrsa init-pki
./easyrsa --batch build-ca nopass
./easyrsa --batch gen-req server nopass
./easyrsa --batch sign-req server server
```

---

## Revoke Client Certificate

If a team member leaves or certificate is compromised:

```bash
# SSH to VPN server
gcloud compute ssh ${VPN_VM_NAME} --zone=${ZONE} --project=${PROJECT_ID}

cd ~/openvpn-ca

# Revoke certificate
./easyrsa revoke <client-name>
./easyrsa gen-crl

# Update server
sudo cp pki/crl.pem /etc/openvpn/server/
echo "crl-verify crl.pem" | sudo tee -a /etc/openvpn/server/server.conf
sudo systemctl restart openvpn-server@server
```

---

## Optional: Set Up DNS

Instead of using IP address, set up DNS:

```bash
# Option 1: Cloud DNS
gcloud dns managed-zones create ${PROJECT_ID}-vpn \
  --dns-name="${PROJECT_ID}.com" \
  --description="DNS for ${PROJECT_ID}" \
  --project=${PROJECT_ID}

gcloud dns record-sets create vpn.${PROJECT_ID}.com \
  --zone=${PROJECT_ID}-vpn \
  --type=A \
  --ttl=300 \
  --rrdatas=${VPN_EXTERNAL_IP}

# Option 2: Local hosts file (quick testing)
echo "${VPN_EXTERNAL_IP} vpn.${PROJECT_ID}.com" | sudo tee -a /etc/hosts
```

---

## Cost Estimate

**Monthly costs (approximate):**
- n2d-standard-2 VM: ~$50/month
- Static IP: ~$3/month
- Network egress: Variable
- **Total: ~$53-70/month**

**Cost optimization:**
- Use e2-medium: ~$25/month
- Use preemptible VM (requires auto-restart): ~$15/month
- Schedule VM to run only during work hours

---

## Security Best Practices

1. **Change default values** in Easy-RSA vars (country, org, email)
2. **Use strong passwords** for any additional authentication
3. **Regularly update** OpenVPN and system packages
4. **Monitor connections** via `/var/log/openvpn/openvpn-status.log`
5. **Limit source IPs** in firewall rule if you have fixed IPs
6. **Enable audit logging** for compliance
7. **Revoke certificates** immediately when team members leave

---

## Summary Checklist

After completion, you should have:

- [x] VPN server VM running
- [x] Static external IP assigned
- [x] Firewall rules configured
- [x] Routes for VPN clients
- [x] OpenVPN server running
- [x] Client certificate generation script
- [x] At least one client config file
- [x] Successful VPN connection test
- [x] Access to private VMs via VPN

---

## Quick Reference Commands

```bash
# Variables (set these first)
PROJECT_ID="your-project-id"
ZONE="asia-south1-c"
VPN_VM_NAME="vpn-server"

# Check VPN server status
gcloud compute ssh ${VPN_VM_NAME} --zone=${ZONE} --project=${PROJECT_ID}
sudo systemctl status openvpn-server@server

# Generate new client
gcloud compute ssh ${VPN_VM_NAME} --zone=${ZONE} --project=${PROJECT_ID}
./generate-vpn-client.sh <username>

# Download client config
gcloud compute scp ${VPN_VM_NAME}:~/client-configs/<name>/<name>.ovpn ~/Downloads/ --zone=${ZONE} --project=${PROJECT_ID}

# Connect to VPN
sudo openvpn ~/Downloads/<name>.ovpn

# View logs
gcloud compute ssh ${VPN_VM_NAME} --zone=${ZONE} --project=${PROJECT_ID}
sudo tail -f /var/log/openvpn/openvpn.log

# View active connections
sudo cat /var/log/openvpn/openvpn-status.log
```

---

## Next Steps

1. Test with multiple team members
2. Document VPN access procedures for your team
3. Set up monitoring/alerts for VPN server
4. Schedule regular certificate renewals (CA cert expires in 10 years)
5. Consider setting up Cloud Monitoring for VPN server metrics

---

**Questions or issues?** Review the troubleshooting section or check OpenVPN documentation at https://openvpn.net/community-resources/

**Done!** Your VPN server is ready for production use.
