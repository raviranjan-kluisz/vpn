Absolutely. Here's the cleaned-up version of what you just built.

# Tailscale + Multiple VPN Gateway (OpenVPN / FortiVPN)

Goal:

```text
Mac
  ↓ Tailscale
VPN Gateway VM
  ├── FortiVPN (vij)
  ├── FortiVPN (mum)
  ├── FortiVPN (hyd)
  └── OpenVPN (future)
```

No VPN clients on your laptop.

---

# 1. Create Gateway VM

Ubuntu 22.04/24.04

Install Tailscale:

```bash
curl -fsSL https://tailscale.com/install.sh | sh
```

Login:

```bash
sudo tailscale up
```

Verify:

```bash
tailscale status
```

---

# 2. Enable Forwarding

Required for subnet routing.

```bash
echo 'net.ipv4.ip_forward=1' | sudo tee /etc/sysctl.d/99-tailscale.conf
sudo sysctl --system
```

Verify:

```bash
sysctl net.ipv4.ip_forward
```

Expected:

```text
net.ipv4.ip_forward = 1
```

---

# 3. Configure FortiVPNs

Create separate configs:

```text
/etc/openfortivpn/
├── vij.conf
├── mum.conf
├── hyd.conf
└── gcp.conf
```

Example:

```ini
host = vpn.company.com
port = 443
username = ravi
password = xxxx

set-dns = 0
pppd-use-peerdns = 0
set-routes = 0
pppd-ifname = ppp-vpn-name
```

Important:

```ini
set-routes = 0
```

prevents VPNs from fighting over routes.

Use a stable PPP interface name per VPN:

```ini
# /etc/openfortivpn/hyd.conf
pppd-ifname = ppp-hyd

# /etc/openfortivpn/mum.conf
pppd-ifname = ppp-mum

# /etc/openfortivpn/vij.conf
pppd-ifname = ppp-vij
```

This avoids depending on kernel-assigned names like `ppp0`, `ppp1`, and
`ppp2`, which can change when services restart in a different order.

---

# 4. Run VPNs with systemd

Each VPN runs as its own service. This keeps the tunnels alive after SSH disconnects and lets systemd restart them after FortiGate session timeout.

Install the route helper first:

```bash
sudo tee /usr/local/sbin/vpn-routes.sh >/dev/null <<'EOF'
#!/usr/bin/env bash
set -euo pipefail

vpn="${1:-}"

case "$vpn" in
  hyd)
    iface="ppp-hyd"
    routes=("10.2.0.0/24")
    ;;
  mum)
    iface="ppp-mum"
    routes=("10.2.1.0/24")
    ;;
  vij)
    iface="ppp-vij"
    routes=("10.10.10.0/24")
    ;;
  *)
    echo "usage: $0 {hyd|mum|vij}" >&2
    exit 2
    ;;
esac

last_error=""
for _ in {1..90}; do
  if ip link show "$iface" >/dev/null 2>&1; then
    ok=1
    for route in "${routes[@]}"; do
      if out=$(ip route replace "$route" dev "$iface" 2>&1); then
        echo "installed route $route dev $iface for $vpn"
      else
        ok=0
        last_error="$out"
        break
      fi
    done
    if [ "$ok" -eq 1 ]; then
      exit 0
    fi
  else
    last_error="interface $iface not found"
  fi
  sleep 1
done

echo "failed to install routes for $vpn on $iface: $last_error" >&2
exit 1
EOF

sudo chmod 755 /usr/local/sbin/vpn-routes.sh
```

Hyd:

```ini
# /etc/systemd/system/openfortivpn-hyd.service
[Unit]
Description=OpenFortiVPN HYD
After=network-online.target tailscaled.service
Wants=network-online.target

[Service]
Type=simple
ExecStart=/usr/bin/openfortivpn -c /etc/openfortivpn/hyd.conf
ExecStartPost=/usr/local/sbin/vpn-routes.sh hyd
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

Mum:

```ini
# /etc/systemd/system/openfortivpn-mum.service
[Unit]
Description=OpenFortiVPN MUM
After=network-online.target tailscaled.service
Wants=network-online.target

[Service]
Type=simple
ExecStart=/usr/bin/openfortivpn -c /etc/openfortivpn/mum.conf
ExecStartPost=/usr/local/sbin/vpn-routes.sh mum
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

Vij:

```ini
# /etc/systemd/system/openfortivpn-vij.service
[Unit]
Description=OpenFortiVPN VIJ
After=network-online.target tailscaled.service
Wants=network-online.target

[Service]
Type=simple
ExecStart=/usr/bin/openfortivpn -c /etc/openfortivpn/vij.conf
ExecStartPost=/usr/local/sbin/vpn-routes.sh vij
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

Enable and start:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now openfortivpn-hyd.service
sudo systemctl enable --now openfortivpn-mum.service
sudo systemctl enable --now openfortivpn-vij.service
```

---

# 5. Verify VPN Interfaces

Check services:

```bash
systemctl status openfortivpn-hyd.service
systemctl status openfortivpn-mum.service
systemctl status openfortivpn-vij.service
```

Check PPP interfaces:

```bash
ip -br addr | grep ppp
```

Expected on `ravi-vpn`:

```text
ppp-hyd -> hyd
ppp-mum -> mum
ppp-vij -> vij
```

Confirm from logs:

```bash
sudo journalctl -u openfortivpn-hyd.service --no-pager | grep 'Using interface'
sudo journalctl -u openfortivpn-mum.service --no-pager | grep 'Using interface'
sudo journalctl -u openfortivpn-vij.service --no-pager | grep 'Using interface'
```

---

# 6. Persistent Routes

Because:

```ini
set-routes = 0
```

routes are installed by `/usr/local/sbin/vpn-routes.sh` from each service's
`ExecStartPost`.

### vij

Need access:

```text
10.10.10.39
```

Add:

```bash
sudo /usr/local/sbin/vpn-routes.sh vij
```

Verify:

```bash
ip route get 10.10.10.39
```

Expected:

```text
10.10.10.39 dev ppp-vij
```

Test:

```bash
ping 10.10.10.39
```

---

### mum

Need access:

```text
10.2.1.0/24
```

Add:

```bash
sudo /usr/local/sbin/vpn-routes.sh mum
```

Verify:

```bash
ip route get 10.2.1.1
```

Expected:

```text
10.2.1.1 dev ppp-mum
```

---

### hyd

Need access:

```text
10.2.0.11
10.2.0.12
10.2.0.13
10.2.0.14
```

Add:

```bash
sudo /usr/local/sbin/vpn-routes.sh hyd
```

Verify:

```bash
ip route get 10.2.0.11
```

Expected:

```text
10.2.0.11 dev ppp-hyd
```

Test:

```bash
ping 10.2.0.11
```

---

# 7. Advertise Routes to Tailscale

```bash
sudo tailscale up \
  --advertise-routes=10.10.10.0/24,10.2.0.0/24,10.2.1.0/24
```

Verify:

```bash
tailscale debug prefs | sed -n '/AdvertiseRoutes/,+10p'
```

Expected:

```json
"AdvertiseRoutes": [
  "10.10.10.0/24",
  "10.2.0.0/24",
  "10.2.1.0/24"
]
```

---

# 8. Approve Routes

Open:

```text
https://login.tailscale.com/admin/machines
```

Select:

```text
ravi-vpn
```

Approve:

```text
10.10.10.0/24
10.2.0.0/24
10.2.1.0/24
```

Without this step, subnet routing will not work.

---

# 9. Configure Laptop

Install Tailscale.

Login with the same account.

Verify:

```bash
tailscale status
```

Should show:

```text
ravi-vpn
ravis-macbook-pro
```

Accept routes:

```bash
sudo tailscale up --accept-routes
```

---

# 10. Test

Verify VM:

```bash
tailscale ping ravi-vpn
```

Test subnet routes:

```bash
ping 10.10.10.39
ping 10.2.0.11
ping 10.2.1.1
```

Open iDRAC:

```text
https://10.10.10.39
```

---

# 11. Useful Commands

VPN interfaces:

```bash
ip addr | grep ppp
```

Routes:

```bash
ip route | grep ppp
```

Current routing decision:

```bash
ip route get <IP>
```

Tailscale status:

```bash
tailscale status
```

Tailscale diagnostics:

```bash
tailscale ping <node>
```

---

# Future Expansion

Add more VPNs:

```text
ppp-hyd        -> hyd
ppp-mum        -> mum
ppp-vij        -> vij
ppp-customer-a -> customer-a
tun0 -> gcp-openvpn
```

Routes:

```bash
10.2.0.0/24   -> ppp-hyd
10.2.1.0/24   -> ppp-mum
10.10.10.0/24 -> ppp-vij
10.5.0.0/16   -> ppp-customer-a
172.16.0.0/16 -> tun0
```

Advertise:

```bash
sudo tailscale up \
  --advertise-routes=10.10.10.0/24,10.2.0.0/24,10.2.1.0/24,10.5.0.0/16,172.16.0.0/16
```

This scales surprisingly well for 5–10 VPNs as long as the subnets don't overlap. If two VPNs expose the same network (e.g. both use `10.0.0.0/8`), you'll need policy-based routing or separate gateway VMs.
