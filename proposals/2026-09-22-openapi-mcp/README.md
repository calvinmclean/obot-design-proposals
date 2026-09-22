# 2026-09-22: OpenAPI-backed MCP catalog entries

- **Authors:** @calvinmclean
- **Created:** 2026-09-22

## Summary

Let catalog authors upload an OpenAPI specification or provide its URL to create
an MCP catalog entry. Run a reusable FastMCP container that converts the API to
MCP using FastMCP's OpenAPI integration, based on the existing Python example.
The wrapper source and container build live in `mcp-images`.
Users configure API keys or OAuth and compose the entry into a vMCP.

Configuration allows users to enable FastMCP Tool Search and, only when search
is enabled, supply basic rules that disable tools inside FastMCP. Without Tool
Search, tool selection belongs to vMCP.

## Related issues

- [obot#5555](https://github.com/obot-platform/obot/issues/5555) — expose REST
  APIs through Obot catalogs and the MCP gateway.
- [obot#1247](https://github.com/obot-platform/obot/issues/1247) — historical
  context on credential refresh in earlier OpenAPI tools; this remains closed.

## Related ODPs

- [Virtual MCPs](../2026-09-01-virtual-mcps/README.md) — extends the available
  catalog component types and follows its connection, configuration, and
  snapshot/upgrade model.

## Problem and motivation

Services often publish OpenAPI specifications but do not serve MCP. Users should
be able to connect them without building a custom integration. FastMCP already
provides OpenAPI conversion and Tool Search; a configurable container lets Obot
reuse these features through its existing hosted-MCP paths.

## Goals

- Create catalog entries from uploaded or linked specifications.
- Use a shared container image, configured per API, with no per-API code generation.
- Support API keys, OAuth, and vMCP composition.
- Make Tool Search configurable and allow FastMCP exclusions only in search mode.
- Keep running entries stable until an explicit upgrade.

## Non-goals

Custom search/HTTP execution engines or full OpenAPI compatibility in the first
release.

## Context and constraints

The local Python example uses FastMCP's OpenAPI integration to generate direct
tools and can replace their listing with BM25 Tool Search. Both direct calls and
search followed by invocation have passed live tests. Authentication and
container deployment still need implementation.

The production image must pin and test its dependencies rather than assume
every valid specification is supported.

## Proposed design

### Container and catalog setup

Keep the Python wrapper, dependencies, tests, and Dockerfile in `mcp-images` so
the image can be built directly from that repository. Package it in one reusable
image. Each deployment receives the
specification, API base URL, authentication configuration, Tool Search setting,
and any permitted exclusion rules. FastMCP exposes Streamable HTTP for Obot to
connect to through its existing containerized runtime.

```text
MCP client → Obot gateway / vMCP → FastMCP container → REST API
```

Obot owns catalog setup, deployment, connections, and gateway access controls.
FastMCP owns OpenAPI conversion, tool schemas, search, and HTTP request execution.
No new protocol translation handler is needed inside Obot.

Import previews generated tools and reports unsupported features. Store a
versioned copy of the specification; a linked URL is its refresh source.
Review specification and configuration changes through the vMCP upgrade flow.
A failed import must not replace a working revision.

### Tool Search and filtering

| Mode | What FastMCP lists | Where tools are restricted |
| --- | --- | --- |
| Search disabled | Generated API tools | vMCP tool selection; FastMCP exclusion rules are not allowed |
| Search enabled | search_tools and call_tool | FastMCP exclusions restrict underlying operations; vMCP controls access to the exposed search/call tools |

Enable search using FastMCP's BM25 search transform. It returns matching tool
definitions and input schemas, and call_tool invokes a discovered tool. Use the
library's implementation rather than adding custom general tools.

Expose a small declarative set of exclusion rules using FastMCP's OpenAPI route
mapping, such as HTTP method, path pattern, and tag. The exact rule fields are
proposed for review. Rules only disable matching operations; they do not run
user-provided Python or enable excluded tools. Apply exclusions when creating
the OpenAPI tools, before applying Tool Search.

For example, a proposed configuration could be:

```yaml
toolSearch: true
exclude:
  - methods: [DELETE]
  - pathPattern: "^/admin/"
```

These field names are illustrative, not a finalized catalog schema.

Validate the combination in both Obot and the container: nonempty exclusions
with search disabled are an error, not silently ignored. The UI only offers
exclusions in search mode. Switching search off requires removing the rules and
reviewing the resulting direct tools and vMCP selections.

This distinction matters because vMCP sees individual operations without search,
but sees only the search and invocation tools with search enabled. It cannot
restrict the operations behind call_tool using ordinary tool-name selection.
FastMCP exclusions therefore define the operations available to that configured
server. They are not a new per-user vMCP permission system. Users needing
different underlying operation sets need separately scoped configurations or
deployments; incompatible rule sets must not share one FastMCP instance.

Tool Search hides direct tools from listing but keeps them callable. Excluded
operations must instead be unavailable through search, call_tool, and direct
calls by name. Verify this behavior against the pinned FastMCP release.

### Authentication

Keep the client's Obot credentials separate from upstream API credentials.
Support API-key injection and OAuth, following existing connection ownership
rules for per-user and explicitly shared credentials. Secrets must not appear in
specifications, tool arguments, descriptions, or results.

API keys use the configured header, query parameter, or cookie. OAuth requires
client setup, consent, token storage, refresh, and reconnect behavior; merely
supplying a bearer token is insufficient. The division of this work between
Obot and the container needs review. Do not assume existing MCP OAuth handling
automatically supplies REST-provider tokens.

Credential or filter differences must be reflected in deployment/connection
isolation so users cannot inherit another connection's access.

### Failures and operations

Use the existing hosted-MCP lifecycle for startup, health, restart, and shutdown.
Validate configuration before serving requests. Bound parsing, HTTP requests,
concurrency, and response sizes; return useful tool errors for upstream failures.
Apply network policy to specification/reference loading, OAuth, and API requests,
including redirects. Containerization does not remove these requirements.

Keep existing gateway auditing and secret redaction. In search mode, ordinary
audit records identify call_tool; confirm how to record its target operation and
how hooks or approvals inspect that target before enabling sensitive operations.
Confirm FastMCP's session behavior works with existing proxying and deployment
routing rather than treating a container as a solution to session state.

## Alternatives considered

- **Generate or hand-write each MCP:** allows tailored tools but adds maintenance
  and per-API builds.
- **vMCP filtering alone in search mode:** cannot select individual operations
  behind a single call_tool. FastMCP must exclude them before invocation.

## Trade-offs

This reuses FastMCP and Obot's container infrastructure, but adds Python
dependencies and workload overhead.

Search reduces the advertised tool list but moves operation filtering into
container configuration. Direct mode keeps familiar vMCP tool selection and
avoids a second filtering configuration.

## Risks and open questions

Resolve during review:

- Should Tool Search be enabled by default? Which exclusion matchers ship first?
- How are specification content and configuration passed to the container?
- Which API authentication flows are supported first, and who owns OAuth refresh?
- How do existing deployment-sharing rules account for credentials and exclusions?
- How do auditing, hooks, and approvals handle call_tool targets?
- Which OpenAPI features, resource limits, and session behaviors are supported?

## Rollout and migration

Introduce the container image and catalog creation flow without changing existing
entries. Validate unauthenticated APIs, then API keys and OAuth. No automatic
migration is needed. Pin image and specification revisions and use explicit
upgrades. Reverting a deployment must restore its filtering configuration too;
never fall back to an unfiltered server after a configuration error.

## Testing and validation

Verify the image builds directly from `mcp-images`. Test container startup and
real MCP calls through Obot and a vMCP in both modes.
Without search, verify direct tool restrictions are enforced by vMCP and
FastMCP exclusions are rejected. With search, verify excluded operations cannot
be found or called by either invocation path.

Cover invalid rules, mode changes, isolated configurations, credential refresh,
secret redaction, upstream errors, session routing, and failed upgrades. Use
local APIs for repeatable tests and a public API for optional live smoke tests.

## References

- [FastMCP OpenAPI](https://gofastmcp.com/integrations/openapi) — conversion and
  route exclusion rules.
- [FastMCP Tool Search](https://gofastmcp.com/servers/transforms/tool-search) —
  discovery and invocation.
