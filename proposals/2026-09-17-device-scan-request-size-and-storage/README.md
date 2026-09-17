# 2026-09-17: Device Scan Request and Storage Size

- **Authors:** @calvinmclean
- **Created:** 2026-09-17

## Summary

<!-- A short explanation of the change and the proposed approach. -->
`obot-sentry` device scan submissions can fail due to request size limits in Nginx and in Obot. There is no indication in Obot that scans failed. We need to improve how large requests are handled and indicate if a scan upload fails.

This proposal advocates for a multi-part approach:
- If requests fail due to large size, truncate the large skill markdown files and add a field indicating that it was truncated. If that also fails, send a simple request to indicate the failure and reason
- Also use this failure indicator for other failures
- Fingerprint parts of the upload to de-duplicate previous uploads. This will be stored client-side and server-side to reduce upload size and storage size
- Needs further discussion: add a product telemetry field to track uploads that require truncation so we can see if this is a consistent issue. We need to determine if this is an appropriate metric to collect
- It also explores the possibility to break scans into multiple requests, but decides against it since it is complex and and failure-prone. Metrics can help us decide if this is necessary in the future


## Related issues

<!--
Link every GitHub issue driving this work. Add a short description when an
issue's relationship to the proposal is not obvious. Every proposal must have
at least one related issue.
-->
[Large device scan submissions fail without reporting scan failure](https://github.com/obot-platform/obot/issues/7506)

## Related ODPs

<!--
Link related Obot Design Proposals and state the relationship, such as
"supersedes," "extends," "depends on," or "provides context for." Write "None"
when there are no related ODPs.
-->
None

## Problem and motivation

<!-- What problem are we solving, for whom, and why is it worth solving now? -->
Obot-sentry fails to send scans that are too large for Nginx or Obot API handler. This means we silently lose scan data indefinitely for devices with large amount of skills. We need to solve this so we can effectively collect data as intended.


## Goals

<!-- Outcomes this design must achieve. Prefer observable results. -->
- If/when scan uploads fail for reasons other than unreachable server, we need to record these failures in Obot for admins to see
- Collect at least partial data when request size is too large
- Reduce storage size within Obot backend


## Non-goals

<!-- Boundaries that prevent readers from assuming a broader scope. -->

## Context and constraints

<!--
Existing behavior and architecture, compatibility requirements, scale or
performance constraints, security boundaries, and relevant prior decisions.
Link to source or documentation where useful.
-->

## Proposed design

<!--
Explain how the design works. Cover components, responsibilities, interfaces,
data flow, APIs, schemas, persistence, and failure behavior as applicable.
Use examples and diagrams when they make the design easier to evaluate.
-->

## Alternatives considered

<!-- Include the status quo and the strongest credible alternatives. -->

### Alternative name

<!-- Brief description and why it was not selected. -->

## Trade-offs

<!--
Describe the trade-offs of the proposed design relative to the important
alternatives. What becomes easier or harder? What cost or flexibility are we
giving up?
-->

## Risks and open questions

<!--
List known risks, unresolved decisions, and assumptions that need validation.
Name an owner or resolution point for open questions when possible.
-->

## Rollout and migration

<!--
Describe sequencing, compatibility during transition, feature gates, data
migration, observability, and rollback. Write "Not applicable" when appropriate.
-->

## Testing and validation

<!--
How will we demonstrate correctness and know the change met its goals? Include
unit, integration, end-to-end, performance, security, or operational validation
as relevant.
-->

## References

<!-- Related proposals, ADRs, issues, prior art, or external documentation. -->
