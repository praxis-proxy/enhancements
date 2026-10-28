---
issue: https://github.com/praxis-proxy/praxis/issues/124
discussion: https://github.com/praxis-proxy/praxis/discussions/1411
status: proposed
repos:
  - praxis
authors:
  - rkaplan-hub
graduation_criteria:
  - How? section with requirements and design
  - Body-byte accounting boundary specified, excluding headers, framing, and TLS overhead
  - Per-connection budget is the downstream client connection, shared by its HTTP/2 streams and successive HTTP/1.1 requests
  - A positive burst refills at the byte rate, independently per direction; omitted and zero are a no-burst mode; both modes make progress on a nonempty transfer, including a chunk larger than the burst
  - HTTP and TCP pacing applies bounded backpressure without blocking runtime workers, and names where transfer suspends and resumes
  - A full per-IP table closes a new address, and eviction or recreation preserves the remaining allowance across reconnects, including while the IP has no connection
  - Stream reset, TCP half-close, full disconnect, upstream failure, and shutdown each release unused reservations without refunding bytes already sent
  - Shared locks are not held across a pacing wait or a socket write
stakeholders:
  - shaneutt
  - twghu
origin:
  repo: praxis
  issue: https://github.com/praxis-proxy/praxis/issues/124
  file: 00124_bandwidth-limiting.md
  pr: https://github.com/praxis-proxy/praxis/pull/1412
---

# Bandwidth Limiting Filter

## What?

Add a bandwidth limit that paces upload and download
throughput in bytes per second. Operators can apply it
per connection, or share it across connections from the
same source IP, on both HTTP and TCP listeners.

Praxis already limits how many requests a client may
start. The `rate_limit` filter uses a token bucket in
`global` or `per_ip` mode and rejects excess requests
with `429`. That filter does not pace the bytes of a
request body or a response body, and it does not apply
to TCP listeners. A single allowed request, streamed
model response, file transfer, or TCP connection can
still saturate the link.

This proposal adds:

- Independent upload and download ceilings, in bytes
  per second. The ceiling counts body bytes that pass
  through this filter instance. It is not a cap on
  every byte on the wire. HTTP headers, framing, and
  TLS overhead are excluded. On HTTP, upload is the
  request body and download is the response body. On
  TCP, upload is client-to-upstream and download is
  upstream-to-client, counted as the payload this
  filter forwards. If TLS is terminated before this
  filter, the decrypted payload is what counts.
  Operators choose the counting stage, including
  before or after compression, by where they place
  this filter. Bytes are counted at that position.
  Each replayed upstream upload is charged once per
  attempt. Cached and locally generated response
  bodies are included and paced. A body that never
  enters this filter is outside the ceiling. That is
  a placement limit, not a second accounting rule.
  Omitting a direction leaves that direction unlimited.
- Per-connection scope. The budget is the downstream
  client connection, not an HTTP request and not an
  upstream connection. Every HTTP/2 stream on that
  connection, and every later HTTP/1.1 request on a
  kept-alive connection, shares one upload budget and
  one download budget. Ten streams under a 1 MiB/s
  limit stay at 1 MiB/s together. The budget and its
  burst are created when the connection first reaches
  this filter instance, and they are released when that
  downstream connection closes. They are not reset
  between requests or streams. An upstream connection
  pool must not merge budgets from different clients.
  A request on the same downstream connection that
  reaches a different `bandwidth_limit` instance has
  that instance's own budget.
- Per-source-IP scope. Connections from the same source
  IP that pass through one `bandwidth_limit` instance
  share one upload budget and one download budget. That
  instance owns the map, as `RateLimitFilter::from_config`
  does for `rate_limit`. Another instance, including one
  on a different listener or chain, has its own budgets.
  Nothing in the process holds a shared map.
- The same filter on HTTP and TCP listeners.
- Pacing. Traffic above the limit is delayed so the
  stream stays at the configured rate. The delay
  applies backpressure toward the producer and keeps
  pending body data bounded, for upload and download.
  The transfer still streams; the filter does not
  buffer the whole body. Pacing waits must not block
  runtime workers. Request admission stays with
  `rate_limit`.
- Optional burst. When `burst_bytes` is greater than
  zero, it is the capacity of a token bucket that
  refills at the configured byte rate. It is not a
  one-time allowance. Omitting it, or setting it to
  zero, is a separate no-burst mode, not a bucket of
  capacity zero. In that mode pacing applies from the
  first byte, and a nonempty transfer must still make
  progress. Upload and download each have their own
  burst of that size. In `per_ip` mode the burst
  belongs to the shared budget, so another connection
  from the same IP does not receive a new one. A body
  chunk larger than the burst must still move,
  incrementally, as tokens refill. It must not wait
  until the whole chunk fits in the bucket. The same
  progress rule applies in no-burst mode. A small
  body avoids delay only while that direction's budget
  still has burst available.
- Bounded local state for per-IP budgets, in the same
  spirit as the existing per-IP rate limiter: a cap on
  tracked addresses, idle eviction, and metrics for map
  size and evictions. Evicting or recreating a bucket
  must preserve the remaining per-IP allowance,
  including while that IP has no connection. A depleted
  bucket stays tracked until a normal refill would
  make a new full bucket equivalent. Debt and unused
  reservations stay with that allowance. A reconnect
  before then does not receive a fresh burst. An IP is
  never given a second budget. When the table is full,
  a connection from an untracked IP is closed instead
  of paced.
  That close is the one case that is rejected rather
  than delayed. Waiters and queued bytes for a tracked IP
  stay bounded. The same outcomes apply to HTTP and
  TCP.

### Goals

- Let operators set upload and download byte rates
  without application changes.
- Keep request-count limiting and byte pacing as
  separate controls that can be used together.
- Apply the same limits on HTTP body streams and on
  TCP connections.
- Cancel pacing for the work that actually ended.
  An HTTP/2 stream reset cancels only that stream.
  Sibling streams keep using the shared budget. A TCP
  client that half-closes after its upload still
  receives the download. Full disconnect, upstream
  failure, and shutdown release unused reservations
  and waiter state without refunding bytes already
  sent. A slow reader that stays connected is not a
  disconnect and can still consume a shared per-IP
  budget.
- Keep each per-IP map inside its filter instance and
  bounded in memory. Listener chains do not share it. A
  budget shared across instances or replicas belongs
  with the shared state model in
  [proposal #99](00099_stateful-proxy-state-management.md)
  and is outside this proposal.
- Leave the data path unchanged when the filter is not
  configured.

### Example configuration

Aspirational shape, using the same listener and filter
chain structure as the existing `rate_limit` config.
The HTTP and TCP blocks are separate instances, so
their per-IP budgets stay separate.

```yaml
listeners:
  - name: api
    address: "0.0.0.0:8080"
    filter_chains:
      - http-limited

  - name: db
    address: "0.0.0.0:5432"
    protocol: tcp
    upstream: "10.0.0.5:5432"
    filter_chains:
      - tcp-limited

filter_chains:
  - name: http-limited
    filters:
      - filter: bandwidth_limit
        mode: per_ip                  # per_connection | per_ip
        upload_bytes_per_sec: 1048576 # 1 MiB/s request body
        download_bytes_per_sec: 5242880 # 5 MiB/s response body
        burst_bytes: 262144

      - filter: router
        routes:
          - path_prefix: "/"
            cluster: backend

      - filter: load_balancer
        clusters:
          - name: backend
            endpoints:
              - "127.0.0.1:3000"

  - name: tcp-limited
    filters:
      - filter: bandwidth_limit
        mode: per_connection
        upload_bytes_per_sec: 1048576
        download_bytes_per_sec: 1048576
```

## Why?

### Motivation

Operators use Praxis in front of model servers, file
transfers, and TCP services. Those workloads are often
one long stream rather than many small requests.
`rate_limit` can allow the request and still leave the
proxy unable to cap how fast the body moves.

That shows up in a few places:

1. **One allowed request can saturate the link.** A
   large upload, a file download, or a long streamed
   response (model output, server-sent events) has no
   bytes-per-second ceiling.
2. **Request counting and throughput are different
   controls.** `rate_limit` decides whether a request
   may start. It does not pace bytes once the request
   is in flight.
3. **No shared bandwidth budget per client.** Several
   connections from the same source IP can each run at
   full speed. Opening more connections multiplies that
   client's throughput.
4. **TCP listeners have the same gap.** Bulk TCP
   streams, such as database or tunnel traffic, have no
   upload or download rate. TCP filters today learn byte
   totals when the connection closes, which is too late
   to pace the stream.

Without this control, operators push throttling into
every client or place another proxy in front of Praxis.

Envoy's bandwidth limit filter, NGINX `limit_rate`, and
HAProxy byte-rate limits treat throughput as a normal
proxy control, separate from request-rate limiting.

Per-IP budgets are local runtime state owned by the
filter instance, like the per-IP map `rate_limit`
creates in `from_config`. This proposal does not add a
process-wide store or a new shared storage class.
Cross-instance and cross-replica budgets would follow
[proposal #99](00099_stateful-proxy-state-management.md)
if a later issue requires them.

### User Stories

- As a proxy operator, I want a bytes-per-second cap on
  request and response bodies so that one client cannot
  saturate the proxy or the upstream.
- As an AI gateway operator, I want to pace long streamed
  model responses so that a single generation cannot
  consume the entire downlink.
- As a platform operator, I want connections from the
  same source IP that pass through one `bandwidth_limit`
  instance to share an upload budget and a download
  budget so that opening more connections on that
  listener does not multiply that client's throughput.
- As an operator of a TCP listener, I want the same
  upload and download limits on raw TCP connections so
  that HTTP and TCP entry points follow one policy.
- As an SRE, I want excess traffic delayed rather than
  rejected so that large transfers complete at a steady
  rate, except when the per-IP table is full and the
  address is new.
- As an operator, I want a burst allowance so that small
  API calls are not delayed while that budget still has
  burst available.

### Non-Goals

- Replacing `rate_limit`. Request admission, `429`
  responses, and rate-limit headers stay there.
- A single byte budget shared by every client.
  [Issue #124](https://github.com/praxis-proxy/praxis/issues/124)
  covers per-connection and per-source-IP limits.
- One per-IP budget shared across filter instances,
  listener chains, or proxy processes.
- Fair queuing, priority tiers, or traffic classes beyond
  the two scopes above.
- Changing payload bytes. The filter changes when bytes
  are released, not their contents.
