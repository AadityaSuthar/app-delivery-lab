# Application Delivery Lab — NGINX Load Balancing, Failover & Health Checks

An NGINX reverse proxy load-balancing traffic across three containerized
backend replicas. Compares three balancing algorithms under load, and
demonstrates passive health-check failover when a backend dies.

## Stack

- **Load balancer:** NGINX (open-source) reverse proxy
- **Backends:** 3 × NGINX replicas, each identifying itself in its response
- **Orchestration:** Docker Compose, single private network
- **Load testing:** ApacheBench (ab)

Traffic path: `client -> NGINX load balancer (:8080) -> [backend-1, backend-2, backend-3]`

## Measurement note

Distribution was first tested with a sequential `curl` loop, which showed
traffic pinned to one backend. Cause: sequential curl reuses one TCP
connection, so NGINX serves every request down the same upstream link,
masking round-robin. Switching to ApacheBench — which opens concurrent
connections, as real clients do — revealed true distribution. Lesson: a
load balancer must be measured with concurrent connections, not
sequential single-connection requests.

## Algorithm comparison

Each algorithm is a separate config in `configs/`; results in `results/`.

| Algorithm         | Test            | Distribution          | Behaviour                         |
| ----------------- | --------------- | --------------------- | --------------------------------- |
| Round-robin       | ab -n 30000 -c3 | 10005 / 9997 / 9998   | Even across all backends          |
| Least-connections | ab -n 3000 -c3  | 989 / 1007 / 1004     | Even (identical backends)         |
| IP-hash           | ab -n 3000 -c3  | 3000 / 0 / 0          | One client pinned to one backend  |

- **Round-robin** (default): requests distributed in turn. Converges to
  perfectly even over volume (spread of 8 across 30,000 requests).
- **Least-connections**: routes to the backend with fewest active
  connections. With identical, equally-fast backends it behaves like
  round-robin; the two diverge only under uneven response times, where
  least_conn steers away from a slow, connection-heavy backend.
- **IP-hash**: routes each client IP to a fixed backend (session
  persistence). A single client is not load-balanced — the intended
  trade-off. Simulated clients (via X-Forwarded-For) confirmed different
  clients spread across backends while the same client always returns to
  the same one.

## Failover (passive health checks)

A backend was stopped (`docker stop`) mid-traffic. NGINX detected the
failure on the next proxied request, marked the backend down (`max_fails`
/ `fail_timeout`), and rerouted to the surviving backends within one
detection cycle. Clients continued to be served with no sustained errors.

**Architectural note:** open-source NGINX performs *passive* health
checks — a failure is only detected when a real request to the backend
fails. NGINX Plus (F5's commercial version) adds *active* health checks,
probing backends proactively so a dead node is detected and removed
before a real request ever hits it.

## Files

- `configs/lb-round-robin.conf`, `lb-least-conn.conf`, `lb-ip-hash.conf`
- `results/round-robin.txt`, `least-conn.txt`, `ip-hash.txt`, `failover.txt`
- `docker-compose.yml` — 3 backends + load balancer
- `backend/index.html` — per-replica identity page
