# Local Rate Limiting for httpbin

> Kubernetes manifests: [template/httpbin-local-rate-limit.yaml](template/httpbin-local-rate-limit.yaml)

## Overview

Local rate limiting is applied to the **httpbin** service deployed in the `httpbin` namespace. It is enforced at the sidecar (Envoy proxy) level via an `EnvoyFilter` resource (`filter-local-ratelimit-svc`), targeting pods with the label `app: httpbin`. Rate limiting is applied on **inbound** traffic to the httpbin sidecar (`SIDECAR_INBOUND` context).

---

## Request Flow Diagram

```
                          istio-system namespace
 ┌──────────────────────────────────────────────────────────────┐
 │                                                              │
 │   Client ──► Istio Ingress Gateway Pod                       │
 │              (Gateway: ingress-gateway, port 8080)           │
 │              (HTTPRoute: host=httpbin.suman.com → httpbin:80)│
 │                                                              │
 └──────────────────────────┬───────────────────────────────────┘
                            │ forwards to httpbin Service (ClusterIP)
                            ▼
                          httpbin namespace
 ┌──────────────────────────────────────────────────────────────┐
 │                                                              │
 │   httpbin Pod                                                │
 │  ┌────────────────────────────────────────────────────────┐  │
 │  │  Envoy Sidecar  (SIDECAR_INBOUND)                      │  │
 │  │                                                        │  │
 │  │  ┌─────────────────────────────────────────────────┐   │  │
 │  │  │  envoy.filters.http.local_ratelimit             │   │  │
 │  │  │                                                 │   │  │
 │  │  │   Token Bucket                                  │   │  │
 │  │  │   ┌─────────────────────────────────────────┐   │   │  │
 │  │  │   │  max_tokens:      1                     │   │   │  │
 │  │  │   │  tokens_per_fill: 1                     │   │   │  │
 │  │  │   │  fill_interval:   60s                   │   │   │  │
 │  │  │   └─────────────────────────────────────────┘   │   │  │
 │  │  │                                                 │   │  │
 │  │  │   Request arrives                               │   │  │
 │  │  │        │                                        │   │  │
 │  │  │        ▼                                        │   │  │
 │  │  │   Token available?                              │   │  │
 │  │  │        │                                        │   │  │
 │  │  │   YES  │  NO                                    │   │  │
 │  │  │    ┌───┘   └───────────────────────────────┐    │   │  │
 │  │  │    │                                       │    │   │  │
 │  │  └────┼───────────────────────────────────────┼────┘   │  │
 │  │       │                                       │        │  │
 │  │       ▼                                       ▼        │  │
 │  │  Consume token                     Reject request      │  │
 │  │  forward to app                    HTTP 429            │  │
 │  │       │                            header:             │  │
 │  │       │                            x-local-rate-limit: │  │
 │  │       │                            true                │  │
 │  │       │                                       │        │  │
 │  └───────┼───────────────────────────────────────┼────────┘  │
 │          │                                       │           │
 │          ▼                                       ▼           │
 │   httpbin container                        Client gets       │
 │   processes request                        429 response      │
 │   returns 2xx/4xx/5xx                                        │
 │                                                              │
 └──────────────────────────────────────────────────────────────┘
```

**Key points:**
- The token bucket lives **inside each pod's sidecar** — not in the gateway, not in a shared service.
- The gateway simply routes by hostname/path; it has no knowledge of rate limits.
- A pod with an empty bucket rejects the request **before it ever reaches the httpbin container**.
- After a rejection, the bucket refills 1 token after 60 seconds, allowing the next request through.

---

## Token Bucket Algorithm

The rate limiter uses the **token bucket** algorithm, configured as follows in the `EnvoyFilter`:

```yaml
token_bucket:
  max_tokens: 1
  tokens_per_fill: 1
  fill_interval: 60s
```

### How it works

- The bucket holds a maximum of **1 token** at any time.
- **1 token is added** every **60 seconds**.
- Each incoming request consumes 1 token. If the bucket is empty, the request is rate-limited (HTTP 429) and the response header `x-local-rate-limit: true` is added.
- The filter is both **enabled** and **enforced** at 100% of traffic (no dry-run mode).

This translates to a limit of **1 request per minute** per pod instance.

---

## Handling Bursts of Traffic

The token bucket algorithm provides limited burst tolerance through its `max_tokens` setting. With `max_tokens: 1`, there is effectively **no burst allowance** — the bucket can hold at most 1 token, so a burst of requests will immediately exhaust it after the first request.

If burst tolerance were needed, increasing `max_tokens` (e.g., `max_tokens: 10`) would allow up to 10 requests to be served instantly before the limit kicks in, while still refilling at the configured rate. This is useful for absorbing sudden spikes without rejecting legitimate traffic:

| Scenario | `max_tokens: 1` | `max_tokens: 10` |
|---|---|---|
| Single request arrives | Served, bucket empty | Served, 9 tokens remain |
| 5 requests arrive at once | 1 served, 4 rejected | All 5 served, 5 tokens remain |
| Refill after 60s | 1 token available | 1 token added (up to max 10) |

In the current httpbin configuration, the tight limit (`max_tokens: 1`) is intentional — it demonstrates and tests rate limiting behavior rather than serving production traffic.

---

## Token Capacity When Pods Scale

Because the token bucket lives inside each pod's Envoy sidecar, **scaling adds independent buckets — one per replica**. There is no shared state between pods.

```
replicas: 1                        replicas: 3

 ┌─────────────────┐                ┌─────────────────┐
 │  httpbin Pod 1  │                │  httpbin Pod 1  │
 │  ┌───────────┐  │                │  ┌───────────┐  │
 │  │  bucket   │  │                │  │  bucket   │  │
 │  │ tokens: 1 │  │                │  │ tokens: 1 │  │
 │  └───────────┘  │                │  └───────────┘  │
 └─────────────────┘                └─────────────────┘
                                    ┌─────────────────┐
  Effective limit:                  │  httpbin Pod 2  │
  1 req / 60s                       │  ┌───────────┐  │
                                    │  │  bucket   │  │
                                    │  │ tokens: 1 │  │
                                    │  └───────────┘  │
                                    └─────────────────┘
                                    ┌─────────────────┐
                                    │  httpbin Pod 3  │
                                    │  ┌───────────┐  │
                                    │  │  bucket   │  │
                                    │  │ tokens: 1 │  │
                                    │  └───────────┘  │
                                    └─────────────────┘

                                     Effective limit:
                                     3 req / 60s (1 per pod)
```

**What this means in practice:**

- Each new pod starts with a **fresh, full bucket** (`max_tokens: 1` → 1 token ready immediately).
- Scaling from 1 → N pods multiplies the cluster-wide throughput by N, because load balancing spreads requests across pods and each pod allows its own quota.
- Scaling down removes buckets — the remaining pods continue enforcing their own limits independently.
- There is **no redistribution** of tokens between pods. Pod 1's empty bucket does not borrow from Pod 2's full bucket.

**Effective cluster-wide limit formula:**

```
total requests per interval = replicas × tokens_per_fill
                            = N        × 1
                            = N req / 60s
```

| Replicas | Requests allowed per 60s (cluster-wide) |
|---|---|
| 1 | 1 |
| 2 | 2 |
| 5 | 5 |
| 10 | 10 |

This is a side-effect of local rate limiting, not a feature. If you need the **total** to stay at 1 req/60s regardless of replica count, a linked (global) rate limit backed by a shared counter is required instead.

---

## Comparison: Local Rate Limit vs. Linked (Global) Rate Limit

Both approaches serve the same fundamental purpose — **protecting services from excessive traffic** — but differ significantly in scope and consistency.

| Aspect | Local Rate Limit (this config) | Linked / Global Rate Limit |
|---|---|---|
| **Where enforced** | Each Envoy sidecar independently | Centralized rate limit service (e.g., Envoy's `ratelimit` gRPC service) |
| **Limit scope** | Per-pod — each replica has its own bucket | Per-service / per-cluster — shared across all replicas |
| **Consistency** | No coordination between pods; a client can exceed the intended limit by hitting different replicas | Consistent across all pods; the limit holds regardless of which pod handles the request |
| **Latency overhead** | None — decision is made locally in the sidecar | Small overhead — requires a network call to the external rate limit service |
| **Failure mode** | Never fails due to an external dependency | Rate limiting may degrade if the central service is unavailable |
| **Configuration** | `EnvoyFilter` with `envoy.filters.http.local_ratelimit` | `EnvoyFilter` with `envoy.filters.http.ratelimit` + `RateLimitConfig` CRD or Envoy `rate_limit_service` |
| **Best for** | Per-instance protection, simple throttling, no external deps | Accurate service-wide quotas, per-user/per-IP limits at scale |

### When to choose local rate limiting

- You want a **lightweight, zero-dependency** guard against runaway traffic on individual pods.
- Approximate limiting is acceptable (a client hitting 3 replicas effectively gets 3x the per-pod limit).
- You need **fast enforcement** with no network round-trips.

### When to choose linked (global) rate limiting

- You need **exact, cluster-wide quotas** (e.g., 100 req/min total, not 100 per pod).
- You are rate limiting based on dynamic keys like user ID, API key, or IP address.
- You have an existing rate limit service (e.g., `envoyproxy/ratelimit`) already deployed.
