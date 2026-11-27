---
name: linux-networking
description: Debug Linux hosts and network paths. Use when a connection times out, is refused, or resets; when DNS resolves wrongly or slowly; when a host is out of memory, disk, inodes, or file descriptors; when a systemd unit will not start or keeps restarting; when TLS handshakes fail; when throughput is bad or MTU/path MSS is suspected; or when you need to find which process owns a port, a file, or the CPU.
---

# Linux & networking

Diagnosis is a ladder. Climb it in order — most time is lost by starting in the
middle and assuming a layer that is actually broken.

```
1. Is the host healthy?        load, memory, disk, inodes, fds
2. Is the process running?     systemd state, exit codes, logs
3. Is it listening?            ss -lntp, and on which address
4. Can packets reach it?       ip route, firewall, MTU
5. Does the name resolve?      dig, /etc/resolv.conf, nsswitch
6. Does the handshake work?    TCP, then TLS
```

## Fast triage

```bash
uptime                       # load averages: 1, 5, 15 min
free -h                      # look at "available", not "free"
df -h                        # disk
df -i                        # inodes — a separate way to run out of disk
ss -s                        # socket summary
dmesg -T | tail -50          # OOM kills, hardware, network resets
journalctl -p err -b --no-pager | tail -50    # errors this boot
```

**`free` reports `available`, and that is the number that matters.** Low `free`
with high `available` is a healthy machine using its page cache. Alerting on
`free` produces false pages forever.

Load average is per-runqueue, not a percentage: compare it to core count
(`nproc`). On Linux it also counts uninterruptible-sleep tasks, so a load spike
with idle CPU means I/O wait, not CPU pressure.

## Connection failed — read the error literally

| Symptom | Meaning | Next step |
|---|---|---|
| `Connection refused` | Reached the host, **nothing listening** on that port | `ss -lntp` on the target |
| `Connection timed out` | No response at all — dropped by a firewall, or wrong route | `ip route get`, check filtering |
| `Connection reset by peer` | Something actively closed it — app crash, LB idle timeout, or a middlebox | app logs, then `tcpdump` |
| `No route to host` | Local routing has no path, or an ICMP unreachable came back | `ip route get <ip>` |
| `Name or service not known` | DNS, not connectivity | `dig`, `/etc/resolv.conf` |
| Hangs after connecting | TCP is fine; the application protocol or TLS is stuck | `openssl s_client`, app logs |

Refused vs timed out is the single most valuable distinction in network
debugging. Refused means you got there.

## Who is listening

```bash
ss -lntp                     # TCP listeners with process
ss -lnup                     # UDP
ss -tanp                     # all TCP sockets with state and process
ss -tan state time-wait      # filter by state
ss -tanp 'sport = :8080'     # by port
```

`0.0.0.0:8080` accepts from anywhere; `127.0.0.1:8080` accepts only local
traffic. A service that works locally and refuses remotely is almost always bound
to loopback. In a Kubernetes pod, every container shares one network namespace,
so a sidecar can reach the app on `127.0.0.1`, but nothing outside the pod can.

`ss -s` gives the summary. A large `timewait` count is usually normal; a large
`orphaned` count is not.

## Routing and interfaces

```bash
ip -br addr                  # brief: interface, state, addresses
ip -br link                  # state and MAC
ip route                     # the table
ip route get <dest-ip>       # which route WILL be used, including source IP
ip neigh                     # ARP cache
```

`ip route get` is better than reading the table by hand — it applies the actual
selection logic including policy routing, and tells you the source address that
will be used. Asymmetric routing shows up here.

## DNS

```bash
dig +short <name>
dig +trace <name>            # full delegation path, bypasses the resolver cache
dig @<server> <name>         # ask a specific server
dig +short -x <ip>           # reverse
resolvectl status            # on systemd-resolved hosts
cat /etc/resolv.conf
cat /etc/nsswitch.conf       # order: files, dns, ...
```

`dig` talks to DNS directly; applications go through NSS. So `dig` succeeding
while the app fails means `/etc/hosts`, `nsswitch.conf`, or a caching layer —
not DNS. `getent hosts <name>` follows the same path the application does and is
the better comparison.

## Systemd

```bash
systemctl status <unit>
systemctl list-units --failed
journalctl -u <unit> -n 200 --no-pager
journalctl -u <unit> -f
journalctl -u <unit> --since "10 min ago"
systemd-analyze blame        # slowest units at boot
```

`Active: failed (Result: exit-code)` with `status=1/FAILURE` is an app error.
`Result: timeout` means the unit did not signal readiness in time — often a
`Type=notify` service that never called `sd_notify`, or `Type=forking` that did
not fork as expected.

A unit that restarts forever hits `StartLimitBurst` and then stops with
`start request repeated too quickly`. That message describes the rate limiter,
**not the failure** — scroll further back for the real error.

## References

| File | Covers |
|---|---|
| [references/connectivity.md](references/connectivity.md) | tcpdump, conntrack, firewalls, MTU, TLS handshakes |
| [references/resources.md](references/resources.md) | CPU, memory, OOM, disk, inodes, file descriptors, cgroups |
| [references/processes-and-systemd.md](references/processes-and-systemd.md) | Unit files, restart policy, journald, signals, limits |

## Gotchas

- **`df` says space is free but writes fail** → out of inodes (`df -i`), or a
  deleted file still held open by a process (`lsof +L1`). Deleting the file does
  not reclaim space until the holder closes it or restarts.
- **`free` shows almost no free memory** → page cache. Read `available`.
- **Ping works, TCP does not** → ICMP and TCP are filtered separately. Ping
  proves routing, nothing more.
- **Works for small requests, hangs on large ones** → MTU/PMTU black hole.
  Classic on VPNs and overlay networks.
- **`localhost` resolves to `::1` first** on dual-stack hosts, so a service bound
  only to IPv4 refuses connections that `127.0.0.1` would accept.
- **Ephemeral port exhaustion** looks like random connection failures under load:
  check `net.ipv4.ip_local_port_range` and the `time-wait` count.
- **`iptables` on a modern host may be an `nft` shim.** `iptables -L` can show an
  incomplete picture; use `nft list ruleset` for the truth.
