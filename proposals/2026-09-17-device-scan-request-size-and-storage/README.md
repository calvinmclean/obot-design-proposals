# 2026-09-17: Device Scan Request and Storage Size

- **Authors:** @calvinmclean
- **Created:** 2026-09-17

## Summary

Make device scan failures visible and allow scans that exceed a single-request
limit to be submitted successfully. Submit one scan through multiple bounded
requests. Publish it to fleet inventory when complete; if the upload fails,
show the received portions in scan history with a failure status.

Fingerprinting raw content could later reduce repeated uploads and backend
storage, but it does not solve the initial large upload. It is therefore an
optional future optimization rather than the primary design.

## Related issues

- [obot-platform/obot#7506: Large device scan submissions fail without reporting scan failure](https://github.com/obot-platform/obot/issues/7506)

## Related ODPs

None

## Problem and motivation

Obot Sentry currently submits one complete device scan in one request. A large
scan can exceed either a reverse proxy's request limit or Obot's request limit.
The failure is visible only on the device, so administrators may continue
seeing stale inventory without knowing that a newer scan failed.

The reported production examples are not explained by project traversal depth.
One approximately 9.9 MB scan attributed about 95% of its payload to Codex
plugin-cache data, dominated by raw file contents and repeated skill and plugin
material. Compression can reduce bytes on the wire, but it does not guarantee
that a scan will fit every request limit.

Once failed attempts are recorded, "latest scan" becomes ambiguous. A failed
scan is the latest attempt but must not replace the last usable inventory.

## Goals

- Make pending and failed attempts visible to administrators when Obot is
  reachable.
- Preserve the latest usable inventory when a newer attempt fails.
- Accept a complete scan even when it cannot fit in one request.
- Keep per-request and whole-scan resource usage bounded.
- Prevent incomplete scans from appearing in inventory.
- Preserve received portions of a failed upload in scan history.
- Allow interrupted uploads to be retried without duplicating scan data.
- Remain compatible with older Sentry clients during rollout.

## Non-goals

- Guaranteeing failure reporting when the device cannot reach Obot.
- Bypassing deployment-specific limits with unbounded requests or scans.
- Reducing repeated backend storage as part of the multi-request design.
- Defining a general-purpose large-object upload protocol.
- Replacing the existing fleet inventory and scan-history experiences.

## Context and constraints

- A scan contains separate collections for clients, MCP servers, skills,
  plugins, and files. Obot already persists these collections separately under
  a scan record.
- A single collection, especially files, may itself exceed a request limit.
  Sending one request per collection is therefore not sufficient; a collection
  must be divisible across requests. Sentry already omits content for files
  over 1 MiB while retaining their path, size, and oversized status. Readable
  text files within that limit can keep their content.
- HTTP chunked transfer and a multipart body are still one request and remain
  subject to aggregate request limits. This design requires multiple HTTP
  requests.
- Obot does not control the reverse proxy configuration in every deployment,
  so increasing Obot's request limit alone cannot resolve all failures.
- Obot currently limits both the compressed request body and its decoded
  content to 8 MiB per request. A proxy can impose a lower limit on the
  compressed body before the request reaches Obot.
- Every request must be authorized for the same device and Obot installation.
- The current single-request limit caps each submission. A multi-request scan
  needs its own aggregate limit; the reported 9.9 MB scan is the only detailed
  size example available so far.
- Existing clients submit complete scans through the current endpoint and must
  continue to work during migration.
- Skill and plugin detail views display captured file content. This proposal
  preserves that content in complete scans; changing collection policy is a
  separate product decision.
- Existing scan history defaults to 90 days of retention; setting retention to
  zero disables automatic cleanup. Failed attempts and their received portions
  use the same policy.

## Proposed design

### Scan states

A scan attempt has one of three states:

- **Pending:** Obot has accepted the start of a scan but not the complete scan.
- **Successful:** Obot has accepted the complete scan.
- **Failed:** The scan could not be completed. Any portions Obot received remain
  available in scan history with the failure reason.

Only successful scans are usable inventory. Fleet views use the latest
successful scan for each device. Device and history views also show the latest
attempt and any received portions, so a newer failed attempt is visible without
replacing older usable inventory.

### Multi-request scan submission

Add a scan-specific upload interface with three operations:

1. **Start** creates a pending scan and returns its scan ID.
2. **Append** submits a bounded part of one collection. Parts are independently
   retryable and idempotent, and collections may use as many parts as needed.
3. **Finalize** declares the expected parts for each collection. Obot checks
   that every declared part was received, including an explicit empty result
   for collections with no observations, before changing the scan from pending
   to successful. Until then, none of its observations appear in fleet
   inventory.

The interface exposes logical scan collections rather than database tables.
Obot remains responsible for how parts are persisted and for enforcing both
per-request and aggregate scan limits. The whole-scan cap defaults to 64 MiB of
decoded data and is configurable by the operator. A scan that exceeds its
configured cap fails with its received portions visible in history.

Sentry groups observations into parts targeting at most 512 KiB before
independently compressing each request. A captured file stays intact: if it
does not fit alongside other observations, Sentry sends it in its own part,
which may exceed that target. Existing per-file capture limits remain
unchanged. Compression may bring a near-1 MiB file below Nginx's default 1 MiB
compressed-body limit, but acceptance by that or another proxy is not
guaranteed. Obot enforces per-request limits on both compressed and decoded
bodies. A retry of an acknowledged part must have the same content; a changed
part with the same identity is rejected.

Sentry may resume a pending scan after a transient failure by resending only
unacknowledged parts. Obot allows one pending scan per device; starting a new
scan for that device marks the earlier pending scan as failed with a superseded
reason. Distinct scan IDs prevent parts from one attempt changing another.

On HTTP 413, Sentry does not resize or retry the rejected part; it reports a
request-too-large failure for the attempt through a small, best-effort status
request. Sentry likewise reports other non-recoverable errors when it abandons
a pending scan. If the failure report cannot reach Obot, Obot changes the
pending scan to failed after 24 hours without a successful append, recording
an upload timeout as the reason. A failed scan ID cannot later be finalized.

Received portions remain visible with the failed attempt until normal
scan-history retention deletes both. At the proposed 64 MiB cap and default
90-day retention, one near-limit failure per day could retain about 5.6 GiB of
partial data per device. This is a capacity bound, not a production estimate;
operators can change or disable scan-history cleanup.

If submission fails before Obot creates a pending scan, Sentry makes a
best-effort, small failure report when the server is reachable. Failed attempts
never replace the last successful inventory. Failure reports use a bounded
reason category and safe message, without raw proxy responses or local file
paths.

### Observability

Operators can distinguish pending, successful, and failed attempts, including
failures caused by an upload timeout. Operational measurements include request
and aggregate scan sizes, part retries, completion time, and timeout counts
without logging raw scan content.

Whether aggregate product telemetry should report failed scans remains a
separate decision subject to the product-telemetry consent model.

## Alternatives considered

### Fingerprinted raw content

Store raw content once under a fingerprint and let later scans refer to it.
This can reduce repeated transfer and backend storage, but it cannot guarantee
that previously unseen content will fit in the first request. It also requires
reference-aware retention, protection against cross-tenant existence leaks,
and recovery when client and server knowledge disagree.

Fingerprinting may be added later, narrowly for raw content, if measurements
show that storage duplication is significant over the actual scan-retention
window. It can use the multi-request interface when the first set of artifacts
is large.

### Compression and larger limits only

Keep each scan in one request, compress it, and raise the accepted limit. This
is a smaller protocol change but remains dependent on every proxy in the path,
increases resource exposure, and provides no reliable upper bound on future
scan growth.

### gRPC streaming

Send a scan through one client-streaming gRPC call. This offers flow control
and avoids repeated request setup, but the stream is still one HTTP/2 request
subject to proxy body limits. Protobuf would reduce some metadata overhead,
not the captured file content that dominates large scans. Supporting gRPC
through deployment proxies and resuming interrupted streams would add
complexity without resolving the reported size rejection.

### WebSocket upload

After a WebSocket upgrade, scan data is tunneled rather than sent as an HTTP
request body. This could avoid a proxy's body limit, including for an intact
near-1 MiB file. It would still need bounded messages, acknowledgments, and
resume after disconnection, while depending on WebSocket support and long-lived
connections across deployments. For the reported scan size, this added
transport and operational complexity is not justified over a few dozen HTTP
requests that can reuse a connection. Reconsider it if single-file parts
commonly fail at deployment proxy limits.

### Retry without bulky content

After a size rejection, retry a smaller scan that omits optional raw content.
This could provide quicker relief, but adds a second, incomplete scan format
that would remain after multi-request submission is available. It also does not
meet the goal of accepting a complete scan.

### Never collect raw content

Remove raw file content from every scan. This produces the smallest and least
sensitive representation, but may remove diagnostic or product value that has
not yet been assessed.

## Trade-offs

- Multi-request submission accepts complete first-time scans and is independent
  of one-request limits, at the cost of pending state, finalization, and cleanup.
- Aligning parts with logical scan collections keeps the interface small while
  allowing any large collection to be divided further.
- Keeping files intact avoids file-chunk reassembly but leaves a near-limit
  captured file vulnerable to a proxy's lower per-request limit.
- Atomic finalization prevents partial fleet inventory; failed uploads still
  show their received portions in scan history.
- Idempotent parts make retries safe but require stable scan and part identity.
- One pending scan per device bounds unfinished storage but means a newer attempt
  ends an older unfinished upload.
- This design improves submission reliability but does not reduce repeated
  backend storage; fingerprinting remains a separate possible optimization.
- Separating latest attempt from latest successful scan makes failures visible,
  but consumers must choose which meaning of "latest" they need.

## Risks and open questions

- **Capacity validation:** Validate the proposed 64 MiB whole-scan default,
  24-hour inactivity timeout, and storage cost of retaining failed portions
  for the configured scan-history period against representative scans and
  deployment budgets before release.
- **Production distribution:** One reported 9.9 MB scan was dominated by file
  content from the Codex plugin cache. Measure broader scan sizes and proxy
  limits before treating it as representative.

## Rollout and migration

1. Add attempt status, failure reporting, administrator visibility, and the
   multi-request interface while retaining the existing complete-scan endpoint.
   Older clients continue submitting complete scans as successful attempts.
2. Enable multi-request submission in newer Sentry clients. When connected to
   an older Obot server, they use the existing endpoint; large scans can still
   fail until that server is upgraded.
3. Monitor completion, retries, expiration, and aggregate scan size. Evaluate
   fingerprinted raw content separately using measured backend storage
   duplication and the configured scan-retention window.

Rollback disables the new client path and leaves the existing endpoint
available. Pending scans created before rollback become failed after their
timeout and remain visible in scan history without affecting fleet inventory.

## Testing and validation

- Reproduce proxy-originated and Obot-originated size rejection with realistic
  plugin-cache payloads.
- Verify a complete large scan can be submitted through individually bounded
  requests.
- Verify duplicate, reordered, interrupted, and resumed part submission.
- Verify a changed retry cannot replace an acknowledged part, and a captured
  file too large for the target part size is sent intact in its own part.
- Verify a 413 fails the attempt with a safe request-too-large reason, retains
  acknowledged portions in history, and does not trigger repartitioning.
- Verify missing or invalid parts prevent finalization and never affect fleet
  inventory.
- Verify per-request and aggregate limits, including a non-default operator
  cap and a large files collection split across several requests.
- Verify starting a new scan for a device fails its earlier pending attempt and
  that parts cannot cross scan IDs.
- Verify abandoned scans transition from pending to failed after the timeout,
  retain received portions in history, and do not replace the latest successful
  inventory.
- Verify failed portions and their failure status are removed together when
  configured scan-history retention expires, and both remain when cleanup is
  disabled.
- Verify failure reports are bounded and sanitized, including failures before
  a pending scan is created.
- Verify old and new Sentry clients remain compatible during rollout and
  rollback.

## References

- [Issue #7506 and production payload breakdown](https://github.com/obot-platform/obot/issues/7506)
- [Nginx `client_max_body_size` documentation](https://nginx.org/en/docs/http/ngx_http_core_module.html#client_max_body_size)
- [gRPC over HTTP/2](https://grpc.io/blog/grpc-on-http2/)
- [Nginx WebSocket proxying](https://nginx.org/en/docs/http/websocket.html)
- Existing Obot scan model: `pkg/gateway/types/devicescan.go`
- Existing Obot scan persistence: `pkg/gateway/client/devicescan.go`
- Existing Obot scan ingest: `pkg/api/handlers/devicescans.go`
- Existing Sentry scan submission: `pkg/client/devicescan.go`
