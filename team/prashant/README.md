# Prashant Pandey — Mac 4 / node-4

**Enrollment:** 2401010339 · **Machine:** Mac 4 → `node-4` · **Address:** `172.31.250.14` (or `10.0.1.14`)

## Role

Backend B owner and the team's test client. Every end-to-end check is run from node-4: DNS lookup, TLS handshake, load balancing, caching and packet capture. Also owns the evidence index and final report.

- Backend B: Python standard-library HTTP service on `:3002`, returns `X-Backend: B`
- Test client: dnsmasq client mode, `dig`, `curl`, `cn-*` helpers
- Packet capture: DNS query/response, TCP handshake, TLS handshake, encrypted application data
- Evidence: `evidence/form/FORM.md`, `evidence/inventory.md`, `evidence/results.tsv`, report
- Failure demonstrations: wrong DNS server, wrong record, one/both backends down, wrong port

## What runs on node-4

| Service | Port | Config |
|---|---|---|
| Backend B | tcp/3002 | `/opt/cn/backend.py`, unit `backend-b.service` |
| test client | — | `/usr/local/bin/cn-*`, `/etc/resolv.conf` pointing at node-1 (Phase 1) |

Backend endpoints:

| Path | Behaviour |
|---|---|
| `/` | service info, `Cache-Control: no-store` |
| `/api/status` | `{"backend": "B", ...}`, `Cache-Control: public, max-age=60`, `ETag` |
| `/api/cached` | identical body on A and B, supports `If-None-Match` → 304 |

## Proof it works

```bash
# on node-4
curl -i https://app.teamepicrouter.test/api/status   # 200, X-Backend: A or B
dig app.teamepicrouter.test                          # SERVER: 172.31.250.11#53
cn-pingall

# packet capture: DNS + TCP + TLS in one file
sudo cn-capture            # writes evidence/form/client-dns-tcp-tls.pcap

# inspect the capture
tshark -r evidence/form/client-dns-tcp-tls.pcap -Y dns
tshark -r evidence/form/client-dns-tcp-tls.pcap -Y 'tcp.flags.syn==1 || tls.handshake.type==1'
```

Evidence: [`evidence/packet-capture.txt`](evidence/packet-capture.txt), [`evidence/caching-headers.txt`](evidence/caching-headers.txt), [`evidence/node-4-topology.txt`](evidence/node-4-topology.txt), [`evidence/inventory.md`](evidence/inventory.md), [`evidence/client-dns-tcp-tls.pcap`](evidence/client-dns-tcp-tls.pcap).

## What the capture must show

1. **DNS** — query `A app.teamepicrouter.test` from node-4 to node-1 on UDP/53; response with `A 172.31.250.12`, TTL 30.
2. **TCP** — SYN, SYN/ACK, ACK between node-4 and the edge on tcp/443.
3. **TLS** — ClientHello with SNI `app.teamepicrouter.test`, ServerHello, Certificate for the domain, then encrypted application data (HTTP/2).

If the capture shows plaintext HTTP, the client used port 80 and got a 301; capture 443 instead.

## Viva points

- **Why the domain, never an IP, in the URL:** an IP URL fails certificate name validation and skips DNS entirely.
- **What the pcap proves:** each layer is visible in order — DNS (UDP/53), TCP handshake (L4), TLS handshake (L5/L6), HTTP (L7).
- **Why `dig @8.8.8.8` returns NXDOMAIN:** public DNS has no route to a private `.test` zone; this proves the name is served by our own resolver.
- **Failure isolation:** wrong DNS server → resolution fails; wrong record → connection refused/timeout; backend down → 502 only if both are down, otherwise retry succeeds; wrong port → connection refused.
- **Why `Cache-Control: no-store` on `/`:** service identity must never be cached, unlike cacheable content.

## Files in this part

```
README.md               this document
backend.py              Backend source (runs as B on :3002 here)
backend-b.service       rendered systemd unit for Backend B
cn-capture              packet capture helper
cn-resolver             client resolver helper
evidence/               packet capture, caching headers, topology, inventory
```
