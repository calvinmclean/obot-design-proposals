# 2026-09-17: Device Scan Request and Storage Size

- **Authors:** @calvinmclean
- **Created:** 2026-09-17

## Summary

Make device scan failures visible and allow scans that exceed a single-request
limit to be submitted successfully.

First, retry an oversized scan with bulky content omitted and record the
failure when Obot still cannot accept it. Next, allow one scan to be submitted
through multiple bounded requests and make it available only after the whole
scan is complete.

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

Once truncated and failed scans are recorded, "latest scan" becomes ambiguous.
A truncated scan is the latest usable inventory but must be identified as
incomplete. A failed scan is the latest attempt but must not replace the last
usable inventory.

## Goals

- Make truncated and failed attempts visible to administrators when Obot is
  reachable.
- Preserve the latest usable inventory when a newer attempt fails.
- Accept a complete scan even when it cannot fit in one request.
- Keep per-request and whole-scan resource usage bounded.
- Prevent incomplete scans from appearing in inventory.
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
  must be divisible across bounded requests.
- HTTP chunked transfer and a multipart body are still one request and remain
  subject to aggregate request limits. This design requires multiple HTTP
  requests.
- Every request must be authorized for the same device and Obot installation.
- A sequence of individually valid requests must not bypass the maximum allowed
  size for one completed scan.
- Existing clients submit complete scans through the current endpoint and must
  continue to work during migration.

## Proposed design

### Scan states

A scan attempt has one of three states:

- **Pending:** Obot has accepted the start of a scan but not the complete scan.
- **Successful:** Obot has accepted the complete scan. A successful scan may be
  marked truncated when optional content was omitted.
- **Failed:** Obot knows that the scan could not be completed.

Only successful scans are usable inventory. Fleet views use the latest
successful scan for each device. Device and history views also show the latest
attempt so a newer pending or failed attempt is not hidden by older inventory.

### Immediate failure handling

Continue accepting complete scans through the existing endpoint. If a scan is
rejected as too large, Sentry makes one retry without bulky optional content
and marks an accepted result as truncated.

If Obot still cannot accept the scan, Sentry makes one best-effort request with
the device identity, scan timing, and a sanitized failure reason. Other
submission failures may use the same failure report when the server is
reachable. A failure report does not replace the last successful inventory and
does not recursively retry itself.

### Multi-request scan submission

Add a scan-specific upload interface with three operations:

1. **Start** creates a pending scan and returns its scan ID. The request
   describes the collections that make up the scan so Obot can later determine
   whether it is complete.
2. **Append** submits a bounded part of one collection. Parts are independently
   retryable and idempotent, and collections may use as many parts as needed.
3. **Finalize** validates the complete scan and changes it from pending to
   successful. Until this succeeds, none of its observations appear in fleet
   inventory.

The interface exposes logical scan collections rather than database tables.
Obot remains responsible for how parts are persisted and for enforcing both
per-request and aggregate scan limits.

Sentry may resume a pending scan after a transient failure by resending only
unacknowledged parts. Concurrent scans from one device use different scan IDs
and cannot append to or finalize each other.

Sentry reports a failed attempt when it abandons a pending scan after a
non-recoverable error. Obot expires abandoned pending scans after a bounded
period, records them as failed attempts, and removes their partial data. An
expired or failed scan ID cannot later be finalized.

### Observability

Operators can distinguish pending, successful, truncated, failed, and expired
attempts. Operational measurements include request and aggregate scan sizes,
part retries, completion time, and expiration counts without logging raw scan
content.

Whether aggregate product telemetry should report failed or truncated scans
remains a separate decision subject to the product-telemetry consent model.

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

### Truncated retry only

Use only the immediate fallback. This restores useful inventory and failure
visibility with limited change, but it intentionally loses raw content and
does not allow a complete first scan when that content is required.

### Never collect raw content

Remove raw file content from every scan. This produces the smallest and least
sensitive representation, but may remove diagnostic or product value that has
not yet been assessed.

## Trade-offs

- Multi-request submission accepts complete first-time scans and is independent
  of one-request limits, at the cost of pending state, finalization, and cleanup.
- Aligning parts with logical scan collections keeps the interface small while
  allowing any large collection to be divided further.
- Atomic finalization prevents partial inventory but delays visibility until
  the last part is accepted.
- Idempotent parts make retries safe but require stable scan and part identity.
- This design improves submission reliability but does not reduce repeated
  backend storage; fingerprinting remains a separate possible optimization.
- Separating latest attempt from latest successful scan makes failures visible,
  but consumers must choose which meaning of "latest" they need.

## Risks and open questions

- **Completeness contract:** Decide what the start request must declare so
  finalization can prove that every expected part was received.
- **Part sizing:** Decide whether Sentry chooses part sizes proactively or adapts
  after a request is rejected by a deployment-specific proxy.
- **Pending lifetime:** Choose an expiration period long enough for resume but
  short enough to bound partial storage.
- **Failure status:** Decide how long pending and failed attempts remain visible
  relative to successful scan retention.
- **Raw-content requirement:** Determine whether raw file content is required
  product data, optional diagnostic data, or data Obot should not retain.
- **Production distribution:** Measure which collections dominate affected
  scans and the limits at each rejecting layer.
- **Failure privacy:** Define sanitization so attempt errors help administrators
  without persisting local paths, content, credentials, or proxy response
  bodies.

## Rollout and migration

1. Add attempt status, truncated retry, failure reporting, and administrator
   visibility while retaining the existing complete-scan endpoint.
2. Add multi-request submission alongside the existing endpoint. Older clients
   continue submitting complete scans as successful attempts.
3. Enable multi-request submission in newer Sentry clients and fall back to the
   existing endpoint when connected to an older Obot server.
4. Monitor completion, retry, expiration, and aggregate size before making the
   multi-request interface the preferred submission path.
5. Evaluate fingerprinted raw content separately using measured backend storage
   duplication and the configured scan-retention window.

Rollback disables the new client path and leaves the existing endpoint
available. Pending scans created before rollback expire without affecting
inventory.

## Testing and validation

- Reproduce proxy-originated and Obot-originated size rejection with realistic
  plugin-cache payloads.
- Verify a complete large scan can be submitted through individually bounded
  requests.
- Verify duplicate, reordered, interrupted, and resumed part submission.
- Verify missing or invalid parts prevent finalization and never affect fleet
  inventory.
- Verify per-request and aggregate limits, including a large files collection
  split across several requests.
- Verify concurrent scans from one device cannot modify each other.
- Verify abandoned scans expire, remove partial data, and appear as failed
  attempts without replacing the latest successful inventory.
- Verify a truncated retry is marked incomplete and a failure report is bounded
  and sanitized.
- Verify old and new Sentry clients remain compatible during rollout and
  rollback.

## References

- [Issue #7506 and production payload breakdown](https://github.com/obot-platform/obot/issues/7506)
- Existing Obot scan model: `pkg/gateway/types/devicescan.go`
- Existing Obot scan persistence: `pkg/gateway/client/devicescan.go`
- Existing Obot scan ingest: `pkg/api/handlers/devicescans.go`
- Existing Sentry scan submission: `pkg/client/devicescan.go`
