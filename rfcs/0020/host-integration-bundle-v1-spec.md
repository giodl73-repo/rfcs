# Host Integration Bundle v1 Core Specification

This document is the implementer-facing bundle specification for RFC 0020,
Hosted Owner Bindings and Managed Dispatch. The RFC explains why host
integration remains owner-oriented. This file defines the cell-local bundle,
contribution inventory, typed reference, generation, registration, and
diagnostic contract.

Status: draft, tied to RFC 0020.

## Scope

This specification defines:

- one immutable host integration bundle snapshot per Gateway cell;
- bundle manifests and runtime availability assertions;
- typed contribution identity and provenance;
- atomic validation and publication;
- contribution reference resolution;
- bundle generations and snapshot-scoped disposal;
- owner activation boundaries;
- readiness, Status, and Doctor projection requirements;
- failure categories and conformance checks.

This specification intentionally does not define:

- product-specific plugin configuration;
- provider, Channel, approval, lifecycle, or identity semantics;
- credential value acquisition;
- provider request traffic-policy rule syntax;
- hosted-dispatch carrier frames;
- fleet-global activation or a cross-tenant registry.

Those concerns remain with their semantic owners or the other RFC 0020
sidecars.

Readiness composition is defined by
[`host-integration-readiness-v1-spec.md`](host-integration-readiness-v1-spec.md).

## Core Invariant

One complete Gateway cell may publish at most one current host integration
bundle snapshot.

The bundle packages independently owned implementations. It does not become
their semantic owner and does not activate them as one distributed
transaction.

```text
external plugin loads in one Gateway cell
  -> plugin validates its complete contribution inventory
  -> one immutable bundle snapshot is published
  -> owner configuration resolves typed contribution references
  -> each owner independently prepares and activates its generation
  -> Status, Doctor, and readiness observe owner evidence
```

Installing the same plugin in multiple cells creates independent registrations,
snapshots, generations, failures, and readiness states.

## Version

The v1 contract identifier is:

```text
host-integration-bundle/v1
```

Implementations must reject unsupported major contract versions before
publishing a snapshot.

## Identifiers

Bundle and contribution ids are stable namespaced identifiers:

```text
<namespace>/<name>
```

Examples:

```text
example/managed-host
example/managed-egress
msteams/acf-token
m365mail/graph-token
```

Ids are authority-bearing references, not display names. A renamed display
label must not change an id. Implementations should restrict ids to lowercase
ASCII letters, digits, `.`, `_`, `-`, and `/`.

Bundle package versions use exact semantic versions. Contribution contract
versions are exact opaque contract identifiers such as
`credential-slot-resolver/v1`.

## Bundle Manifest

A v1 bundle manifest has this logical shape:

```json
{
  "version": "host-integration-bundle/v1",
  "id": "example/managed-host",
  "bundleVersion": "1.0.0",
  "contributions": []
}
```

| Field | Type | Required | Semantics |
| --- | --- | --- | --- |
| `version` | string | Yes | Must be `host-integration-bundle/v1`. |
| `id` | string | Yes | Stable namespaced bundle id. |
| `bundleVersion` | string | Yes | Exact semantic version of the external bundle package contract. |
| `contributions` | array | Yes | Complete desired contribution inventory for this bundle generation. |

The manifest is declarative. Static package discovery alone is not proof that a
runtime implementation loaded successfully.

## Contribution Declaration

Each contribution declaration has:

| Field | Type | Required | Semantics |
| --- | --- | --- | --- |
| `owner` | string | Yes | Semantic owner namespace. |
| `kind` | string | Yes | Owner-defined contract kind. |
| `id` | string | Yes | Stable namespaced implementation id. |
| `version` | string | Yes | Exact owner contract version. |
| `required` | boolean | Yes | Whether bundle registration fails when the contribution is unavailable or incompatible. |
| `readinessCriteria` | string array | Yes | Canonical RFC 0018 selector ids associated with this contribution. |

Initial v1 contribution kinds include:

| Owner | Kind | Purpose |
| --- | --- | --- |
| `model-provider` | `model-provider-adapter` | Provider-owned request and response semantics. |
| `web-search-provider` | `web-search-provider-adapter` | Web-search-owner request and result semantics. |
| `provider-request` | `credential-slot-resolver` | Exact credential-slot materialization. |
| `provider-request` | `provider-request-traffic-policy` | Host route and destination constraints. |
| `provider-request` | `provider-request-dispatcher` | Physical one-hop execution. |

An implementation may support additional owner/kind pairs only when the owner
publishes a versioned contract and conformance suite. Unknown owner/kind pairs
must not be treated as generic host methods.

Contribution ids must be unique across the bundle. Readiness selectors must
use canonical `openclaw.*` or `plugin.*` ids, be unique within one
contribution, and remain bounded to 64 unique selectors across the bundle. A
contribution cannot reference the aggregate
`openclaw.host-bindings-ready` selector because that would be recursive.

`provider-request` is the non-semantic owner namespace for credential
placement, traffic-policy intersection, and physical dispatch. It must not
define provider, Channel, web-search, mail, identity, response, or replay
semantics.

`readinessCriteria` is declaration metadata. It does not register a readiness
provider, select a criterion, make it required, or prove that a contribution
is ready. Each declared id resolves directly against the active RFC 0018
criterion catalog as defined by the readiness sidecar.

## Runtime Availability Assertion

Before publication, the plugin supplies or derives a runtime availability
assertion for each implementation it claims is loaded:

```json
{
  "owner": "provider-request",
  "kind": "credential-slot-resolver",
  "id": "example/provider-token",
  "version": "credential-slot-resolver/v1",
  "provenance": {
    "pluginId": "example-host",
    "source": "/plugins/example-host/index.js",
    "origin": "config"
  }
}
```

| Provenance field | Required | Semantics |
| --- | --- | --- |
| `pluginId` | Yes | Loaded OpenClaw plugin id. |
| `source` | Yes | Bounded package or configuration source label. |
| `origin` | Yes | `bundled`, `config`, `global`, or `workspace`. |

A live plugin registration call may serve as the runtime availability assertion
when it executes only after the corresponding implementations loaded and the
plugin API derives trusted provenance. Reading a static manifest from disk is
not sufficient by itself.

Availability assertions contain implementation identity and contract metadata,
not credential values, tokens, private keys, or live dispatcher handles.

## Validation And Publication

Registration is atomic:

1. Validate the bundle version, id, and exact package version.
2. Validate and normalize every contribution declaration.
3. Reject duplicate ids, duplicate owner/kind/id keys, and malformed criteria.
4. Validate runtime availability assertions and provenance.
5. Match declarations to availability by owner, kind, and id.
6. Mark each declaration `resolved`, `missing`, or `incompatible`.
7. Fail registration when any required declaration is missing or incompatible.
8. Freeze and publish the complete snapshot.
9. Notify owner and diagnostic observers only after publication succeeds.

No contribution becomes visible before the complete required inventory passes.
Failed registration may publish a separate diagnostic status snapshot, but it
must not replace the last authoritative runtime snapshot with a partial bundle.

## Bundle Snapshot

The published snapshot has:

```json
{
  "version": "host-integration-bundle/v1",
  "id": "example/managed-host",
  "bundleVersion": "1.0.0",
  "generation": "opaque-cell-local-generation",
  "inventory": []
}
```

Inventory entries contain the declaration plus:

| Field | Required | Semantics |
| --- | --- | --- |
| `status` | Yes | `resolved`, `missing`, or `incompatible`. |
| `resolvedVersion` | Conditional | Runtime implementation version when one was observed. |
| `provenance` | Conditional | Runtime plugin provenance when one was observed. |

Snapshots and nested inventory records are immutable.

`generation` is an opaque cell-local identifier. Consumers may compare it for
equality but must not parse it for authorization or ordering. A new accepted
registration creates a new generation even when the bundle id and package
version are unchanged.

## Registration Ownership

V1 allows one authoritative bundle registration per Gateway cell.

- A competing plugin registration fails closed.
- The Gateway serializes concurrent registration attempts so the ownership
  check and snapshot publication are atomic.
- Existing plugin activation order determines registration attempt order;
  asynchronous validation completion must not select the winner.
- Re-registering from the current owner creates a new immutable generation.
- A disposer or unload callback may clear only the snapshot it published.
- Disposing an older snapshot must not clear a newer registration.
- Clearing the bundle invalidates dependent owner and carrier authority.

Fleet or another control plane may author normal plugin configuration, but it
does not hold a fleet-global runtime registration.

## Typed Reference Resolution

Owner configuration selects a contribution with an exact typed reference:

```json
{
  "owner": "provider-request",
  "kind": "credential-slot-resolver",
  "id": "example/provider-token",
  "version": "credential-slot-resolver/v1"
}
```

Resolution must:

- use one immutable snapshot supplied by the caller or the current cell
  snapshot;
- match owner, kind, and id exactly;
- require `status: "resolved"`;
- require the declared, requested, and resolved versions to match;
- return bounded metadata only;
- fail explicitly when the bundle is absent, the contribution is unknown, or
  the version is incompatible.

Reference resolution does not activate the implementation and does not acquire
credentials.

## Owner Activation

After reference resolution, each semantic owner independently:

1. validates owner configuration and trusted identity;
2. resolves all typed contribution references from one bundle snapshot;
3. validates implementation compatibility and authority;
4. prepares an immutable owner binding;
5. activates one authoritative owner generation;
6. publishes one immutable bounded activation snapshot; and
7. lets canonical readiness evaluate selected criteria from that snapshot.

There is no cross-owner activation transaction. One owner may be ready while
another remains unavailable.

## Generations And Fencing

Remote or side-effecting operations carry at least:

- owner generation;
- host bundle generation; and
- binding or carrier incarnation when physical execution is remote.

A new bundle generation requires owner re-resolution and remote re-admission.
Stale requests, frames, and results fail closed. A stale late result may still
represent an indeterminate provider side effect; fencing does not make replay
safe.

## Readiness, Status, And Doctor

Readiness uses the canonical RFC 0018 condition model and the RFC 0020
readiness sidecar. The bundle does not define another readiness state enum.

Status inventory combines bundle resolution and owner activation snapshots
without probing, materializing credentials, mutating configuration, or
activating owners.

Diagnostics must:

- report one cell only;
- preserve bundle, owner, and carrier generations;
- distinguish missing, incompatible, stale, and owner-unavailable states;
- redact secrets, credential values, unbounded owner messages, and private
  implementation details;
- keep reason codes bounded and machine-readable;
- never let bundle health override an owner failure.

Operator readiness selection or an opt-in Hosting Profile decides which owner
criteria are individually required. The bundle's `required` field controls
registration completeness and membership in the explicitly selected
`openclaw.host-bindings-ready` aggregate; it never promotes a referenced
criterion from advisory to required. The readiness sidecar defines that
aggregate behavior.

## Failure Categories

Implementations should expose stable categories equivalent to:

- `invalid-manifest`
- `duplicate-contribution`
- `duplicate-available-contribution`
- `unknown-readiness-criterion`
- `missing-required-contribution`
- `incompatible-required-contribution`
- `bundle-not-registered`
- `unknown-contribution`
- `incompatible-contribution`
- `competing-registration`
- `stale-generation`

Messages must be bounded and redacted. Diagnostics may include the affected
contribution id but not credential values or raw implementation state.

## Example

```json
{
  "version": "host-integration-bundle/v1",
  "id": "example/managed-host",
  "bundleVersion": "1.0.0",
  "contributions": [
    {
      "owner": "provider-request",
      "kind": "credential-slot-resolver",
      "id": "example/provider-token",
      "version": "credential-slot-resolver/v1",
      "required": true,
      "readinessCriteria": ["plugin.example-host.provider-credentials"]
    },
    {
      "owner": "provider-request",
      "kind": "provider-request-traffic-policy",
      "id": "example/managed-egress",
      "version": "provider-request-traffic-policy/v1",
      "required": true,
      "readinessCriteria": ["plugin.example-host.provider-policy"]
    },
    {
      "owner": "provider-request",
      "kind": "provider-request-dispatcher",
      "id": "example/reverse-provider",
      "version": "provider-request-dispatcher/v1",
      "required": true,
      "readinessCriteria": ["plugin.example-host.provider-dispatch"]
    }
  ]
}
```

The ids above are examples. OpenClaw core does not reserve or ship an
`example/managed-host` bundle.

## Compatibility And Evolution

- Unknown bundle major versions fail closed.
- Contribution contract versions are owned independently by their semantic
  owners.
- New optional snapshot metadata may be ignored.
- New required fields require a new bundle contract version.
- New owner/kind pairs require an owner contract and conformance suite.
- A bundle update must not silently retarget an existing typed reference to a
  different owner or kind.

## Bundle Publisher Checklist

An external host plugin is compatible when it:

- registers one complete cell-local bundle;
- uses stable namespaced ids and exact contract versions;
- asserts runtime availability only after implementations load;
- derives trusted provenance through the plugin lifecycle;
- treats readiness criterion declarations as metadata, not success evidence;
- keeps credentials and dispatcher state out of the inventory;
- handles unload and re-registration with snapshot-scoped disposal;
- publishes no product semantics through a generic owner/kind pair.

## OpenClaw Client Checklist

An OpenClaw implementation is compatible when it:

- validates before publication;
- publishes one immutable current snapshot;
- rejects competing registrations;
- resolves typed references from one snapshot;
- fences owner and carrier work by bundle generation;
- preserves independent owner activation;
- exposes redacted Status and Doctor evidence;
- proves missing, incompatible, stale, and disposal behavior.

## Conformance Fixtures

The v1 suite should include:

- a valid multi-owner bundle;
- duplicate contribution ids;
- missing required and optional contributions;
- incompatible required and optional versions;
- malformed ids, versions, criteria, and provenance;
- an unpublished owner/kind pair without an owner contract;
- an unknown owner readiness criterion;
- competing registration;
- same-owner re-registration;
- stale disposer behavior;
- snapshot-scoped reference resolution;
- independent owner readiness;
- redaction of owner-provided diagnostic text;
- two cells loading the same plugin with independent generations.
