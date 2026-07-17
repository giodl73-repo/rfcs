# Elastic Host Lifecycle v1 Specification

This document is the implementer-facing host lifecycle specification for RFC
0021, Runtime State Continuity. It defines the minimal contract by which a host
hibernates an OpenClaw runtime, retains wake causes while compute is absent,
and restores one fenced runtime generation before delivering work.

The [State CAPE v1 Specification](state-cape-v1-spec.md) defines the Elastic
guarantee. The
[Portable Publication Provider v1 Specification](portable-publication-provider-v1-spec.md)
defines immutable publication and retrieval. This sidecar defines the host
orchestration boundary above those contracts.

Status: draft, tied to RFC 0021.

## Thesis

An Elastic host needs one activation path, not knowledge of OpenClaw state
formats:

```text
EnsureRuntimeReady(runtime, wake causes)
  -> allocate one fenced generation
  -> retrieve one accepted recovery point
  -> restore and validate
  -> reconcile OpenClaw scheduler semantics
  -> publish readiness
  -> admit and deliver retained work
```

Teams, host/API work, semantic deadlines, and authenticated operator requests
may all call that path. They remain independently durable and deduplicated even
when their compute provisioning coalesces.

## Scope

This specification defines:

- planned Elastic hibernation;
- final recovery-point and wake-registration atomicity;
- revocable sleep authorization;
- generation-bound destruction authorization;
- retained wake causes;
- semantic scheduler deadline publication;
- one idempotent `EnsureRuntimeReady` operation;
- restored-readiness gating before delivery;
- host lifecycle status, failure, quarantine, replay, and conformance.

This specification does not define:

- OpenClaw state stores, cron schemas, or Channel payloads;
- publication-provider implementation or credentials;
- one host transport, HTTP route, queue, database, or compute platform;
- host placement, pricing, idle threshold, or retention policy;
- a normalized ingress method;
- Channel acknowledgement or delivery semantics;
- cron due, catch-up, or duplicate-suppression policy;
- traffic routing outside readiness and generation fencing;
- automatic disaster reset.

The JSON objects below are the canonical V1 data model. A binding may map them
onto HTTP, RPC, queues, or an in-process port, but it must preserve the named
fields and semantics. Mutation inputs reject unknown fields.

## Ownership

OpenClaw owns:

- state meaning, capture, restore, compatibility, and migration;
- cron definitions, due decisions, catch-up, and duplicate suppression;
- the earliest semantic `nextRequiredAt` value;
- scheduler reconciliation;
- continuity restored-readiness truth;
- Gateway work admission.

The host owns:

- idle and cost policy;
- durable lifecycle and generation authority;
- accepted recovery-point references while compute is absent;
- retained ingress and wake-cause records;
- sleep authorization and revocation;
- compute provisioning and placement;
- cold-start lead time;
- retry scheduling and terminal host failure;
- the durable `SafeToDestroy` conclusion.

Channel and API owners retain their existing acknowledgement, payload,
deduplication, and delivery contracts. They do not become continuity owners.

## Minimal host surface

The complete V1 host surface consists of three logical operations:

```text
PrepareHibernate
EnsureRuntimeReady
InspectElasticLifecycle
```

`PrepareHibernate` is the only planned path from a running generation to absent
compute. `EnsureRuntimeReady` is the only activation path used by retained
ingress, semantic deadlines, host/API work, and operators.
`InspectElasticLifecycle` is read-only.

Implementations may expose asynchronous operation handles, but they must not
require callers to orchestrate capture, publication, retrieval, restore,
scheduler reconciliation, or readiness as separate host-facing operations.

## Identity model

Every mutation is scoped to one tenant cell and continuity lineage. The
contract carries:

- `runtimeId`: stable logical runtime identity;
- `continuityEpoch`: lineage reset boundary;
- `sourceRuntimeGeneration`: generation being hibernated;
- `handoffId`: stable planned-handoff operation identity;
- `recoveryPointId`: accepted immutable recovery-point identity;
- `manifestDigest`: capture-time manifest identity;
- `sleepAuthorizationId`: revocable authorization identity;
- `sleepAuthorizationRevision`: monotonic lifecycle revision;
- `wakeRegistrationId`: accepted semantic wake snapshot identity;
- `schedulerGeneration`: OpenClaw scheduler snapshot generation;
- `wakeRequestId`: idempotent activation request identity;
- `destinationRuntimeGeneration`: newly granted restore generation.

A friendly deployment, container, worker, or user name is never sufficient
authority. Stale epochs, generations, revisions, recovery points, or operation
identities fail closed.

## Wake registration

OpenClaw publishes one semantic wake snapshot:

```json
{
  "version": "continuity-wake-registration/v1",
  "runtimeId": "runtime/tenant-cell",
  "continuityEpoch": "epoch-7",
  "handoffId": "handoff-42",
  "recoveryPointId": "recovery-point-42",
  "manifestDigest": "sha256:...",
  "wakeRegistrationId": "wake-registration-42",
  "schedulerGeneration": "scheduler-19",
  "nextRequiredAt": "2026-07-17T03:00:00Z",
  "reasonClass": "cron"
}
```

`nextRequiredAt` is nullable when OpenClaw has no semantic deadline. The reason
class is bounded metadata such as `cron`, `internal-maintenance`, or `none`;
it does not contain job names, prompts, payloads, or tenant content.

OpenClaw may expose an advisory running snapshot for Status and host idle
planning. Only the final wake registration accepted with the final recovery
point is authoritative while compute is absent.

The authoritative registration is returned through `PrepareHibernate`; it is
not independently committed by a scheduler callback or inferred from Status.
No continuous host callback is required for every running cron mutation.

The final registration must be derived from the same closed scheduler state
included in the final recovery point. A scheduler mutation accepted before
admission closes must be reflected in both. A scheduler mutation rejected or
retained after admission closes must not silently change the accepted
registration.

The host may compute:

```text
hostWakeAt = nextRequiredAt - configuredColdStartLead
```

The host must not reinterpret cron expressions, decide whether a job is due,
invent catch-up work, or suppress duplicates.

## PrepareHibernate

The logical request contains:

```json
{
  "version": "continuity-prepare-hibernate/v1",
  "runtimeId": "runtime/tenant-cell",
  "continuityEpoch": "epoch-7",
  "sourceRuntimeGeneration": "runtime-generation-18",
  "handoffId": "handoff-42",
  "reason": "idle-policy",
  "deadline": "2026-07-16T23:30:00Z"
}
```

The host must authenticate and authorize the runtime owner and exact source
generation before dispatch.

`PrepareHibernate` performs one durable operation:

1. establish retained-ingress authority before sleep can be authorized;
2. close root-work admission;
3. drain or fence active work and writable semantic owners;
4. cleanly stop the Gateway and state owners;
5. capture the final closed state;
6. publish and independently retrieve-verify the immutable recovery point;
7. derive the final semantic wake registration from the captured scheduler
   state;
8. atomically accept the recovery point, wake registration, and sleep
   authorization;
9. strictly finalize the source and durably derive `SafeToDestroy`.

The result is asynchronous lifecycle state, not an OpenClaw-authored boolean.
The host may remove source compute only when its durable authority contains a
generation-bound destruction authorization:

```json
{
  "version": "continuity-destruction-authorization/v1",
  "runtimeId": "runtime/tenant-cell",
  "continuityEpoch": "epoch-7",
  "sourceRuntimeGeneration": "runtime-generation-18",
  "handoffId": "handoff-42",
  "recoveryPointId": "recovery-point-42",
  "manifestDigest": "sha256:...",
  "wakeRegistrationId": "wake-registration-42",
  "schedulerGeneration": "scheduler-19",
  "sleepAuthorizationId": "sleep-42",
  "sleepAuthorizationRevision": 9,
  "safeToDestroy": true
}
```

`SafeToDestroy` authorizes removal of only that source compute generation. It
does not authorize persistent tenant-data deletion, recovery-point deletion,
lineage reset, another worker, or another tenant cell.

## Sleep revocation

Sleep authorization is valid only at one durable lifecycle revision.

Before physical removal, the host must atomically consume the exact
authorization revision. Retained-ingress revocation and destruction consumption
compete at that durable linearization point.

Any newly retained wake cause must atomically:

1. persist its owner-defined payload or reference and deduplication identity;
2. revoke the current sleep authorization or prove it was already revoked;
3. create or join an idempotent wake request.

If revocation wins before source destruction, the host must not act on the
older destruction authorization. It either resumes the still-valid source
generation through an explicitly supported path or provisions a replacement
through `EnsureRuntimeReady`.

If source destruction wins first, the retained wake cause remains durable and
continues through `EnsureRuntimeReady`.

No absent OpenClaw process is expected to observe or resolve this race.

## Wake causes

Each wake cause has:

- stable owner and cause type;
- durable owner-defined payload or opaque reference;
- stable deduplication identity;
- accepted-at time;
- delivery state;
- retry state;
- terminal failure state;
- continuity epoch;
- wake request association.

Initial reason classes are:

```text
retained-ingress
host-api
semantic-deadline
operator
health-replacement
```

Teams is a representative retained-ingress owner. It durably stores the
message before acknowledging the upstream delivery, revokes sleep, calls
`EnsureRuntimeReady`, waits for readiness to the current generation, and then
uses its existing generation-fenced delivery path.

Wake causes may share one provisioning attempt. Coalescing compute must not
merge, discard, acknowledge, or change the retry state of individual work.

## EnsureRuntimeReady

The logical request contains:

```json
{
  "version": "continuity-ensure-runtime-ready/v1",
  "runtimeId": "runtime/tenant-cell",
  "continuityEpoch": "epoch-7",
  "wakeRequestId": "wake-93",
  "wakeCauseIds": ["teams-message-771", "deadline-wake-registration-42"],
  "reasonClasses": ["retained-ingress", "semantic-deadline"],
  "requestedAt": "2026-07-17T02:58:00Z"
}
```

The request does not carry Channel payloads, provider credentials, artifact
locations, cron definitions, or caller-selected recovery points.
Wake-cause identities must already exist in host authority; callers cannot
manufacture a reason class to bypass retention or authorization.

The host authority selects the newest policy-allowed accepted recovery point
and performs:

```text
revoke sleep authorization
  -> coalesce or start provisioning
  -> grant one destination runtime generation
  -> allocate fresh compute
  -> acquire the restore hold
  -> retrieve the accepted recovery point
  -> materialize, migrate, and reconstruct dependencies
  -> restore OpenClaw scheduler state
  -> reconcile missed and due schedules
  -> durably queue policy-allowed catch-up work
  -> publish ContinuityRestoreComplete
  -> satisfy required owner readiness
  -> open Gateway admission
  -> return the ready generation
```

Only after the operation returns a ready current generation may wake-cause
owners deliver retained work.

The successful logical result contains:

```json
{
  "version": "continuity-ensure-runtime-ready-result/v1",
  "ok": true,
  "runtimeId": "runtime/tenant-cell",
  "continuityEpoch": "epoch-7",
  "wakeRequestId": "wake-93",
  "destinationRuntimeGeneration": "runtime-generation-19",
  "recoveryPointId": "recovery-point-42",
  "readinessGeneration": "readiness-19"
}
```

The result does not mean every retained activity was delivered. Each owner
advances its own delivery and acknowledgement state.

## Idempotency and coalescing

The same `wakeRequestId` and identity must return the same operation or terminal
result. Reuse with different identity is a conflict.

Concurrent wake requests for one runtime and epoch may join one provisioning
operation. The host records every joined request and reason class. Only one
destination generation may hold restore or admission authority.

A failed or unknown provisioning attempt must not cause a second generation to
start until durable authority proves the first attempt lost authority or was
terminated. Timeout is not proof of failure.

## Scheduler reconciliation

The host alarm is only a provisioning trigger. After restore, OpenClaw:

1. loads the restored scheduler definitions and durable execution state;
2. compares them with the current time;
3. applies OpenClaw-owned missed-run and catch-up policy;
4. suppresses duplicates using scheduler-owned identities;
5. durably queues accepted catch-up work;
6. publishes a new scheduler generation and next semantic deadline;
7. completes `ContinuityRestoreComplete`.

Gateway admission remains closed until reconciliation completes. The host must
not directly invoke restored cron jobs to reduce cold-start latency.

## InspectElasticLifecycle

The read-only snapshot exposes bounded, redacted state:

```json
{
  "version": "continuity-elastic-lifecycle-snapshot/v1",
  "runtimeId": "runtime/tenant-cell",
  "continuityEpoch": "epoch-7",
  "capeDesired": "elastic",
  "capeEffective": "elastic",
  "phase": "absent",
  "sourceRuntimeGeneration": "runtime-generation-18",
  "currentRuntimeGeneration": null,
  "recoveryPointId": "recovery-point-42",
  "recoveryPointAgeMs": 42000,
  "sleepAuthorization": "consumed",
  "safeToDestroy": true,
  "nextRequiredAt": "2026-07-17T03:00:00Z",
  "hostWakeAt": "2026-07-17T02:58:00Z",
  "retainedWakeCauseCount": 0,
  "activeWakeRequestId": null,
  "lastFailureCode": null,
  "quarantined": false
}
```

Stable phases include:

```text
running
hibernating
safe-to-destroy
absent
wake-requested
provisioning
restoring
reconciling
ready
held
quarantined
```

The snapshot is not lifecycle authority. Consumers must use the mutation
operations and their exact identities rather than act on a cached projection.

## Failure model

Stable failure classes include:

```text
ContinuityHibernateBlocked
ContinuitySleepRevoked
ContinuityFinalRecoveryPointFailed
ContinuitySourceFinalizationFailed
ContinuityWakeAuthorityUnavailable
ContinuityWakeRequestConflict
ContinuityProvisioningFailed
ContinuityGenerationAuthorityConflict
ContinuityRestoreFailed
ContinuitySchedulerReconciliationFailed
ContinuityReadinessFailed
ContinuityRetainedDeliveryExhausted
ContinuityLifecycleQuarantined
ContinuityLifecycleStatusUnknown
```

Failures state whether retry is safe for the same operation identity. They do
not expose credentials, artifact locations, Teams messages, prompts, cron
definitions, or raw provider errors.

Retained work remains retained when provisioning, restore, reconciliation, or
readiness fails. A caller-visible timeout does not acknowledge or discard it.

## Quarantine

Contradictory identity, stale authority, corrupt recovery evidence, unsafe
replay, or irreconcilable restore state enters durable quarantine.

While quarantined:

- no source destruction authorization may be newly issued;
- no destination generation may be granted;
- `EnsureRuntimeReady` cannot provision or admit work;
- retained work and semantic deadlines remain preserved;
- recovery points and diagnostic evidence remain preserved;
- implicit clean start and disaster reset are forbidden.

Exit requires authenticated, authorized, and audited reconciliation or an
explicit audited disaster reset permitted by policy.

## Security and privacy

- Every mutation authenticates the runtime owner and tenant cell.
- Generation and revision checks are mandatory and fail closed.
- Wake callers cannot choose artifact locations, recovery points, or
  destination generations.
- Retained payloads remain with their semantic owner.
- Lifecycle snapshots and audit events expose only bounded redacted metadata.
- Host credentials are re-resolved at destination and never stored in
  continuity artifacts or wake registrations.
- `SafeToDestroy` cannot be supplied by OpenClaw, a publication provider,
  readiness, or a caller.
- Readiness cannot be supplied by the host.

## Compatibility and migration

V1 is additive. A host may shadow existing Teams warmup and cron wake paths,
but exactly one authority may provision compute or deliver work for a runtime
generation.

Migration must prove:

1. old and new paths observe the same retained Teams activities;
2. old and new paths compute equivalent host wake timing from OpenClaw's
   semantic deadline;
3. only the selected path performs provisioning and delivery;
4. rollback preserves retained work and accepted recovery points;
5. stale private wake records cannot grant a generation;
6. private duplicate restore and cron wake orchestration is deleted only after
   conformance.

Unknown fields are rejected on mutation contracts. New reason classes and
snapshot fields require version-compatible evolution.

## Conformance

Elastic Host Lifecycle v1 conformance must prove:

- planned hibernation from a valid Portable generation;
- final recovery-point, wake-registration, and sleep-authorization atomicity;
- no destruction before generation-bound `SafeToDestroy`;
- a retained Teams activity racing hibernation revokes sleep without loss;
- destruction winning the race still leads to successful retained delivery;
- a semantic deadline wakes with no resident OpenClaw process;
- host cold-start lead changes wake timing but not cron semantics;
- a scheduler mutation during hibernation is either captured consistently or
  blocks/revokes sleep;
- no-deadline state remains hibernatable;
- multiple wake causes coalesce one provisioning attempt without lost
  individual delivery;
- same-request replay returns the same destination generation or terminal
  result;
- stale epoch, source generation, lifecycle revision, wake request, and late
  provisioning results fail closed;
- restore and scheduler reconciliation complete before admission;
- retained work remains durable across provisioning, restore, readiness, and
  host-process restart failures;
- quarantine blocks destruction, generation grant, delivery, and implicit
  reset;
- Status and Doctor expose bounded reason codes and operator actions;
- no credential, artifact location, message payload, prompt, or cron definition
  appears in lifecycle status or audit evidence.

The cheapest complete proof uses one hibernated runtime, one retained Teams
message, one cron deadline, one host restart during provisioning, and one fresh
destination generation. It must show one coalesced wake, scheduler
reconciliation before readiness, and independent delivery of both causes.
