# Build Notes

Step-by-step record of the build, in the order it actually happened. Environment:
Proxmox VE on a 2013 Mac Pro (node `PROX-LAB`, mgmt `10.99.20.10`), dual-site lab
fabric (WEST = PA-440 + 3560CG-2, EAST = SRX345 + 3560CG-1 + Arista 710P), flat
OSPF area 0 across a route-based IPsec tunnel.

## 1. Host prep (Proxmox)

WireGuard runs in an **unprivileged LXC**, which cannot load kernel modules. The
host must provide both `wireguard` and `tun`:

```bash
modprobe wireguard && modprobe tun
echo wireguard > /etc/modules-load.d/wireguard.conf
echo tun       > /etc/modules-load.d/tun.conf
pveam update && pveam download local debian-13-standard_13.6-1_amd64.tar.zst
```

## 2. Bridge selection

The host has two bridges:

* `vmbr0` — plain bridge, NIC on a switch **access** port (VLAN 99), carries the
  host mgmt IP. Deliberately untouched: no risk of dropping the web UI mid-change.
* `vmbr1` — VLAN-aware (`bridge-vids 20 99`), NIC on a **dot1q trunk** allowing
  20,99. Two stacked plain bridges ride its subinterfaces: `data20` on `vmbr1.20`
  and `mgmt99` on `vmbr1.99`.

The container attaches to `bridge=data20` with **no `tag=` parameter** — `data20`
*is* VLAN 20. Verify the switch side is genuinely trunking before blaming anything
else (`show interfaces trunk`), and verify the veth actually enslaved:

```bash
ls /sys/class/net/data20/brif/     # expect veth101i0 alongside vmbr1.20
```

## 3. Container create

```bash
pct create 101 local:vztmpl/debian-13-standard_13.6-1_amd64.tar.zst \
  --hostname CA-WG-LAB \
  --cores 1 --memory 512 --swap 512 \
  --rootfs local-lvm:4 \
  --net0 name=eth0,bridge=data20,ip=10.20.2.50/24,gw=10.20.2.1 \
  --nameserver 1.1.1.1 \
  --features nesting=1 \
  --onboot 1 --unprivileged 1 \
  --password
```

512 MB / 1 core / 4 GB is generous for WireGuard; it's a kernel-space data plane
with a tiny userspace footprint.

## 4. TUN passthrough

Not automatic on this PVE version. With the container **stopped**, append to
`/etc/pve/lxc/101.conf`:

```
lxc.cgroup2.devices.allow: c 10:200 rwm
lxc.mount.entry: /dev/net/tun dev/net/tun none bind,create=file
```

LXC reads this file only at container start — stop/start, not reboot-from-inside.
Inside the container, expect:

```
crw-rw-rw- 1 nobody nogroup 10, 200 /dev/net/tun
```

`nobody nogroup` is the unprivileged UID mapping and is **normal** — mode 666
means container root can still open it.

## 5. Fabric validation before touching WireGuard

Three pings from inside the container, each proving a different layer:

```bash
ping 10.20.2.1     # VLAN 20 tagging + local SVI
ping 1.1.1.1       # PA source-NAT for the data subnet (apt needs this)
ping 10.99.10.14   # far-site device: crosses OSPF + the IPsec tunnel (TTL 60 = 4 hops)
```

If any fail, fix the fabric first. Debugging WireGuard on top of a broken underlay
is misery.

## 6. Keys and server config

```bash
umask 077                      # keys land 600, wg-quick stays quiet
mkdir -p /etc/wireguard
wg genkey | tee /etc/wireguard/server_private.key | wg pubkey > /etc/wireguard/server_public.key
wg genkey | tee /etc/wireguard/laptop_private.key | wg pubkey > /etc/wireguard/laptop_public.key
echo "net.ipv4.ip_forward = 1" > /etc/sysctl.d/99-wireguard.conf
sysctl --system
```

Server config: [`configs/wg0.conf.example`](../configs/wg0.conf.example).
Two deliberate choices:

* **MTU 1380** — inner traffic to the far site crosses IPsec at MTU 1400. The
  wg-quick default of 1420 fragments or blackholes ("ping works, SSH hangs").
* **No PostUp MASQUERADE** — routed design. The fabric routes back to the pool
  (section 8), so client source IPs stay real.

```bash
systemctl enable --now wg-quick@wg0
wg show
```

## 7. Edge: DDNS + port forward (home router)

* DDNS: router keeps `vpn.example.com` pointed at the current residential WAN IP.
  Clients dial the hostname, so an ISP lease rotation costs one reconnect instead
  of editing every client config from somewhere you can't reach home.
* Port forward: external **UDP**/51820 → PA-440 WAN interface. UDP explicitly —
  WireGuard is UDP-only and a TCP forward produces perfect silence.

## 8. Return routing (distribution switch, Cisco IOS)

Full config: [`configs/3560cg2-return-route.txt`](../configs/3560cg2-return-route.txt).
Static route for the pool toward the WireGuard host, redistributed into OSPF
behind a prefix-list route-map. Verify the Type-5 LSA locally, then confirm the
far-site switch installs it as `O E2` — that's the proof the pool crossed the
IPsec tunnel.

## 9. Firewall: dst-NAT + security policy (PAN-OS)

Full config: [`configs/pa440-nat-security.txt`](../configs/pa440-nat-security.txt).

* NAT rule: from WAN, **to WAN** (pre-NAT zone of the destination), dest = WAN
  interface IP, service UDP/51820 → translate to the WireGuard host.
* Security rule: from WAN, to **WEST-LAN** (post-NAT zone), destination =
  **the WAN interface IP (pre-NAT address)**, service UDP/51820, allow.

That last field is the classic PAN-OS dst-NAT trap — see
[troubleshooting.md](troubleshooting.md#the-big-one).

## 10. Client config

Template: [`configs/client-laptop.conf.example`](../configs/client-laptop.conf.example).

* `Address` = one /32 from the pool, matching the server's `AllowedIPs` for that peer
* `AllowedIPs` = split tunnel: lab subnets only; general traffic stays local
* `Endpoint` = **the DDNS hostname** — the public-facing name, never an inside
  address (see troubleshooting for how that mistake presents)
* `PersistentKeepalive = 25` keeps the NAT mapping alive through both the home
  router and the PA

Each additional client gets its own keypair, its own /32, and its own `[Peer]`
block on the server. Never reuse keys across devices.

## Verification

From a network you don't control (phone hotspot at minimum):

```
ping 10.20.2.1        # near-side SVI — basic tunnel function
ping 10.99.10.14      # far-site device — proves OSPF + IPsec + return routing
```

Expected non-answers that are *not* VPN failures: Windows hosts (inbound ICMP off
by default — RDP works while ping doesn't), firewall interfaces without an ICMP
management profile.

## Future work

* Per-client firewall policy on the PA (the payoff of the routed design)
* Split-DNS for the lab domain — the client `DNS =` already points at the domain
  controller, see [`configs/client-laptop.conf.example`](../configs/client-laptop.conf.example)
* Additional peers: phone, second laptop
