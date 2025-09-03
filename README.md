# UDP-53-VPN

This walks thru a fresh Ubuntu Server to a setup where your OpenVPN server (clients on tun0) can toggle its exit between:
- Tor transparent proxy (tor-on)
- Commercial OpenVPN provider (vpn-on, creates tun2)
Both scripts are idempotent and clean up the other mode before enabling their own.

## 0) System prep
```
sudo apt update
sudo apt install -y openvpn easy-rsa tor obfs4proxy iptables conntrack iproute2 curl jq
# (Optional but handy)
sudo apt install -y resolvconf
```
Use nftables backend for iptables (Ubuntu default, but make sure):
```
sudo update-alternatives --set iptables /usr/sbin/iptables-nft || true
```
Kernel sysctls (routing-friendly defaults):
```
sudo tee /etc/sysctl.d/99-vpn-toggle.conf >/dev/null <<'EOF'
net.ipv4.ip_forward=1
net.ipv4.conf.all.rp_filter=2
net.ipv4.conf.default.rp_filter=2
net.ipv4.conf.all.src_valid_mark=1
net.ipv4.tcp_mtu_probing=1
EOF
sudo sysctl --system
```
## 1) Free UDP/53 for OpenVPN server (if you want the server on UDP/53)

systemd-resolved binds a stub listener on 127.0.0.53 which can conflict.
```
echo -e "[Resolve]\nDNSStubListener=no" | sudo tee /etc/systemd/resolved.conf
sudo systemctl restart systemd-resolved
# Ensure /etc/resolv.conf points to real upstream DNS (e.g., Cloudflare + Quad9):
echo -e "nameserver 1.1.1.1\nnameserver 9.9.9.9" | sudo tee /etc/resolv.conf
```
Verify port 53 is free (only OpenVPN should take it later):
```
sudo ss -lunp | grep ':53 ' || echo "53/udp free"
```
## 2) OpenVPN server (your clients connect here → tun0)
### 2.1 Create PKI with Easy-RSA (quick start)
```
make-cadir ~/easyrsa
cd ~/easyrsa
./easyrsa init-pki
./easyrsa build-ca nopass
./easyrsa build-server-full server nopass
./easyrsa gen-dh
sudo mkdir -p /etc/openvpn/server
sudo cp pki/ca.crt pki/private/server.key pki/issued/server.crt pki/dh.pem /etc/openvpn/server/
```
### 2.2 Minimal /etc/openvpn/server/server.conf
```
sudo tee /etc/openvpn/server/server.conf >/dev/null <<'EOF'
port 53
proto udp
dev tun0
user nobody
group nogroup

server 10.8.0.0 255.255.255.0
topology subnet
keepalive 10 120
persist-key
persist-tun

# Push default route; DNS is forcibly redirected by tor-on/vpn-on anyway
push "redirect-gateway def1"

# Certificates/keys
ca /etc/openvpn/server/ca.crt
cert /etc/openvpn/server/server.crt
key /etc/openvpn/server/server.key
dh /etc/openvpn/server/dh.pem

# Modern ciphers (OpenVPN 2.5+)
data-ciphers AES-256-GCM:AES-128-GCM:CHACHA20-POLY1305
data-ciphers-fallback AES-256-GCM
tls-version-min 1.2

# Silence
verb 3
EOF
```
Enable & start:
```
sudo systemctl enable --now openvpn-server@server
sudo systemctl status openvpn-server@server --no-pager
```
You should see it listening on 0.0.0.0:53/udp and tun0 created:
```
sudo ss -lunp | egrep 'openvpn|:53 '
ip -brief addr show tun0
```
## 3) Tor (transparent proxy mode)
### 3.1 /etc/tor/torrc (no deprecated options)
```
sudo tee /etc/tor/torrc >/dev/null <<'EOF'
Log notice syslog
ClientOnly 1
SocksPort 9050
# Transparent proxy ports (used by tor-on)
TransPort 9040
DNSPort 5353
AutomapHostsOnResolve 1
VirtualAddrNetworkIPv4 10.192.0.0/10
# Bridges OPTIONAL: add if your VPS region blocks Tor (obfs4proxy installed)
# UseBridges 1
# ClientTransportPlugin obfs4 exec /usr/bin/obfs4proxy
# Bridge obfs4 <IP:PORT> <FINGERPRINT> cert=<...> iat-mode=0
EOF
```
Start:
```
sudo systemctl enable --now tor@default
sudo ss -ltnup | egrep ':(9040|9050|5353)\s'
```
## 4) Commercial OpenVPN provider (exit tunnel → tun2)
Create the directory & drop your provider’s .ovpn:
```
sudo mkdir -p /etc/openvpn/provider
sudo cp ~/Downloads/provider.ovpn /etc/openvpn/provider/provider.ovpn
```
Edit the config to pin the device name and creds file:
```
# Remove any existing "dev" or "auth-user-pass" lines first, then add:
sudo sed -i '/^dev /d;/^auth-user-pass/d' /etc/openvpn/provider/provider.ovpn
printf 'dev tun2\nauth-user-pass /etc/openvpn/provider/creds\n' | sudo tee -a /etc/openvpn/provider/provider.ovpn
```
Create credentials (use your provider’s OpenVPN username/password):
```
sudo install -m 600 -o root -g root /dev/stdin /etc/openvpn/provider/creds <<'EOF'
<USERNAME>
<PASSWORD>
EOF
```
Systemd unit /etc/systemd/system/openvpn-client@provider.service:
```
sudo tee /etc/systemd/system/openvpn-client@provider.service >/dev/null <<'EOF'
[Unit]
Description=Commercial VPN client (%i)
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
ExecStart=/usr/sbin/openvpn --config /etc/openvpn/provider/%i.ovpn
Restart=on-failure
RestartSec=2

[Install]
WantedBy=multi-user.target
EOF
```
Reload systemd:
```
sudo systemctl daemon-reload
```
You do not start this manually; vpn-on will do it and verify tun2.
## 5) Use it
Tor exit:
```
sudo /usr/local/sbin/tor-on
ss -ltnup | egrep ':(9040|5353)\s'    # Tor listeners up (OPTIONAL - Troubleshooting)
```
Provider exit:
```
sudo /usr/local/sbin/vpn-on
ip -o link show tun2                  # should exist (OPTIONAL - Troubleshooting)
ip rule | grep 10.8.0.0/24            # points to table 201 (OPTIONAL - Troubleshooting)
ip route show table 201               # default via tun2/peer (OPTIONAL - Troubleshooting)
sudo iptables-nft -t nat -L POSTROUTING -v -n | head -20   # top SNAT to tun2 IP growing (OPTIONAL - Troubleshooting)
```
From a VPN client (behind tun0):
```
ping -c1 1.1.1.1
curl -4 https://ifconfig.co
```
Browse normally.
## 6) Quick Troubleshooting
RSTs to 10.8.0.x on tun2 tcpdump → SNAT missing/overridden.
Fix: ensure only one NAT rule:
```
sudo iptables-nft -t nat -L POSTROUTING -v -n
# Keep only: SNAT -o tun2 -s 10.8.0.0/24 to <tun2-IP> at top.
```
No tun2 → provider auth or .ovpn broken.
Check:
```
sudo journalctl -u openvpn-client@provider -e --no-pager | tail -80
```
Port 53 conflict when starting server → redo step 1 (free stub listener), then:
```
sudo systemctl restart openvpn-server@server
```
Still weird? Clear stale conntrack:
```
sudo conntrack -D -s 10.8.0.0/24 || true
```
