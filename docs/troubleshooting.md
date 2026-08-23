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

## The phantom DOWN interface

**Symptom:** one second after `pct start`, the container's `eth0` was DOWN with no
address. Config was correct, bridge membership was correct, service showed no errors.

**Cause:** nothing. ifupdown2 inside the container took ~16 seconds to finish;
the check ran at second one. A re-check after boot completed showed everything up.

**Lesson:** timestamp the service (`systemctl status networking`) before declaring
a failure. `Active: active (exited)` with a finish time *after* the failed check =
you photographed the middle of boot.

## Missing port forward

**Symptom:** zero packets at the server, no sessions at the firewall — genuinely
nothing arriving.

**Cause:** the home-router port forward was planned, discussed, and never created.

**Lesson:** the boring explanation first. Before packet captures and drop
counters, confirm every hop in the chain *exists*. A checklist beats intuition:
DNS resolves → forward exists (and is **UDP**) → NAT rule → security rule →
committed (`show config diff` empty).

## Shell traps (cost more time than the network did)

* **Pasting a block after `su -` / `pct enter`** — everything after the first line
  is swallowed; the subshell isn't reading stdin yet. Paste the shell-entering
  command alone, wait for the prompt, then paste the rest.
* **`su` vs `su -`** — plain `su` keeps the old user's PATH; `/usr/sbin` tools
  (`sysctl`, `modprobe`, `bridge`) come back "command not found" and everything
  redirected into `/etc` fails with permission denied. The login shell (`su -`)
  fixes both. Related: "command not found" for admin tools plus pmxcfs
  `ipcc_send_rec` errors on Proxmox = you're not root, not a broken box.
* **Bracketed paste artifacts** — commands arriving as `^[[200~cmd~` fail
  mysteriously. Type the command by hand, or disable with `printf '\e[?2004l'`.
* **Case sensitivity** — `adduser Admin` then `usermod ... admin` are two
  different users.

## Noise worth recognizing (not chasing)

* `sysctl: permission denied` on host-namespace keys inside an unprivileged
  container — expected; only the keys you actually set need to take.
* `/dev/net/tun` owned by `nobody nogroup` — expected UID mapping, works fine.
* Firewall/switch mgmt interfaces not answering ping from new source subnets —
  management-profile/ACL scoping, not a routing failure. Prove the path with a
  device that *does* answer before touching route tables.
