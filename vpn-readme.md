# VPN Setup Notes

## FortiClient ZTNA

FortiClient on Linux doesn't create a tunnel interface (`tun0`). Instead it:
1. Adds a secondary IP (e.g. `10.167.32.42/32`) to the physical interface
2. Adds routes for corporate subnets with that IP as the source
3. Intercepts packets sourced from that IP via kernel hooks and tunnels them over DTLS/HTTPS to the FortiGate gateway
4. Runs `fctdns` on `127.0.0.1:53` to intercept DNS and forward to corporate DNS servers

### Checking VPN status
```bash
ip addr show enp0s31f6 | grep 10.167       # VPN IP assigned?
ip route | grep 10.167                       # Routes in place?
nc -zw2 10.169.2.54 53                       # Corporate DNS reachable?
dig @127.0.0.1 <internal-hostname>           # DNS resolution working?
nc -zw5 <internal-host> 22                   # SSH port reachable?
```

The VPN IP (e.g. `10.167.32.x`) changes on each reconnect. If forwarding to another machine stops working after a VPN reconnect, re-run `sudo bash ~/route.sh` — it now detects the current IP automatically.

### Known issue: DNS on connection
FortiClient may fail to connect with "IPsec VPN failed" if DNS is not resolving correctly on the current network. Verify DNS is working before attempting to connect.

### Known issue: Snap browsers and custom URI schemes
Firefox and Chromium installed as Snaps cannot handle the `fabricagent://` URI scheme for EMS onboarding. Workaround: copy the `fabricagent://...` URL from the browser and run it manually:
```bash
xdg-open 'fabricagent://ems/onboarding?username=ajlee&auth_token=TOKEN_HERE'
```

## Forwarding VPN to another machine (route.sh)

When on the office network (`10.12.34.0/24`), another machine can use this machine as a gateway to reach corporate resources through the FortiClient VPN.

### Setup (this machine - gateway)
```bash
sudo bash ~/route.sh
```

`route.sh` adds:
- SNAT rules to rewrite forwarded traffic source to the current VPN IP (auto-detected from `enp0s31f6`)
- TCP MSS clamping to 1130 to avoid MTU black holes on the VPN path

**Note:** The VPN IP changes on each reconnect. `route.sh` now detects it dynamically, so re-running it after reconnect is sufficient. It also cleans up any stale rules from previous runs before adding new ones.

### Setup (other machine)
```bash
sudo route add -net 10.169.0.0 netmask 255.255.0.0 gw 10.12.34.217
# Add other subnets as needed
```

### Teardown (this machine)
```bash
sudo bash ~/routedown.sh
```

This removes only the FortiClient forwarding rules without affecting Docker or WireGuard.

## MTU Issues

MTU problems are the most common cause of VPN connections that partially work: ping succeeds but SSH hangs at key exchange (`SSH2_MSG_KEX_ECDH_REPLY`). Small packets get through, large packets (especially the `sntrup761x25519` SSH key exchange at ~1700 bytes) are silently dropped.

### Diagnosing
```bash
# Find the path MTU to a destination
ping -M do -s 1400 -c 1 -W 3 <destination>
# -M do sets the "Don't Fragment" flag
# Total packet size = -s value + 28 bytes (IP + ICMP headers)
# Binary search: if it fails, go lower; if it works, go higher
```

### Quick SSH workaround
Force a smaller key exchange algorithm:
```bash
ssh -o KexAlgorithms=curve25519-sha256 user@host
```

### FortiClient path MTU (office ethernet)
- Interface MTU: 1280
- Actual path MTU through VPN: 1198 (82 bytes DTLS overhead)
- TCP MSS clamp needed: 1130

### WireGuard MTU
WireGuard adds ~60 bytes of overhead. The default `MTU = 1420` assumes a 1500-byte network. On networks with lower MTU (e.g. 1480 due to PPPoE), this causes the same SSH key exchange failures.

- Office network (MTU 1500): `MTU = 1420` works
- Home WiFi (MTU 1480): need `MTU = 1360` in `/etc/wireguard/wg1.conf`

### Why PMTU discovery fails
TCP always sets the Don't Fragment flag and relies on routers sending back ICMP "Fragmentation Needed" messages. When firewalls drop these ICMP messages, TCP never learns the path is too small, keeps retransmitting oversized packets, and the connection hangs. MSS clamping (`iptables -j TCPMSS`) works around this by rewriting the maximum segment size in SYN packets at the gateway.
