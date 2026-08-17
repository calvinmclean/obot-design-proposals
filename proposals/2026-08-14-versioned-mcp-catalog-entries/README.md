# Versioned MCP catalog entries

- **Authors:** @calvinmclean
- **Created:** 2026-08-14

## Summary

Model an MCP catalog entry as one stable integration with multiple numbered
versions. A publisher can release a breaking change without changing the
entry's name, identity, or connect URL. An administrator can test it, promote
it as the default, and then adopt it through Obot's existing drift, diff, and
explicit update flow.

Promotion does not modify existing deployments. It changes their update target:
deployments that differ from the new default are marked as needing an update,
administrators review the diff, and each deployment changes only when the
update is accepted. Selecting an earlier active version as the default enables
the same flow in reverse.

A key benefit of this proposal is that basic users do not have to migrate to the new
version of our catalog entries and update clients. Instead, Admins will test/verify and
then seamlessly release this using the same connection URL.

## Related issues

- [obot-platform/obot#7376](https://github.com/obot-platform/obot/issues/7376) —
  drives the catalog-wide replacement and deprecation work.
- [obot-platform/obot#7377](https://github.com/obot-platform/obot/issues/7377) —
  pilots the lifecycle by replacing the local Context7 implementation with the
  hosted one.

## Related ODPs

None.

## Problem and motivation

A catalog entry currently has one published definition. Catalog-derived
deployments copy that definition and remain unchanged until a user accepts an
update. Obot already detects drift, shows the diff, validates the candidate,
and preserves compatible URLs and credentials while applying the update.

This supports controlled forward adoption, including changes with new required
configuration. It does not retain the previous catalog definition as a target.
After a deployment accepts an update, it cannot use the catalog to restore the
old configuration. Administrators also cannot create new connections against
both definitions while evaluating a breaking change.

Publishing the replacement as a separate entry retains both definitions, but:

- users see entries such as **Context7** and **Context7 Remote** side by side;
- temporary lifecycle labels become permanent names and lengthen connect URLs;
- the replacement has a different identity and loses its relationship to the
  original; and
- adoption and rollback cannot be managed as one release lifecycle.

The catalog needs to retain both sides of a breaking change while users adopt
it on their own schedule.

## Goals

- Preserve one catalog identity and connect URL across breaking releases.
- Separate publishing, default selection, and deployment adoption.
- Support exact-version testing and concurrent use of active versions.
- Reuse the existing drift, diff, validation, and update flow.
- Support rollback to an earlier active version.
- Record the version applied to each deployment.
- Preserve unversioned entries and existing API consumers.

## Non-goals

- Automatically decide which implementations belong to one entry.
- Automatically migrate deployments after publication or promotion.
- Add configuration preflight. Newly required values may still be collected
  after an update, as they are today.
- Guarantee that configuration, OAuth credentials, or runtime data can migrate
  between versions.
- Retain every unreferenced historical version forever.
- Add semantic-version ranges or cross-entry dependency resolution.

## Context and constraints

Catalog-derived deployments store copied manifests. Drift detection compares
their runtime, package or image, arguments, egress, remote configuration,
environment definitions, multi-user settings, and resources with the current
catalog definition. The UI marks drifted deployments and can show their diff.

On update, Obot builds and validates the candidate before stopping the server,
then copies the catalog definition while preserving compatible connection URLs
and credential values. Newly required shared configuration is detected after
the manifest update; the deployment can remain unconfigured until those values
are supplied. Personal deployments may likewise require later configuration.

Consequently, deployments can incidentally run old and new configurations when
only some have accepted an update. The old definition is not addressable: it
cannot be tested, used for a new connection, selected as an update target, or
restored after an update.

## Proposed design

### Identity and catalog format

Catalog authors publish complete manifests with the same exact `name` and
`entryKey` and distinct positive integer `version` values:

```yaml
entryKey: context7
name: Context7
version: 1
runtime: remote
```

The shared `name` and `entryKey` define the version family. Version numbers are
monotonically increasing release labels, not semantic versions or immutable
content identifiers. Legacy unversioned manifests become version `1`; version
`0` remains reserved for unset or not-yet-migrated references.

Synchronization rejects duplicate version numbers or manifests in a family
that disagree on `name` or `entryKey`. Composite references continue to target
the stable entry and resolve through its default version.

### Versions, defaults, and compatibility

The existing `MCPServerCatalogEntry` remains the stable parent and retains its
current interface. It gains `spec.defaultVersion`, which selects the version
used for new MCP servers and serves as the target for the existing diff/update
flow. It also gains `status.latestVersion`, which lets the UI indicate when a
newer version is available. The existing `spec.manifest` remains as a copy of
the default version's manifest for existing clients.

A new `MCPServerCatalogEntryVersion` resource stores each complete versioned
definition read from the catalog source. The resource layout changes as
follows:

```text
Before
MCPServerCatalogEntry
├── metadata.name                  # stable resource identity
├── spec.mcpCatalogName
├── spec.manifest
├── spec.sourceURL
├── spec.editable / spec.detached
├── spec.powerUserWorkspaceID
└── status.*                       # existing usage, drift, and OAuth state

After
MCPServerCatalogEntry
├── metadata.name                  # unchanged stable identity
├── spec.mcpCatalogName            # unchanged parent metadata
├── spec.sourceURL                 # unchanged parent metadata
├── spec.editable / spec.detached  # unchanged parent behavior
├── spec.powerUserWorkspaceID      # unchanged ownership
├── spec.defaultVersion            # selected update and connection target
├── spec.manifest                  # copy of default version for existing clients
├── status.latestVersion           # greatest active published version
└── status.*                       # other existing status remains unchanged

MCPServerCatalogEntryVersion  # one independent child per version
├── spec.mcpServerCatalogEntryName
├── spec.version
├── spec.manifest
├── spec.sourceURL
└── spec.active

MCPServer
├── spec.mcpServerCatalogEntryName
└── spec.mcpServerCatalogEntryVersion  # version last applied
```

Migration creates a version `1` child from each existing entry's definition,
sets it as the parent's default, and records version `1` on existing
catalog-derived deployments without changing their manifests or the parent's
identity. Version children are siblings identified by parent entry and version
number; they do not point to previous or next versions. Listing versions
queries children by `mcpServerCatalogEntryName` rather than traversing a chain.

A new installation initially selects the latest active version. Later catalog
synchronization may advance `latestVersion` but preserves the installation's
default while it remains active. Existing deployments also retain their copied
manifests and record the version last applied to them.

Publishers may correct an existing version in place; they should use a new
number when a change needs a separate evaluation and adoption cycle. Existing
deployments do not follow such edits. An update binds to the resolved target
content and fails if that content changes before commit, requiring review and
retry.

### Adoption workflow

Versioning chooses and retains update targets; the existing update flow applies
them:

1. **Publish:** synchronization stores the new active version and advances
   `latestVersion`. Defaults and deployments do not change.
2. **Evaluate:** an administrator inspects the version and creates a pinned test
   deployment through an exact-version URL.
3. **Promote:** the administrator selects the version as the default. New stable
   connections use it; existing deployments keep their manifests but enter the
   existing `needsUpdate` flow when they differ.
4. **Review and apply:** the existing diff compares each deployment with the
   default. An accepted update uses the existing validation and apply behavior,
   then records the target version after the manifest update succeeds.

To roll back, an administrator selects an earlier active version as the default,
reviews the reverse diff, and explicitly updates affected deployments. This
restores catalog configuration, not runtime data lost during an earlier
transition.

Bulk rollout attempts deployments independently and reports each result. A
validation or update failure leaves that deployment unchanged and does not stop
the remaining attempts. Missing required configuration is not a pre-apply
blocker and may leave an updated deployment temporarily unconfigured.

Normal users continue to see one catalog card. Version history, testing,
default selection, and rollout status are administrative concerns.

### Withdrawal and cleanup

Only active versions can be tested, selected as the default, or used as update
targets. A withdrawn version remains inactive while a deployment references it
and is deleted once it is neither referenced nor the default. A temporary
catalog synchronization failure must not deactivate or delete versions.

## Alternatives considered

### Add `displayName` and publish parallel entries

Keep separate identities such as `Context7` and `Context7 v2`, but present
friendlier mutable labels. This is much simpler when naming is the only problem.
It also preserves today's explicit update behavior for each entry. We can either:
- present them side by side with the same `displayName`, differentiated only by
  the deprecated badge and descriptions; or
- start with different names and rename the replacement after deleting the old
  entry, which may confuse users.

It does not model the entries as releases of one integration, provide one stable
URL, select a default, record an applied version, or provide a rollback target.
It is therefore a naming workaround rather than an adoption model. This might be
worthwhile to include for flexibility, but doesn't cleanly solve our catalog upgrade
scenario.

### Publish visibly distinct entries

Publish **Context7 V2** beside **Context7** and later remove the original.
This requires no versioning work, but exposes lifecycle state in names,
fragments identities and URLs, and offers no managed relationship between the
entries.

## Trade-offs

Versioning adds a persistent release resource, migration and retention rules,
exact-version routing, version-aware resolution, administrative UI, and audit
data. Catalog authors must decide when to publish a new version rather than an
in-place correction. A prototype changes roughly 5,640 lines across 66 files,
so the operational lifecycle—not naming alone—must justify the cost.

In return, administrators can evaluate, adopt, and reverse breaking changes as
releases of one stable integration. The simpler alternatives avoid most of the
implementation cost but fragment catalog identity or give up addressable
rollback targets.

## Risks and open questions

Known risks are:

- Older Obot releases use permissive YAML decoding and ignore the new `version`
  field. If their catalog source publishes multiple manifests with the same
  identity, they interpret them as duplicate unversioned entries, which can
  collide or expose one definition without version-aware adoption controls.
  Multi-version catalogs must therefore have a minimum compatible Obot version.
- Mutable versions are not historical proof. The deployment's stored manifest,
  not its version number, remains the authoritative applied configuration.
- Apply-then-configure can leave a deployment unavailable after a version adds
  required configuration. Preflight is separate follow-up work.
- Rollback across runtime or authentication models cannot restore lost runtime
  data and may require new authorization.
- Catalog review must prevent unrelated integrations from sharing a version
  family merely to avoid creating a new entry.
- Existing API clients see only the projected default, not version history.
- Changing a component's default must not mutate existing deployed composites.

Open questions for review are whether the operational lifecycle justifies the
implementation cost, whether `displayName` should be added independently for
cases where safe renaming is the only need, and whether incompatible Obot
installations should be excluded through a minimum supported version or remain
on a separate legacy catalog source.

## Rollout and migration

1. Add the version resource, version fields, version-aware resolution, and the
   default-version copies retained on the parent for existing clients. Keep the
   shared catalog source unversioned during this phase; version-aware Obot maps
   those manifests to version `1`.
2. Migrate current entries and deployments to version `1` without changing IDs,
   URLs, manifests, or behavior. This explicitly writes the applied version
   rather than relying on an integer field's zero value.
3. Add synchronization, target-content consistency checks, exact-version
   testing, default selection, target-aware diffs, and per-deployment and bulk
   updates.
4. Before publishing multiple versions, establish a compatibility boundary:
   either all supported Obot releases understand version families, or upgraded
   installations move to a versioned catalog source while the legacy source
   remains unchanged for older installations.
5. Pilot with Context7: retain the local implementation as version `1`, publish
   the hosted implementation as version `2`, evaluate it, promote it, and
   explicitly migrate deployments.
6. Extend the model after the pilot validates configuration and OAuth changes,
   partial failures, cleanup, and rollback.

Before promotion, removing a candidate has no user impact. After promotion,
rollback uses an earlier active default and the same explicit update flow.

## Testing and validation

- Unversioned entries and their deployments migrate to version `1` without
  changing identity, URL, configuration, or behavior.
- An older Obot release continues to consume the unchanged legacy catalog, and
  no default source exposes multiple same-identity manifests to an incompatible
  release.
- Publishing advances `latestVersion` without changing an existing default or
  deployment; new installations initially select the latest active version.
- Promotion changes new stable connections and causes differing deployments to
  enter the existing `needsUpdate` and diff flow without modifying them.
- Exact-version tests do not affect the default, and deployments on different
  active versions can run concurrently.
- Forward updates and rollbacks preserve current validation, URL, credential,
  and post-update configuration behavior and record a version only after the
  manifest update succeeds.
- A changed target fails an in-progress update before commit.
- Bulk rollout continues after individual failures and reports each result.
- Referenced withdrawn versions remain inactive; unreferenced non-default
  versions are deleted; source failures do not trigger cleanup.
- New composite resolution uses current defaults without changing deployed
  composite configuration.
- The full Context7 lifecycle succeeds in staging.

## References

- [Versioned catalog prototype branch](https://github.com/calvinmclean/obot/tree/feat/versioned-mcp-catalog)
