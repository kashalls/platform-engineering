# Packets, firewalls, MTU, and TLS

## tcpdump

```bash
tcpdump -i any -nn 'host 10.0.0.5 and port 443' -c 100
tcpdump -i any -nn 'tcp[tcpflags] & (tcp-syn|tcp-rst) != 0'    # handshakes and resets
tcpdump -i any -nn -w /tmp/cap.pcap 'port 5432'                # write for Wireshark
tcpdump -r /tmp/cap.pcap -nn -A                                # read back with ASCII
```

`-nn` disables name resolution for both hosts and ports — without it tcpdump
issues DNS lookups while you are debugging DNS, and blocks.

Reading a failed connection:

- **SYN, no reply** — dropped. Firewall, security group, or a black hole.
- **SYN → RST** — actively refused. Nothing listening, or a firewall configured
  to reject rather than drop.
- **SYN → SYN-ACK → ACK, then nothing** — TCP is fine. The problem is above it:
  TLS, the application protocol, or an app that never writes.
- **RST mid-stream** — an idle timeout on a load balancer or NAT device, or an
  application crash.
- **Retransmissions** — packet loss; check the path with `mtr`.

## Path testing

```bash
mtr -rwzbc 100 <host>        # report mode: loss and latency per hop
traceroute -T -p 443 <host>  # TCP traceroute, gets through ICMP filters
nc -zv <host> <port>         # is the port open
nc -zvu <host> <port>        # UDP (unreliable by nature)
curl -sv --connect-timeout 5 https://<host>/ -o /dev/null
```

**Loss at an intermediate hop is usually meaningless** — routers deprioritize
ICMP responses to themselves while forwarding perfectly. Only loss that persists
to the final hop is real. This misreads more traceroutes than anything else.

`curl -sv` output shows exactly where it stops: DNS, TCP connect, TLS handshake,
or the request itself. Use `curl -w` for timing breakdown:

```bash
curl -s -o /dev/null -w 'dns:%{time_namelookup} conn:%{time_connect} tls:%{time_appconnect} ttfb:%{time_starttransfer} total:%{time_total}\n' https://<host>/
```

That one line separates a DNS problem from a TLS problem from a slow backend.

## Firewalls

```bash
nft list ruleset                        # the real ruleset on modern hosts
iptables -L -n -v --line-numbers        # legacy view, may be an nft shim
iptables -t nat -L -n -v                # NAT rules
```

On most current distributions `iptables` is a compatibility shim over nftables.
Rules created natively with `nft` are **invisible to `iptables -L`**, so a host
can appear to have no rules while dropping your traffic. Check `nft` first.

Counters are the fast diagnostic: `-v` shows packet and byte counts per rule.
Retry the failing connection and see which rule's counter increments.

`REJECT` sends an RST or ICMP unreachable (fast failure, "connection refused");
`DROP` sends nothing (client waits for its timeout). The client-side symptom
tells you which one is in play.

## conntrack

```bash
conntrack -L                             # current tracked connections
conntrack -S                             # per-CPU stats, including drops
sysctl net.netfilter.nf_conntrack_count net.netfilter.nf_conntrack_max
```

A full conntrack table drops **new** connections while existing ones keep working
— the signature is "the service is up, but new clients cannot connect", plus
`nf_conntrack: table full, dropping packet` in `dmesg`. Common on busy NAT
gateways and Kubernetes nodes.

`conntrack -S` showing nonzero `insert_failed` is the classic marker of the
DNS race that causes intermittent 5-second lookups in Kubernetes.

## MTU and path MTU discovery

Symptom: small requests work, larger ones hang. The handshake completes (small
packets), then the first full-size packet is dropped.

```bash
ip link show <iface> | grep mtu
ping -M do -s 1472 <host>       # 1472 + 28 bytes header = 1500
```

`-M do` sets Don't Fragment. Decrease the size until it succeeds; the largest
working size plus 28 is your path MTU. If 1472 fails but 1400 works, something
in the path has a smaller MTU and PMTU discovery is broken — usually ICMP
"fragmentation needed" is being filtered.

Overlay networks (VXLAN, WireGuard, IPsec, most CNIs) add encapsulation overhead,
so an interface MTU of 1500 inside a tunnel is wrong. Setting the interface MTU
correctly is the fix; clamping MSS is the workaround.

## TLS

```bash
openssl s_client -connect <host>:443 -servername <host> </dev/null
openssl s_client -connect <host>:443 -servername <host> -showcerts </dev/null
echo | openssl s_client -connect <host>:443 2>/dev/null | openssl x509 -noout -dates -subject -issuer
```

`-servername` sets SNI. Without it a virtual-hosted server returns its default
certificate, which produces a confusing mismatch that does not reproduce in a
browser.

| Error | Cause |
|---|---|
| `unable to get local issuer certificate` | Missing intermediate in the server's chain, or the client lacks the root. Servers must send the full chain minus the root. |
| `certificate has expired` | Check `-dates`. Also check the client's clock. |
| `Hostname mismatch` | The name is not in SAN. CN alone has not been accepted for years. |
| `handshake failure` / `no shared cipher` | Protocol or cipher mismatch — often an old client against a TLS 1.3-only server. |
| `tlsv1 alert unknown ca` | Mutual TLS: the server does not trust your client cert's CA. |

```bash
# Verify a cert matches a key (the hashes must be identical). Works for RSA and EC.
openssl x509 -noout -pubkey -in cert.pem | openssl sha256
openssl pkey -pubout -in key.pem         | openssl sha256
```

## Bandwidth and interface errors

```bash
ip -s link show <iface>      # RX/TX errors, drops, overruns
ethtool -S <iface>           # driver-level counters
ethtool <iface>              # negotiated speed and duplex
```

Nonzero `dropped` on RX usually means the kernel could not keep up (ring buffer
or socket buffer too small), not a cable fault. Actual `errors` point at the
physical layer. A link negotiated at an unexpected speed or half duplex explains
throughput problems that look like application slowness.
