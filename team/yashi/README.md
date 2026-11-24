# Yashi Agrawal — Mac 2 / node-2

**Enrollment:** 2401020078 · **Machine:** Mac 2 → `node-2` · **Address:** `172.31.250.12` (or `10.0.1.12`)

## Role

Edge owner. node-2 is the only machine the client talks to over HTTPS. It terminates TLS with the team's private CA, redirects plain HTTP to HTTPS, and balances requests round-robin across Backend A and Backend B.

- nginx reverse proxy and TLS termination
- HTTP/2 over ALPN, TLS 1.2 + 1.3 only
- Round-robin upstream with passive health checks and retry on the other backend
- Response caching headers (`Cache-Control`, `ETag`, `304 Not Modified`)
- Access log fields that prove which backend answered: `backend=$upstream_http_x_backend`

## What runs on node-2

| Service | Port | Config |
|---|---|---|
| nginx edge | tcp/80 → 301, tcp/443 | `/etc/nginx/conf.d/cn-edge.conf` |
| private CA + server cert | — | `/etc/cn-pki/ca.crt`, `tls.crt`, `tls.key` |

Request path:

```
client ──DNS──▶ node-1
client ──HTTPS 443──▶ node-2 (TLS ends here)
                        ├──HTTP 3001──▶ node-3 Backend A
                        └──HTTP 3002──▶ node-4 Backend B
```

The certificate is issued for `app.teamepicrouter.test` and `api.teamepicrouter.test` by `teamepicrouter Private CA`. The CA private key stays on the edge and is never committed.

## Proof it works

```bash
# rendered from config/nginx-edge.conf by make converge
sudo nginx -t
curl -v https://app.teamepicrouter.test          # verify ok, no -k, HTTP/2 200
curl -sI http://app.teamepicrouter.test          # 301 to https

# round-robin: six requests, alternating X-Backend
for i in {1..6}; do curl -si https://app.teamepicrouter.test/api/status | grep -i '^x-backend'; done

# caching
E=$(curl -sI https://app.teamepicrouter.test/api/cached | awk 'tolower($1)=="etag:"{print $2}' | tr -d '\r')
curl -sI -H "If-None-Match: $E" https://app.teamepicrouter.test/api/cached   # 304
```

Evidence: [`evidence/tls-handshake.txt`](evidence/tls-handshake.txt), [`evidence/load-balancing.txt`](evidence/load-balancing.txt), [`evidence/caching.txt`](evidence/caching.txt), [`evidence/nginx-config.txt`](evidence/nginx-config.txt), [`evidence/node-2-topology.txt`](evidence/node-2-topology.txt).

## Viva points

- **Why TLS ends at the edge:** one certificate and one place to enforce protocol versions; backends stay simple and unprivileged.
- **Why `proxy_next_upstream`:** if the chosen backend refuses the connection, nginx retries the other one, so one dead backend does not become a failed request.
- **Why the ETag body must be identical on A and B:** a conditional request may land on the other backend; a backend-specific ETag could never return 304.
- **Why HTTP/2:** one TCP connection multiplexes requests; visible as `ALPN: server accepted h2`.
- **Known single point of failure:** node-2. The Phase 2 standby edge on node-3 is the mitigation.

## Files in this part

```
README.md             this document
nginx-edge.conf       rendered edge configuration (canonical template: config/nginx-edge.conf)
pki.sh                idempotent private CA + certificate issuance (runs on node-2)
evidence/             TLS handshake, load balancing, caching, nginx config and topology
```
