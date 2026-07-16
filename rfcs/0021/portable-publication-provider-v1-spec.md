# Portable Publication Provider v1 Specification

This document is the implementer-facing publication and retrieval specification
for RFC 0021, Runtime State Continuity. The RFC defines lifecycle, capture,
restore, hibernation, and wake semantics. The
[State CAPE v1 Specification](state-cape-v1-spec.md) defines the capability
levels. This sidecar defines the Portable v1 provider contract and managed
source-to-fresh-process proof.

Status: draft, tied to RFC 0021.

## Scope

This specification defines:

- external continuity publication provider discovery and registration;
- manifest ownership and operator selection;
- provider, plugin, contract-version, and generation identity;
- immutable publication acceptance;
- bounded streaming publication and retrieval;
- the strict managed publication operation;
- fresh-process retrieval verification;
- fail-closed errors, replay, and conformance requirements.

This specification does not define:

- artifact capture mechanics or manifest contents;
- restore target allocation or activation;
- object-store, filesystem, sidecar, or hosted storage implementation;
- credentials, provider URLs, or storage policy;
- retention and garbage collection policy;
- Hosted Integration bundles, reverse carriers, or product-specific extensions;
- Elastic wake and retained-ingress semantics.

Optional canonical Readiness, Hosting Profile, and Hosted Integration
composition is defined by the
[State CAPE Readiness and Hosting Composition v1 Addendum](readiness-hosting-composition-v1-addendum-spec.md).

## Model

Portable publication is a continuity-owned capability supplied by an external
OpenClaw plugin.

```text
plugin manifest declares continuityPublicationProviders: [providerId]
  -> continuity.publicationProvider selects providerId
  -> manifest metadata resolves one enabled owner plugin
  -> managed operation loads only that plugin
  -> runtime resolves exact plugin/provider/version/generation
  -> process A publishes and stops
  -> process B loads the same external plugin and retrieves
  -> OpenClaw verifies the complete accepted artifact
```

The provider can use a mounted filesystem, sidecar, object-store client, hosted
service, or another durable implementation. The contract standardizes
continuity semantics, not storage transport.

Portable publication does not depend on Hosted Integration. A future generic
host-composition layer may improve installation or configuration UX without
changing this contract.

## Authority boundaries

| Owner | Authority |
| --- | --- |
| Capture owner | Native state capture semantics and artifact construction. |
| Continuity | Aggregate artifact identity, publication request, acceptance validation, retrieval validation, and lifecycle use. |
| Plugin manifest | Static declaration that one plugin can own a provider ID. |
| Operator config | Selection of the provider ID used by this runtime. |
| Provider plugin | Durable storage implementation and stable provider generation. |
| Lifecycle coordinator | Handoff order, replay, source retirement, and restore orchestration. |

The provider does not decide whether capture is complete, whether source
compute is safe to destroy, or whether restored startup is ready.

## Discovery and selection

### Plugin manifest

An external provider plugin declares every continuity publication provider ID
it can register:

```json
{
  "id": "acme-continuity",
  "name": "Acme Continuity",
  "configSchema": {
    "type": "object",
    "additionalProperties": false
  },
  "contracts": {
    "continuityPublicationProviders": ["acme/continuity"]
  }
}
```

Provider IDs must be stable, lowercase, namespaced identifiers. The v1 shape is:

```text
<namespace>/<provider>
```

Mutable display names, plugin paths, endpoint URLs, and credentials are not
provider IDs.

### Operator config

Portable runtimes select one provider:

```json5
{
  continuity: {
    level: "portable",
    publicationProvider: "acme/continuity",
  },
  plugins: {
    entries: {
      "acme-continuity": {
        enabled: true,
        config: {
          // Provider-owned configuration.
        },
      },
    },
  },
}
```

The selected provider must have exactly one effectively enabled manifest owner.
Disabled declarations do not create ambiguity. Missing ownership and multiple
enabled owners fail closed before plugin import.

### Scoped loading

The managed operation must:

1. resolve the selected provider ID from manifest metadata;
2. resolve exactly one enabled owner plugin ID;
3. load only that plugin;
4. require that plugin to register the selected provider ID;
5. reject registration from another plugin;
6. start only services from the scoped registry.

The operation must not load every plugin to discover the provider.

## Runtime registration

The plugin registers a v1 provider through the ordinary Plugin SDK:

```ts
api.registerContinuityPublicationProvider({
  id: "acme/continuity",
  version: "continuity-publication-provider/v1",
  generation: "storage-contract-2026-07",
  async publish(request) {
    // Store exact request.content bytes durably.
    // Return immutable acceptance evidence.
  },
  async retrieve(request) {
    // Return the bytes bound by request.receipt.
  },
});
```

Registration fields:

| Field | Type | Required | Semantics |
| --- | --- | --- | --- |
| `id` | string | Yes | Stable namespaced provider ID declared by the manifest. |
| `version` | string | Yes | Must be `continuity-publication-provider/v1`. |
| `generation` | string | Yes | Stable compatibility generation for durable receipt replay across processes. |
| `publish` | function | Yes | Bounded asynchronous byte-stream publication. |
| `retrieve` | function | Yes | Bounded asynchronous byte-stream retrieval. |

The provider generation is not a process-local registration counter. It must
remain stable while previously accepted receipts are retrievable under the same
provider semantics. A generation change invalidates old bindings unless an
explicit migration contract is introduced in a later version.

Provider generation is a compatibility generation. It carries no runtime
admission, lifecycle fencing, destruction, or tenant authority.

Duplicate runtime registration of one provider ID fails closed.

## Identity

### Artifact identity

Every publication binds:

| Field | Semantics |
| --- | --- |
| `ownerId` | Stable continuity owner or tenant-cell identity. |
| `sourceRuntimeGeneration` | Exact source runtime generation. |
| `handoffId` | Durable lifecycle handoff identity. |
| `captureId` | Exact final-capture identity. |
| `archiveSha256` | SHA-256 of the complete archive bytes. |
| `manifestSha256` | SHA-256 of the embedded continuity manifest. |
| `archiveSize` | Exact archive byte length. |

### Provider binding

The frozen binding contains:

| Field | Semantics |
| --- | --- |
| `pluginId` | Manifest-resolved external plugin owner. |
| `id` | Selected provider ID. |
| `version` | Provider contract version. |
| `generation` | Stable provider compatibility generation. |

These fields are cross-process authority. A plugin path, module instance,
registry snapshot, service process ID, or host-bundle generation is not.

The artifact `ownerId` is the continuity tenant-cell boundary. Publication,
retrieval, and restore must reject an artifact whose owner differs from the
requesting logical runtime's trusted owner identity. Caller-supplied receipt
data cannot select or override that identity.

## Publication

The provider receives:

```ts
{
  identity,
  content: AsyncIterable<Uint8Array>,
  signal: AbortSignal
}
```

The provider must either:

- return an immutable acceptance for the exact identity and bytes;
- return the same acceptance for exact replay; or
- fail with a structured provider failure.

Reusing a handoff or publication identity for different artifact identity must
conflict. Providers must not return acceptance before their declared durability
boundary has accepted the complete artifact.

The provider must consume exactly `identity.archiveSize` bytes. It must reject
the publication before returning acceptance when the consumed size or SHA-256
differs from `identity.archiveSize` or `identity.archiveSha256`.

The provider must not require OpenClaw to place credentials, provider URLs,
object keys, arbitrary commands, or output paths in the managed request.
Provider-owned configuration supplies transport and credential details.

## Acceptance receipt

Successful acceptance has this logical shape:

```json
{
  "version": "continuity-publication-acceptance/v1",
  "publicationId": "publication/handoff-7",
  "identity": {
    "ownerId": "tenant/cell-1",
    "sourceRuntimeGeneration": "runtime-7",
    "handoffId": "handoff-7",
    "captureId": "capture-7",
    "archiveSha256": "<sha256>",
    "manifestSha256": "<sha256>",
    "archiveSize": 1048576
  },
  "durabilityClass": "immutable",
  "acceptedAt": "2026-07-15T00:00:00.000Z",
  "publicationPluginId": "acme-continuity",
  "publicationBindingId": "acme/continuity",
  "publicationBindingVersion": "continuity-publication-provider/v1",
  "publicationBindingGeneration": "storage-contract-2026-07"
}
```

OpenClaw validates:

- acceptance version;
- exact artifact identity equality;
- immutable durability class;
- syntactically valid RFC 3339 UTC acceptance time;
- publication identifier;
- plugin owner, provider ID, provider contract version, and provider
  generation.

Acceptance with changed, omitted, malformed, or mutable evidence fails closed.

`acceptedAt` is provider-authored diagnostic evidence. It must not be used as
the authority for replay, ordering, RPO, or lifecycle decisions. The lifecycle
coordinator records its own durable observed-acceptance time for those
decisions.

## Retrieval

Retrieval receives the immutable acceptance receipt and returns:

```ts
{
  version: "continuity-publication-retrieval/v1",
  publicationId,
  identity,
  content: AsyncIterable<Uint8Array>
}
```

OpenClaw must verify:

- retrieval version;
- publication ID equality;
- artifact identity equality;
- artifact `ownerId` equality with the requesting logical runtime's trusted
  owner identity;
- every chunk is bytes;
- exact total size;
- exact archive SHA-256;
- archive structure and manifest validity;
- exact manifest SHA-256.

Archive and manifest digests must also match the capture-time identity stored
in the continuity or lifecycle authority independently of the provider
receipt. A provider-authored receipt is not the restore integrity trust anchor.

Retrieval success is not inferred from provider return alone. The complete
stream must be consumed and verified.

## Managed publication operation

The v1 adapter operation is:

```text
openclaw backup publish --managed --json
```

It reads one bounded strict JSON request from stdin. Unknown and missing fields
are rejected.

The request freezes:

- source owner, generation, handoff, capture, and execution-incarnation IDs;
- normalized absolute local archive path;
- archive digest, size, and manifest digest;
- provider plugin ID, provider ID, contract version, and stable generation.

The operation:

1. copies the source archive to a private pinned path;
2. verifies size, archive digest, archive structure, and manifest digest;
3. resolves and loads the selected external provider plugin;
4. starts its services with strict startup failure propagation;
5. publishes the pinned bytes;
6. stops all source-session provider services with strict failure propagation;
7. starts a separate retrieval process with no inherited provider registry,
   module state, handles, or file descriptors, then resolves a fresh scoped
   registry and provider binding;
8. rejects changed ownership, version, or generation;
9. retrieves and completely verifies the accepted artifact;
10. stops destination-session services;
11. removes private temporary files;
12. emits one typed JSON result.

The Gateway remains absent. The operation does not activate restored state,
open admission, or decide `safeToDestroy`.

The operation must enforce declared publish, retrieval, and total deadlines,
plus bounded temporary-disk and memory budgets. Providers must honor
cancellation within a declared budget. Deadline expiry after dispatch is
`outcome-unknown`; it must not be reported as success or definite non-commit.
Retries use bounded attempts and backoff and never outlive the lifecycle
coordinator's source-authority deadline.

Provider services must be scoped to the managed operation and terminated on
normal return, handled failure, cancellation, and parent-process exit. A hard
process or host failure is recovered through durable replay or quarantine, not
by assuming service cleanup completed.

## Failure model

Core binding and evidence failures:

| Code | Meaning |
| --- | --- |
| `provider-not-found` | No enabled manifest owner or no matching runtime registration. |
| `provider-ambiguous` | Multiple enabled manifest owners or runtime registrations. |
| `provider-incompatible` | Provider contract version differs. |
| `provider-provenance-mismatch` | Plugin owner differs from the frozen binding. |
| `stale-provider-generation` | Provider generation differs from the frozen binding. |
| `invalid-acceptance` | Acceptance is malformed or does not match the artifact. |
| `invalid-retrieval` | Retrieval metadata or verified archive is invalid. |
| `resource-exhausted` | A declared memory, stream, or temporary-disk budget was exceeded before a valid result. |

Provider operation failures:

| Code | Required disposition |
| --- | --- |
| `retryable-before-commit` | Retry the same publication while source authority is retained. |
| `outcome-unknown` | Hold for exact replay and reconciliation. |
| `conflict` | Quarantine. |
| `corrupt-retrieval` | Quarantine. |
| `unavailable` | Hold or retry the same publication. |
| `cancelled` | Hold unless the lifecycle owner proves no commit occurred. |

Changed plugin ownership, incompatible version, stale generation, changed
artifact identity, or corrupt retrieval must never fall back to another
provider.

An unavailable, timed-out, or outcome-unknown retrieval after valid acceptance
holds for exact retry; it does not by itself prove corruption or quarantine the
artifact. `corrupt-retrieval` requires a completed retrieval whose metadata,
size, archive digest, archive structure, or manifest digest contradicts the
accepted and independently recorded capture identity.

## Replay and compatibility

- Exact publication replay must be idempotent.
- A receipt remains usable only while plugin ownership, provider ID, provider
  version, and provider generation match.
- Process-local registry and service generations are not persisted authority.
- A new plugin package can replace an old package only if it preserves the
  frozen provider binding and retrieval semantics.
- Provider migration and multi-generation compatibility are outside v1.

## Security requirements

- The managed request must not contain credentials or arbitrary provider URLs.
- Provider configuration remains plugin-owned and follows ordinary secret
  handling.
- Manifest ownership is resolved before runtime import.
- Only the selected owner plugin is loaded.
- The publication stream is bounded by `identity.archiveSize`; retrieval is
  bounded by the accepted `archiveSize`.
- Temporary files are private and removed on success or failure.
- Source archive mutation after pinning cannot change published bytes.
- Provider acceptance is not trusted without complete local validation.

## Conformance

### Provider conformance

A conforming provider must prove:

- valid manifest declaration and runtime registration;
- exact replay returns the same acceptance;
- conflicting identity fails closed;
- at least two independently started processes of the same provider package
  report the same provider compatibility generation;
- retrieval after the publishing process exits;
- exact byte, size, archive digest, and manifest digest preservation;
- structured unavailable, unknown-outcome, conflict, and corruption failures.

### OpenClaw conformance

OpenClaw must prove:

- missing and ambiguous manifest ownership fail before import;
- disabled declarations do not create false ambiguity;
- only the selected plugin loads;
- changed runtime plugin ownership quarantines;
- an initially unsupported provider contract version fails validation, and a
  changed version after binding quarantines;
- stale provider generation quarantines;
- duplicate runtime registration of one provider ID fails closed;
- malformed acceptance and retrieval fail closed;
- publication services stop before fresh retrieval begins;
- service startup, rollback, and shutdown errors cannot produce success;
- process A can publish and exit, then independently started process B can
  retrieve from the same external provider store without inherited memory,
  module state, handles, or file descriptors.

### Lifecycle conformance

The outer coordinator must prove:

- source destruction is not authorized before immutable acceptance;
- unknown publication outcome retains exact replay authority;
- process replacement does not change provider binding;
- retrieval and restore use the accepted artifact identity;
- retrieval and restore reject a receipt for another trusted owner/tenant cell;
- capture-time digests in lifecycle authority, not provider receipt alone,
  anchor restore integrity;
- `safeToDestroy` applies only to the source compute generation.

## Future versions

Later specifications may add:

- explicit provider-generation migration;
- multiple durability classes;
- retention and garbage-collection APIs;
- resumable multipart transfer;
- encrypted artifact envelope metadata;
- richer diagnostics and provider health;
- composition-layer installation and configuration UX.

These additions must preserve the v1 authority split and must not make Hosted
Integration or a bundled product extension a prerequisite for Portable
publication.
