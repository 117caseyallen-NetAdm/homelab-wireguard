# Troubleshooting Log

Every real problem hit during the build, what it looked like, and what it actually
was. Kept because the symptoms are generic but the causes weren't obvious.

## Methodology that worked

One capture point answers the biggest question first. `tcpdump` on the WireGuard
host, filtered to the listen port, splits the entire path in half:

```bash
tcpdump -ni eth0 udp port 51820
```

* Packets arriving → edge (router forward, firewall NAT/policy) is fine; look at
  the server or the return path.
* Silence → the problem is upstream; stop staring at the server.

Paired with the client's transfer counters (`X sent, 0 received` = handshakes
leaving but nothing coming back), most failure modes localize in one step.

## The big one

**Symptom:** NAT rule hit counter climbing, `show session all` empty,
`flow_policy_deny` incrementing in `show counter global filter severity drop delta yes`.

**Cause:** the security rule's destination address was set to the *post-NAT*
(translated) address. PAN-OS evaluates security policy against the **pre-NAT
destination address** and the **post-NAT destination zone** — translation doesn't
happen until egress, so at policy-lookup time the packet still carries the original
destination IP. The rule never matched; `interzone-default` denied silently.

**Fix:** destination zone = post-NAT zone (inside), destination address = pre-NAT
address (the WAN interface IP). One field, commit, tunnel up.

**Order of operations, engraved:** route lookup → NAT *lookup* (determines
post-NAT zone) → security policy (pre-NAT IPs, post-NAT zones) → translation at
egress.

## Endpoint set to an inside address

**Symptom:** client shows bytes sent, zero received; `tcpdump` on the server shows
nothing; firewall shows no sessions. Everything looks dead everywhere.

**Cause:** during config edits, the client `Endpoint` got set to the firewall's
*private* WAN-side address instead of the public DDNS hostname. From an outside
network, handshakes toward an RFC1918 address never leave the client machine.

**Lesson:** the endpoint is what the client dials **from outside** — the public
name. The inside address is where the router forwards it *after* arrival. When
"nothing anywhere sees any packets," read the client config first; the cheapest
check is the one that requires no CLI.

## Missing port forward

**Symptom:** zero packets at the server, no sessions at the firewall — genuinely
nothing arriving.

**Cause:** the home-router port forward was planned, discussed, and never created.

**Lesson:** the boring explanation first. Before packet captures and drop
counters, confirm every hop in the chain *exists*. A checklist beats intuition:
DNS resolves → forward exists (and is **UDP**) → NAT rule → security rule →
committed (`show config diff` empty).

## Key placement during a client key rotation

**Symptom:** after rotating the laptop's keypair, the tunnel would not come up.
Windows `ping` returned `General failure`, then later `Request timed out`, while
`ping 8.8.8.8` worked normally.

**Reading those two messages matters.** `General failure` means Windows could not
send the packet at all — no route, interface not up. `Request timed out` means
packets are leaving and nothing is coming back. The symptom changing from the
first to the second was progress: the interface had come up, and the problem had
moved to the handshake.

On the server, `wg show` listed the peer with **no `endpoint`, no `latest
handshake`, and no `transfer`** — meaning the server had never accepted this
client, not once.

**Cause:** the wrong key in the wrong `[Peer]` block. Each `[Peer]` section
describes the *other* machine:

| File | Section | Holds |
|---|---|---|
| `wg0.conf` (server) | `[Interface] PrivateKey` | server's private key |
| `wg0.conf` (server) | `[Peer] PublicKey` | **client's** public key |
| client config | `[Interface] PrivateKey` | client's private key |
| client config | `[Peer] PublicKey` | **server's** public key |

Your own public key never belongs in your own `[Peer]` block. The client's public
key is never stored on the client at all — WireGuard derives it from the private
key and displays it in the UI.

**WireGuard is silent by design when it cannot authenticate a peer.** No error,
no rejection, no log line. A key mismatch and a firewall blocking the path look
identical from the client, which is excellent security and difficult
troubleshooting.

**Recovering keys without guessing:**

```bash
wg show wg0 public-key             # server's public key -> client's [Peer]
wg show wg0 peers                  # what the server expects -> must match the
                                   # public key the client UI displays
cat <private>.key | wg pubkey      # derive a public key from any private key
```

A public key is always recoverable from its private key, so it cannot be
permanently lost. Only private keys are irreplaceable.

**And config changes need an explicit reload on both ends** —
`systemctl restart wg-quick@wg0` on the server (or `wg syncconf wg0 <(wg-quick
strip wg0)` to apply without dropping the interface), and deactivate/reactivate
the tunnel in the client. Editing a config while the tunnel is running changes
nothing.

## Noise worth recognizing (not chasing)

* `sysctl: permission denied` on host-namespace keys inside an unprivileged
  container — expected; only the keys you actually set need to take.
* `/dev/net/tun` owned by `nobody nogroup` — expected UID mapping, works fine.
* Firewall/switch mgmt interfaces not answering ping from new source subnets —
  management-profile/ACL scoping, not a routing failure. Prove the path with a
  device that *does* answer before touching route tables.
