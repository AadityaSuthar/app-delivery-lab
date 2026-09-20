# Benchmark Results — Docker + Kubernetes App Delivery Lab

Environment: single laptop, kind cluster + Docker Compose, nginx backends.
Numbers are laptop-scale; the value is the methodology and measurement.

## Level 2 — Distribution (round-robin, ab -n 3000 -c 10)
992 / 1010 / 998 across 3 backends
Spread: 18 requests (~0.6% from perfectly even)
Even distribution confirmed under concurrent load.

## Level 3 — Throughput & Latency (Docker LB, ab -n 3000 -c 10)
Requests/sec: ~15,900 (mean, consistent across runs)
Failed requests: 0
Latency: p50 0ms, p95 1ms, p99 2ms (mean 0.62ms)
Zero failures under concurrent load; sub-millisecond median latency.

## Level 4 — Failover (Docker LB)

Test A — backend killed DURING load (ab -n 20000 -c 10):
  Complete: 20000, Failed: 0, throughput held ~19,600 req/s
  Zero client-facing failures when a backend terminated under load.

Test B — backend down BEFORE load (ab -n 5000 -c 10):
  Complete: 5000, Failed: 0
  Distribution: 2511 / 2489 / 82 (dead backend got only 82 probe
  attempts before NGINX marked it down via max_fails, then rerouted).
  Traffic automatically rerouted around the failed backend; survivors
  split load evenly; zero failed requests.

## Level 4 — Self-healing (Kubernetes)
Deleted a backend pod from a Deployment (declared replicas: 3).
K8s detected the shortfall and scheduled a replacement pod automatically.
Replacement Running with AGE ~1s — recovery in about 1 second, no manual
intervention. (Docker Compose cannot self-heal; this is the K8s
differentiator: declared state is continuously reconciled.)

## Level 4 — Self-healing (Kubernetes)
Deleted a backend pod (Deployment declared replicas: 3).
K8s scheduled a replacement automatically — Running with AGE ~1s.
Recovery ~1 second, no manual intervention. Docker Compose cannot do
this; declared-state reconciliation is the K8s differentiator.
