---
title: Runtime State Continuity
authors:
  - Gio Lodi
created: 2026-07-10
last_updated: 2026-07-10
status: draft
issue:
rfc_pr: https://github.com/giodl73-repo/rfcs/pull/3
---

# Proposal: Runtime State Continuity

## Summary

Define the portable state contract required to continue an OpenClaw runtime in
another process, container, cell, or machine. The contract inventories state
ownership, defines consistency and checkpoint boundaries, separates local
materialization from durable publication, fences writers and runtime
generations, specifies ordered restore and compatibility, and exposes stable
continuity evidence.

Build on the existing `gateway.suspend.*` admission/drain model and snapshot
primitives rather than creating a parallel persistence or shutdown system.
`/synced` is the aggregate durability projection, analogous to `/ready` for
serviceability. Safe shutdown is one outcome of reaching a final synchronized
generation; neither is the whole state feature.

## Motivation

Container and managed hosts need more than readiness:

```text
readiness: may this runtime receive work?
continuity: what acknowledged state can survive replacement?
```

A host can stop traffic and terminate a process, but that does not prove that
sessions, transcripts, memory databases, plugin state, pairing state, or
workspace changes are consistent and recoverable elsewhere. A local SQLite
snapshot may be complete without having been uploaded to host storage. A
previously successful checkpoint may be stale if a later mutation was
acknowledged.

Without an upstream contract, hosts infer safe shutdown from filesystem
activity, copy directories opportunistically, add private lifecycle signals,
or claim durability based on local artifact creation. Those approaches create
data-loss races and couple deployment logic to OpenClaw's current paths and
writers.

OpenClaw already has important pieces:

- `gateway.suspend.prepare/status/resume` can stop root work admission, report
  blockers, and issue a leased suspension;
- safe restart and shutdown events exist;
- snapshot work defines SQLite-consistent artifact creation, manifests,
  integrity verification, and restore semantics.

The missing contract composes those pieces into host-visible answers to:

> What state does OpenClaw own, through which state point is this runtime
> recoverable, can that state be restored compatibly elsewhere, and may the
> host now destroy this generation?

OCC can use the same contract to move or replace cells while remaining outside
the hot-path agent traffic and host storage implementation.

## Goals

- Inventory and classify OpenClaw-owned durable, reconstructable, ephemeral,
  and secret state.
- Build on Gateway suspension for admission closure and draining.
- Define a checkpoint request/result with explicit component outcomes.
- Distinguish local consistency/materialization from host publication.
- Fence shutdown against the final mutation generation or component
  watermarks.
- Make "safe to destroy" an explicit result, not a host inference.
- Define restore ordering, compatibility, and failure semantics.
- Allow hosts to provide storage and publication without making OpenClaw know a
  specific backend.
- Preserve forced-termination behavior through the last proven durable state.
- Expose canonical continuity status suitable for Docker, Kubernetes, systemd,
  managed hosts, and OCC.
- Provide release conformance for checkpoint, restore, and shutdown races.

## Non-Goals

- Replacing readiness or Hosting Profiles.
- Requiring every deployment to use remote durable storage.
- Standardizing a particular object store, filesystem, git, Graph, or database.
- Treating caches or reconstructable data as mandatory checkpoint state.
- A distributed transaction across OpenClaw and host storage.
- Making OCC carry state artifacts or runtime event traffic.
- Expanding the snapshot proposal into a global storage coordinator.
- Assuming one global generation before OpenClaw can prove that model.
- Guaranteeing zero loss after forced termination beyond the last durable
  checkpoint.

## Proposal

### Continuity model

Runtime State Continuity is a pipeline, not a shutdown callback:

```text
state inventory and ownership
  -> mutation generation or component watermarks
  -> consistent local checkpoint
  -> durable host publication
  -> synced generation
  -> generation-fenced safe replacement
  -> ordered compatible restore
  -> readiness
```

Each stage has a distinct authority:

- OpenClaw owns state meaning, writer classification, consistency points,
  artifact manifests, restore ordering, schema compatibility, and the current
  runtime generation or component watermarks.
- A publication provider owns the claim that exact artifacts were accepted by
  the selected durability boundary.
- The host owns storage destination, encryption, retention, scheduling,
  placement, retry policy, and whether a declared durability class is
  sufficient for replacement.

The canonical status must therefore distinguish at least `dirty`,
`materialized`, `published`, `synced`, `restoring`, and `unknown` facts without
collapsing them into one success boolean. A runtime becomes dirty whenever a
required acknowledged mutation advances beyond its published generation or
component watermark. It becomes synced only when every required component is
durable through the current target.

### State classes and descriptors

OpenClaw exposes an enumerable continuity inventory. Each component descriptor
states:

- stable component ID and owner;
- state class: durable, reconstructable, ephemeral, or secret;
- required or optional continuity participation;
- consistency/checkpoint mechanism;
- restore phase and dependencies;
- schema/format compatibility identity;
- whether it supports observation only, local materialization, and/or restore;
- redaction and audit metadata.

Illustrative components include workspace, global SQLite state, per-agent
SQLite state, sessions/transcripts, plugin state, credentials/pairing, and
host-provided publication. The actual list must be derived from current state
owners; this RFC does not declare every current filesystem path durable.

Paths are implementation details unless a component contract explicitly makes
them portable. Hosts consume descriptors, artifacts, and results rather than
copying undocumented directories.

The minimum descriptor is machine-readable:

```ts
type ContinuityComponentDescriptor = {
  id: string;
  owner: "core" | `plugin.${string}`;
  stateClass: "durable" | "reconstructable" | "ephemeral" | "secret";
  required: boolean;
  writerIds: string[];
  consistencyMechanism: string;
  checkpointCapability: "none" | "observe" | "materialize";
  restorePhase?: number;
  restoreDependsOn?: string[];
  formatId?: string;
  schemaVersion?: string;
  compatibilityRange?: string;
  redactionClass: "public" | "operator" | "secret";
};
```

Registration rejects duplicate IDs, dependency cycles, required durable
components without classified writers, and restorable components without
format/schema identity. Plugin descriptors are activation-scoped and
namespaced; a plugin cannot claim a core component or another plugin's state.
The release conformance inventory must account for every OpenClaw-owned writer,
including an explicit reconstructable/ephemeral exclusion where appropriate.

### Canonical continuity status

Continuity lives in runtime status:

```json
{
  "continuity": {
    "state": "pending",
    "target": {
      "kind": "watermarks",
      "components": {
        "workspace": "42",
        "global-state": "105"
      }
    },
    "components": [
      {
        "id": "global-state",
        "required": true,
        "state": "materialized",
        "sourceWatermark": "105",
        "artifactId": "snapshot-abc"
      },
      {
        "id": "host-publication",
        "required": true,
        "state": "pending",
        "sourceWatermark": "105"
      }
    ]
  }
}
```

Candidate aggregate states are `current`, `pending`, `syncing`, `degraded`, and
`unknown`. Exact names should align with existing status condition conventions.

The model supports either:

- one runtime generation when all acknowledged mutations can participate in a
  defensible global ordering; or
- per-component watermarks when state owners cannot share one generation.

The first implementation must not fabricate a global counter that excludes
acknowledged writers.

### Suspension and final checkpoint

Safe shutdown begins with the existing suspension contract:

```text
gateway.suspend.prepare
  -> close root work admission
  -> report active blockers or issue suspension lease
  -> host observes ready-to-checkpoint suspension
```

Once suspended, a new checkpoint operation:

1. verifies the suspension lease is active;
2. captures the target generation or component watermarks;
3. asks each required component to reach a consistency point;
4. materializes portable artifacts where supported;
5. returns a manifest of component results and source watermarks;
6. verifies no unaccounted required mutation advanced beyond the target;
7. optionally waits for required host-publication acknowledgement;
8. returns whether the runtime is safe to destroy.

Illustrative result:

```json
{
  "checkpointId": "checkpoint-123",
  "suspensionId": "suspension-456",
  "state": "published",
  "safeToDestroy": true,
  "target": {
    "kind": "watermarks",
    "components": {
      "workspace": "42",
      "global-state": "105"
    }
  },
  "components": [
    {
      "id": "global-state",
      "required": true,
      "state": "published",
      "artifactId": "snapshot-abc",
      "watermark": "105"
    }
  ]
}
```

`safeToDestroy` is true only when:

- admission remains closed under the same suspension;
- active work and required mutation sources are drained;
- every required component is consistent through its target;
- every required durable artifact has been acknowledged by its required
  durability boundary;
- the final generation/watermark verification still matches.

If the host cannot complete checkpoint/publication, it may resume the same
suspension using `gateway.suspend.resume` or force termination with an explicit
last-durable-state result.

### Local materialization versus host publication

The contract separates two facts:

```text
OpenClaw materialized
  a consistent local artifact exists and verifies through watermark N

Host published
  the host has durably stored the artifact set through watermark N
```

OpenClaw is authoritative for local consistency and artifact manifests. The
host/provider is authoritative for remote publication. A completed local
snapshot never implies remote durability by itself.

A publication provider may consume artifact manifests and return a bounded
acknowledgement containing checkpoint ID, artifact IDs/digests, source
watermarks, storage receipt identity, and publication status. ClawBus, the
hosted duplex form of the OpenClaw protocol, may carry that canonical
request/result or a later typed stream on the host's existing session. Artifact
meaning, durability acknowledgement, replay rules, and safe-to-destroy
semantics remain owned by this continuity contract rather than ClawBus.

Hosts may also choose a local-only durability profile. In that case,
`safeToDestroy` means safe for the declared local persistence boundary, not safe
to replace the underlying volume. The checkpoint result must identify the
durability class so operators cannot confuse the guarantees.

### Snapshot integration

SQLite-safe snapshot providers remain independently useful. A snapshot
component contributes:

- source database identity and schema metadata;
- source generation/watermark where available;
- completed artifact path/handle and digest;
- manifest and verification result;
- restore compatibility and ordering metadata.

Continuity coordinates required components and final shutdown fencing; it does
not redefine SQLite snapshot mechanics.

### Restore

Restore is explicit and ordered:

1. verify checkpoint manifest and artifact integrity;
2. verify OpenClaw and component schema compatibility;
3. restore required global/identity state before dependent agent/plugin state;
4. restore workspaces and component artifacts according to dependencies;
5. run component validation/migrations;
6. start Gateway with admission closed;
7. report restored checkpoint/watermarks through status;
8. evaluate readiness before accepting work.

Restore failures use structured component, reason, retryability, and operator
action fields. OpenClaw must not silently start with an incomplete required
restore unless the selected continuity policy explicitly allows a clean start.

### Mutation and acknowledgement rules

A user-visible success that promises durable state must advance or record the
relevant continuity watermark before it is acknowledged. Operations that are
explicitly ephemeral or reconstructable need not participate.

Every required state writer must be one of:

- represented in the global generation;
- represented by a component watermark;
- drained before target capture;
- excluded by an explicit reconstructable/ephemeral classification.

The implementation cannot claim aggregate `current` or `safeToDestroy` while a
required writer is unclassified.

Every checkpoint, publication receipt, restore result, and safe-to-destroy
decision is bound to `{ runtimeId, runtimeGeneration, checkpointId }` plus the
component watermark set and artifact-manifest digest. Equality of a friendly
deployment name is never sufficient fencing. A provider result for an older
runtime generation cannot advance continuity status for a newer process, even
if artifact paths or component IDs are identical.

Restore establishes a new runtime generation and records the source checkpoint
separately. The restored runtime must not reuse the source writer lease or
publish under the source generation. Admission remains closed until required
component validation and any allowed migrations complete.

### `/synced` projection

The canonical model is continuity status and checkpoint results. `/synced` is
a cheap host-facing projection over that model:

```text
synced = every required component is durable through the current target
```

It does not mean ready, drained, or safe to destroy. A runtime may be ready but
dirty, or not ready but synced. Safe shutdown remains:

```text
suspended + drained + synchronized through final target + publication ack
```

`/synced` must fail closed as unknown until the implementation can account for
every required writer through a defensible generation or component-watermark
model. It must not infer success from quiet filesystem activity or the presence
of an older checkpoint.

### Failure model

At minimum, checkpoint and restore distinguish:

- active-work blocker;
- suspension conflict or expiry;
- writer failed to quiesce;
- component timeout or cancellation;
- snapshot/materialization failure;
- integrity verification failure;
- publication unavailable, rejected, or timed out;
- quota/storage exhaustion;
- stale generation or watermark advanced;
- incompatible schema/artifact;
- missing required component;
- restore ordering/migration failure;
- forced termination through an older durable target.

Failures are bounded, machine-readable, and identify whether resume/retry is
safe.

### Conformance

Release tests should prove:

- suspension blocks new root work and reports blockers;
- concurrent mutation cannot produce a false safe-to-destroy result;
- required writers are represented in generation/watermark accounting;
- SQLite and other artifacts verify and restore;
- local materialization is not reported as host publication;
- publication acknowledgements bind exact artifact digests and watermarks;
- stale-generation provider results cannot complete a newer checkpoint;
- timeout/cancellation leaves the runtime resumable or reports otherwise;
- forced termination reports the actual last durable target;
- restored runtime reports its source checkpoint before becoming ready;
- compatibility behavior across supported OpenClaw releases.
- complete descriptor accounting for every required writer;
- dependency-cycle and unclassified-writer rejection;
- restore into a new runtime generation without source-lease reuse;
- duplicate and replayed publication receipt handling;
- redaction of secret component identities and artifact metadata.

### Host persistence migration and deletion gate

A host may remove private persistence coordination only after the component
inventory accounts for every path it currently copies, ignores, uploads, or
restores. Migration proof must compare the old host artifact set with the
OpenClaw manifest, execute drain/checkpoint/publication/restore through the
reported generation, and inject mutation, timeout, stale-writer, corruption,
quota, and incompatible-restore failures. Once that proof passes on the minimum
supported OpenClaw release, the host deletes path discovery, opportunistic copy
logic, private checkpoint/shutdown frames, and duplicate restore ordering.
Temporary dual publication must be read-only for comparison, observable, owned,
and time-bounded.

### Implementation sequence

1. Publish the state component inventory, writer classification, restore
   dependencies, compatibility identities, and canonical continuity status.
2. Establish an honest mutation-generation or component-watermark model and
   expose dirty/synced conditions, including a fail-closed `/synced`
   projection.
3. Add checkpoint orchestration for local materialization and manifest results
   using existing snapshot providers.
4. Add host publication acknowledgement, stale-writer fencing, and an explicit
   generation-bound safe-to-destroy result over `gateway.suspend.*`.
5. Prove ordered restore, migration/compatibility failures, rollback limits,
   and readiness only after required restored state validates.
6. Add release conformance for mutation races, publication races, replacement,
   restore, upgrade, rollback, and forced termination through the last known
   durable generation.

## Rationale

### Why not use readiness?

Readiness answers whether work may enter. A runtime can be unready while still
mutating or while required state is only local. Combining the two would make
both signals ambiguous.

### Why build on suspension?

`gateway.suspend.*` already owns admission closure, blockers, leases, conflicts,
and resume. A separate drain API would duplicate lifecycle state and introduce
races between two authorities.

### Why not rely on filesystem clean/dirty state?

Filesystem activity does not identify acknowledged semantic mutations,
multi-file consistency, SQLite transaction state, remote durability, or restore
compatibility.

### Why distinguish materialized and published?

Only OpenClaw can authoritatively create and verify its local artifacts. Only
the host can authoritatively claim that remote storage accepted them. Merging
the claims would overstate durability.

### Why permit component watermarks?

A global generation is attractive but invalid if any required writer cannot
participate. Component watermarks allow an honest first implementation and can
later collapse into a global generation if the evidence supports it.

### Why keep storage backends outside core?

Storage, encryption, routing, and retry policy are host responsibilities. Core
owns state meaning, consistency, manifests, restore semantics, and the
conditions under which a durability acknowledgement is accepted.

## Unresolved questions

- What is the complete current inventory of durable and reconstructable
  OpenClaw state?
- Can current acknowledged mutations support one global generation, or are
  component watermarks required indefinitely?
- Which state components must participate in the first supported continuity
  profile?
- What exact operation names should extend `gateway.suspend.*`?
- How should a publication provider authenticate receipts and bind them to a
  hosted generation?
- Which snapshot manifest fields need source watermark and compatibility data?
- What is the minimal local-only durability guarantee for Docker volumes?
- Should credentials and pairing state be restored as artifacts, reprovisioned,
  or explicitly excluded by profile?
- Which restore failures permit an operator-approved clean start?
- Which continuity details may be redacted from unauthenticated `/synced`
  callers while preserving a useful boolean host probe?
