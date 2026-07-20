# Restored Startup v2 Specification

This document is the implementer-facing restored-startup specification for RFC
0021, Runtime State Continuity. It defines the private adapter-to-Gateway
descriptor, the OpenClaw-authored durable completion record, the structured
startup result, and the exact admission transaction that turns a committed
restore into a ready runtime.

The [State CAPE v1 Specification](state-cape-v1-spec.md) defines the Portable
and Elastic guarantees. The
[Elastic Host Lifecycle v1 Specification](elastic-host-lifecycle-v1-spec.md)
defines the host activation operation. This sidecar defines the OpenClaw
startup boundary inside that operation.

Status: draft, tied to RFC 0021.

## Scope

Restored Startup v2 defines:

- `continuity-restored-startup/v2`;
- `continuity-restored-startup-result/v2`;
- `continuity-restore-complete/v2`;
- exact evidence binding from accepted recovery point through committed
  restore;
- scheduler, owner, and Gateway readiness ordering;
- durable replay and conflict behavior; and
- exact restored-admission consumption.

It does not define capture, publication-provider APIs, host wake-request APIs,
artifact retrieval, destination preparation, or restore execution. Those
contracts produce the evidence consumed here.

## Ownership

OpenClaw owns:

- validation of restored-startup evidence;
- scheduler reconciliation;
- required owner-readiness evaluation;
- generic Gateway readiness;
- the durable completion record;
- the canonical readiness generation; and
- the decision to open restored admission.

The adapter owns:

- the private descriptor file and journal path;
- exact transport of committed restore evidence;
- process startup and result capture;
- same-incarnation retry boundaries; and
- rejection of missing, malformed, duplicate, late, or contradictory results.

The lifecycle host owns destination-generation authority and may consume the
successful result only through the exact durable restore hold. It must not
author, modify, or synthesize OpenClaw readiness.

## Versioned contracts

| Contract | Version | Purpose |
| --- | --- | --- |
| Restored-startup input and private descriptor | `continuity-restored-startup/v2` | Bind one restored Gateway start to exact completion evidence. |
| Structured startup result | `continuity-restored-startup-result/v2` | Report one typed failure or one exact ready result before public readiness. |
| Durable completion record | `continuity-restore-complete/v2` | Bind restore, scheduler, owner readiness, admission, and readiness generation. |

Unknown versions and unknown fields fail closed. V1 records are not upgraded in
place and cannot authorize v2 admission.

## Generation roles

The following generations are independent:

- `destinationRuntimeGeneration` identifies the newly granted runtime
  generation being restored and admitted.
- `lifecycleOwnerGeneration` identifies the existing lifecycle authority that
  authorized preparation and restore.
- `schedulerGeneration` identifies the reconciled OpenClaw wake descriptor.
- `readinessGeneration` identifies the canonical completed restored-startup
  evidence.
- the Gateway incarnation identity identifies the concrete child process.

No field may substitute for another. In particular, restore execution may be
authorized by the lifecycle owner generation while producing a different new
destination runtime generation.

## Restored-startup input

The existing Gateway start request may carry a restored-only input:

```json
{
  "version": "continuity-restored-startup/v2",
  "evidence": {
    "startupMode": "restored",
    "operationId": "complete/runtime-generation-19",
    "ownerId": "sha256:aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa",
    "destinationRuntimeGeneration": "runtime-generation-19",
    "lifecycleOwnerGeneration": "continuity-lifecycle-1",
    "acceptedRecoveryPoint": {
      "recoveryPointId": "recovery-point-42",
      "publicationIdentity": "publication/handoff-42",
      "manifestSha256": "bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb"
    },
    "preparationIdentity": "preparation/runtime-generation-19",
    "admissionIdentity": "admission/runtime-generation-19",
    "expectedPlanId": "cccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccc",
    "continuityObligations": {},
    "restore": {
      "version": "continuity-restore-execution-result/v1",
      "ok": true,
      "ownerGeneration": "continuity-lifecycle-1",
      "restoreIdentity": "restore/runtime-generation-19",
      "planId": "cccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccc",
      "receiptIdentity": "sha256:dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd",
      "committedRecordIdentity": "sha256:eeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee"
    }
  }
}
```

The evidence must bind:

- restored startup mode;
- one idempotent completion operation;
- the trusted continuity owner;
- independent destination and lifecycle owner generations;
- one accepted immutable recovery point;
- exact preparation, plan, restore, receipt, and committed-record identities;
- one runtime-minted admission identity; and
- artifact-specific dependency obligations.

The restore result's `ownerGeneration` must equal
`lifecycleOwnerGeneration`. Its plan identity must equal `expectedPlanId`.
Mismatches quarantine the exact restore attempt.

## Private descriptor transport

The adapter materializes the child descriptor:

```json
{
  "version": "continuity-restored-startup/v2",
  "journalRoot": "/absolute/adapter-owned/private/path",
  "evidence": {}
}
```

The descriptor path is supplied through
`OPENCLAW_CONTINUITY_RESTORED_STARTUP_FILE`. The file must:

- be an absolute path;
- be a regular file opened without following symlinks;
- be non-empty and no larger than 64 KiB;
- be private to the process owner on non-Windows platforms; and
- contain exactly the declared descriptor fields.

The caller cannot choose the journal root. The adapter owns that private path
and must not place secrets in the descriptor.

Ordinary startup omits the environment variable and runs without any
continuity completion side effect.

## Completion ordering

Restored startup executes:

```text
validate exact descriptor and committed restore evidence
  -> reconcile OpenClaw scheduler state
  -> resolve canonical wake descriptor
  -> evaluate required owner readiness
  -> evaluate generic Gateway readiness
  -> durably publish continuity-restore-complete/v2
  -> consume that exact record to open restored admission
  -> emit continuity-restored-startup-result/v2
  -> allow the caller to publish public ready
```

Gateway admission remains closed until exact record consumption. Process
startup, successful restore, a healthy container, scheduler construction, or a
ready probe cannot open admission early.

Required owner-readiness obligations currently include:

- reconstructed plugin runtime dependencies;
- externally resolved config secret references; and
- externally resolved auth-profile credentials.

Each owner publishes a stable evidence identity. Missing evidence is false.

## Durable completion record

OpenClaw authors:

```json
{
  "version": "continuity-restore-complete/v2",
  "ownerId": "sha256:aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa",
  "destinationRuntimeGeneration": "runtime-generation-19",
  "lifecycleOwnerGeneration": "continuity-lifecycle-1",
  "recoveryPointId": "recovery-point-42",
  "manifestSha256": "bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb",
  "preparationIdentity": "preparation/runtime-generation-19",
  "restoreIdentity": "restore/runtime-generation-19",
  "restoreReceiptIdentity": "sha256:dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd",
  "committedRecordIdentity": "sha256:eeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee",
  "planId": "cccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccc",
  "schedulerGeneration": "sha256:ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff",
  "nextRequiredAt": "2026-07-20T21:00:00.000Z",
  "reasonClass": "cron",
  "requiredOwnerReadinessDigest": "sha256:1111111111111111111111111111111111111111111111111111111111111111",
  "admissionIdentity": "admission/runtime-generation-19",
  "readinessGeneration": "sha256:2222222222222222222222222222222222222222222222222222222222222222"
}
```

`nextRequiredAt` is nullable and `reasonClass` is `cron` or `none`. `cron`
requires a non-null deadline; `none` requires a null deadline.

`requiredOwnerReadinessDigest` binds the sorted required obligations and their
owner-produced evidence. `readinessGeneration` is the SHA-256 identity of the
canonical record fields other than itself. Canonical JSON recursively sorts
object keys, retains array order, uses UTF-8 JSON scalar encoding, and emits no
insignificant whitespace.

The journal is private, atomic, durable, and write-once-or-require-exact. A
replay must return the existing exact record and readiness generation.
Contradictory evidence is a conflict, never a second winner.

## Structured result

Success is emitted before the caller's public ready marker:

```json
{
  "version": "continuity-restored-startup-result/v2",
  "ok": true,
  "phase": "ready",
  "readinessGeneration": "sha256:...",
  "replayed": false,
  "admissionOpen": true,
  "record": {}
}
```

The line prefix is:

```text
[gateway] continuity-restored-startup-result/v2
```

The result is at most 64 KiB. The adapter must accept exactly one canonical
result bound to the expected child, restore receipt, admission identity, and
readiness generation.

Failure is:

```json
{
  "version": "continuity-restored-startup-result/v2",
  "ok": false,
  "phase": "blocked",
  "code": "ContinuityReadinessFailed",
  "disposition": "hold"
}
```

Phases are `restoring`, `reconciling`, and `blocked`. Codes are:

- `ContinuityRestoreFailed`;
- `SchedulerReconciliationFailed`;
- `ContinuityReadinessFailed`; and
- `ReadinessGenerationConflict`.

Dispositions are:

- `retry-same-incarnation`;
- `hold`; and
- `quarantine`.

A failure never emits a ready marker and never opens admission.

## Admission transaction

The lifecycle restore hold transitions through:

```text
RestoreHeld
  -> RestoreCommitted
  -> RestoreStarting
  -> Runnable(restored-ready retained)
```

Opening admission consumes the exact:

- continuity owner;
- destination runtime generation;
- lifecycle owner generation;
- restore receipt;
- admission identity; and
- readiness generation.

The retained `restored-ready` result is durable. A fresh coordinator process
may replay it only after resolving the same child and establishing its own
fresh process-local proxy connection. Replay must not allocate, prepare,
restore, reconcile, or mint a second readiness generation.

Ordinary startup cannot consume a committed restore. Restored startup cannot
reuse an ordinary admission identity.

## Crash and replay behavior

The required crash boundary is after the concrete provisioner durably returns
retained `Ready` and before the host wake operation completes its own ready
transition.

At that boundary:

- the wake operation may still report `provisioning`;
- the restore hold retains the exact restored-ready result;
- the restored worker and readiness generation are fixed;
- a new coordinator process must establish a fresh proxy connection; and
- replay completes the host wake operation without repeating preparation or
  restore.

Timeout, process loss, and response loss are not proof that durable authority
was lost. A second destination generation must not begin while the first can
still be the winner.

## Security and privacy

- Descriptor and journal paths are adapter-owned and private.
- Unknown fields and malformed identities fail closed.
- Recovery artifacts never carry host-issued credentials or short-lived
  tokens.
- Readiness evidence is represented by stable identities, not secret values.
- Result parsing is bounded and accepts one exact structured record.
- Admission cannot be opened by log text, health probes, or host-authored
  readiness.
- Quarantine preserves recovery evidence and retained work for operator
  disposition.

## Compatibility and migration

V2 intentionally separates destination runtime generation from lifecycle owner
generation. Records written before that separation cannot authorize v2
admission.

Migration is forward-only:

1. deploy readers that reject unknown or incomplete v2 evidence;
2. shadow the v2 completion and result path while existing compute remains
   resident;
3. prove same-child replay and no repeated preparation/restore;
4. enable v2 writes for the exact lifecycle profile; and
5. enable Elastic destruction only after paired-image conformance.

Ordinary startup remains compatible because the restored-startup input is
optional and absent by default.

## Conformance

Conformance must prove:

- strict descriptor/result parsing and size bounds;
- private no-follow descriptor loading;
- independent destination and lifecycle generations;
- exact recovery point, plan, restore, receipt, and admission binding;
- scheduler reconciliation before readiness;
- required plugin, secret, and auth-profile readiness;
- generic Gateway readiness before completion;
- durable exact record replay;
- one result before public ready;
- no ready marker on failure;
- exact admission consumption;
- ordinary-start compatibility;
- stale child, stale generation, and contradictory replay rejection;
- crash after retained ready and before host completion;
- fresh coordinator replay to the same child and readiness generation;
- a fresh process-local proxy connection; and
- exactly one destination preparation and restore execution.
