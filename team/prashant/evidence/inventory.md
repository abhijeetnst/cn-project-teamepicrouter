# IP and service inventory  (2026-10-05 00:30:33)

| Machine | Role | Interface | IPv4/prefix | Mask | Gateway | MAC | Listening |
|---|---|---|---|---|---|---|---|
| node-1 | Mac 1 - primary DNS (dnsmasq) + test client | ens5 | 172.31.250.11/24 | 255.255.255.0 | 172.31.250.1 | 0e:ff:f4:92:b9:b9 | 53  |
| node-2 | Mac 2 - edge: nginx reverse proxy, TLS, load balancer | ens5 | 172.31.250.12/24 | 255.255.255.0 | 172.31.250.1 | 0e:ff:c3:e2:6b:f1 | tcp/443 tcp/80  |
| node-3 | Mac 3 - Backend A :3001 (+ backup DNS, standby edge) | ens5 | 172.31.250.13/24 | 255.255.255.0 | 172.31.250.1 | 0e:ff:e1:db:31:95 | 53 tcp/3001 tcp/443 tcp/80  |
| node-4 | Mac 4 - Backend B :3002 + test client | ens5 | 172.31.250.14/24 | 255.255.255.0 | 172.31.250.1 | 0e:ff:d2:be:6d:f1 | tcp/3002  |
