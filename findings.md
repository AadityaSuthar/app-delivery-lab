# Application Delivery Lab — Findings

## Setup
NGINX load balancer -> 3 NGINX backend replicas (backend-1/2/3), Docker network.
Measured with ApacheBench (ab), concurrent connections.

## Measurement note
Sequential curl reuses one TCP connection, masking round-robin (traffic
appears pinned to one backend). ab opens concurrent connections and shows
true distribution.

## Round-robin (default)
ab -n 30 -c 3:
  backend-1: 9
  backend-2: 10
  backend-3: 11
Even distribution across all three. CONFIRMED.
