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
  must be divisible across bounded requests.
- HTTP chunked transfer and a multipart body are still one request and remain
  subject to aggregate request limits. This design requires multiple HTTP
  requests.
- Obot does not control the reverse proxy configuration in every deployment,
  so increasing Obot's request limit alone cannot resolve all failures.
- Every request must be authorized for the same device and Obot installation.
- The current single-request limit caps each submission. The multi-request
  interface needs a separate whole-scan limit; its value is not yet decided.
- Existing clients submit complete scans through the current endpoint and must
  continue to work during migration.

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
non-recoverable error. Obot changes pending scans to failed after a period
without progress, recording an upload timeout as the reason. Received portions
remain visible in scan history until the failed scan reaches its retention
limit. A failed scan ID cannot later be finalized.

If submission fails before Obot creates a pending scan, Sentry makes a
best-effort, small failure report when the server is reachable. Failed attempts
never replace the last successful inventory.

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
- Atomic finalization prevents partial fleet inventory; failed uploads still
  show their received portions in scan history.
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
- **Whole-scan limit:** Set a maximum aggregate scan size before implementation;
  no separate limit exists for a multi-request scan today.
- **Pending lifetime:** Choose an expiration period long enough for resume but
  short enough to bound pending storage. Expiration changes pending to failed.
- **Failure retention:** Decide how long failed attempts and their received
  portions remain visible relative to successful scan retention.
- **Raw-content requirement:** Determine whether raw file content is required
  product data, optional diagnostic data, or data Obot should not retain.
- **Production distribution:** Measure which collections dominate affected
  scans and the limits at each rejecting layer.
- **Failure privacy:** Define sanitization so attempt errors help administrators
  without persisting local paths, content, credentials, or proxy response
  bodies.

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
- Verify missing or invalid parts prevent finalization and never affect fleet
  inventory.
- Verify per-request and aggregate limits, including a large files collection
  split across several requests.
- Verify concurrent scans from one device cannot modify each other.
- Verify abandoned scans transition from pending to failed after the timeout,
  retain received portions in history, and do not replace the latest successful
  inventory.
- Verify failed portions are removed when their scan history retention expires.
- Verify failure reports are bounded and sanitized, including failures before
  a pending scan is created.
- Verify old and new Sentry clients remain compatible during rollout and
  rollback.

## References

- [Issue #7506 and production payload breakdown](https://github.com/obot-platform/obot/issues/7506)
- Existing Obot scan model: `pkg/gateway/types/devicescan.go`
- Existing Obot scan persistence: `pkg/gateway/client/devicescan.go`
- Existing Obot scan ingest: `pkg/api/handlers/devicescans.go`
- Existing Sentry scan submission: `pkg/client/devicescan.go`
