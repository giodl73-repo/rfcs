# State CAPE v1 Specification

This document is the implementer-facing capability and conformance
specification for the State CAPE levels defined by RFC 0021, Runtime State
Continuity. CAPE projects independently implemented continuity capabilities
into four operator-visible guarantees: Conventional, Archived, Portable, and
Elastic.

Status: draft, tied to RFC 0021.

## Scope

This specification defines:

- the four State CAPE v1 levels and their monotonic ordering;
- the minimum capabilities required to claim each level;
- state-surface, artifact, restore, replacement, hibernation, and wake
  invariants;
- validation, Status, Doctor, readiness, and audit requirements;
- downgrade, degraded operation, quarantine, and disaster-reset rules; and
- the conformance evidence required for an implementation to advertise a
  level.

This specification does not define:

- canonical state-store schemas or writer APIs;
- one storage backend, publication transport, host, scheduler, or Channel;
- exact checkpoint cadence, retention count, wake SLO, or idle policy;
- a global transaction or aggregate mutation watermark;
- implementation module boundaries; or
- the Portable publication provider API, which is defined by the
  [Portable Publication Provider v1 Specification](portable-publication-provider-v1-spec.md).

Optional canonical Readiness, Hosting Profile, and Hosted Integration
composition is defined by the
[State CAPE Readiness and Hosting Composition v1 Addendum](readiness-hosting-composition-v1-addendum-spec.md).

## Terminology

- **Logical runtime**: the continuing OpenClaw runtime identity independent of
  one process, machine, container, or generation.
- **Runtime generation**: exclusive authority for one compute generation to
  accept root work and advance recovery lineage.
- **State surface**: persistent or reconstructable state owned by one
  component.
- **Recovery point**: an immutable manifest plus the exact component artifacts
  and provenance it identifies.
- **Local capture**: a verified recovery artifact available at a local
  materialization boundary.
- **Accepted recovery point**: a recovery point immutably accepted by the
  configured durability boundary.
- **Restore dependency closure**: every artifact, compatible runtime, plugin,
  credential, identity, workspace, and shared capability required to activate
  restored state.
- **Retained ingress**: work durably held outside an absent runtime until a
  current generation becomes ready.
- **Selected level**: the level configured by the operator or Hosting Profile.
- **Effective level**: the highest level whose required capabilities currently
  pass validation.

## Level ordering

State CAPE v1 defines this strict order:

```text
Conventional < Archived < Portable < Elastic
```

Each level includes every requirement of the levels below it. An
implementation must not:

- claim a level while omitting a lower-level requirement;
- define separate transition protocols for C-to-A, A-to-P, or P-to-E;
- use a different state inventory, artifact format, or restore validator at a
  higher level without an explicit versioned compatibility rule; or
- silently report a lower effective level as satisfying the selected level.

A deployment may use an upper-level capability while configured for a lower
level. That does not raise its effective level until every requirement of the
higher level is validated.

## Common invariants

All levels preserve existing state ownership. CAPE does not change canonical
stores, transaction boundaries, acknowledgement semantics, or writer
participation.

From Archived onward:

- every required state surface must be captured or explicitly classified;
- recovery claims must bind immutable identity and integrity evidence;
- local capture and external acceptance must remain distinct claims;
- failures and unknown outcomes must be visible and machine-readable;
- host-managed, reissuable, or provider credentials and tokens must be
  re-issued or re-resolved and must never enter artifacts or manifest metadata;
- a credential semantic owner may export only non-reissuable runtime-owned
  identity through an explicitly approved encrypted, runtime-scoped artifact;
- restore must validate before activating restored state; and
- disaster reset must create a new continuity epoch with explicit data-loss
  audit rather than silently abandoning the prior lineage.

From Portable onward:

- runtime identity must not depend on the source process or machine;
- each replacement uses a new runtime generation;
- stale generations must not accept root work, publish newer lineage, authorize
  destruction, or deliver retained ingress;
- final handoff and restore authority must be explicit and durable;
- readiness must remain closed until restore dependency closure validates;
- attachments and user-facing sidecars that read or mutate restored surfaces
  must not activate, report ready, or serve requests before the relevant
  continuity restore criterion is `True`; and
- contradictory, stale, corrupt, or identity-mismatched lifecycle evidence
  must enter durable quarantine rather than fall back or continue.

At Elastic:

- correctness must not depend on a resident OpenClaw process;
- wake intent, deadlines, retained ingress, and retry state must survive absent
  compute;
- newly queued work must revoke sleep authorization; and
- retained work must be delivered only after readiness to the current
  generation.

## C — Conventional

### Guarantee

Conventional preserves existing OpenClaw startup, runtime, and clean-shutdown
behavior. Components persist independently under their current atomicity
rules. CAPE makes no aggregate recovery-point, RPO, replacement, or
scale-from-zero claim.

### Requirements

A Conventional deployment must:

- preserve each state owner's current persistence and recovery semantics;
- report that no aggregate continuity guarantee is active; and
- avoid presenting ordinary local persistence as an Archived recovery point.

Conventional does not require:

- aggregate state inventory;
- checkpoint publication;
- explicit restore;
- runtime-generation fencing;
- host lifecycle authority; or
- retained ingress.

### Conformance

Conformance proves that enabling CAPE configuration at `conventional` does not
change state formats, writer behavior, normal startup, clean shutdown, or
existing readiness semantics.

## A — Archived

### Guarantee

Archived adds integrity-checked recovery points, explicit restore, retention
and fallback policy, and an observable recovery-point objective (RPO).
Archived means actively protected; it does not require cold or write-once
storage.

Archived guarantees recoverability to an accepted point. It does not guarantee
automatic replacement, continuous availability, one aggregate semantic
instant, or zero loss after forced termination.

### State inventory

Every known state surface must be classified as exactly one of:

- **captured**: represented by a component artifact and restore contract;
- **external**: re-resolved from an authority outside the recovery point;
- **reconstructed**: deterministically rebuilt from declared inputs;
- **ephemeral**: intentionally excluded without affecting the Archived
  guarantee; or
- **blocking**: prevents the selected level from validating.

Required captured surfaces must define:

- stable component identity;
- native capture semantics;
- artifact identity, digest, size, and format;
- compatibility requirements;
- restore ordering and target ownership; and
- validation performed before activation.

Unknown required surfaces fail validation. An implementation must not classify
a surface as ephemeral only because capture is unavailable.

### Capture and acceptance

Archived must:

- create recurring recovery points using each component's supported online
  capture semantics;
- avoid a runtime-wide pause for recurring capture;
- produce one immutable manifest for the exact artifact set;
- verify artifact and manifest integrity before reporting capture success;
- distinguish local materialization from acceptance by the configured
  durability boundary;
- bind acceptance to exact runtime, checkpoint, artifact, manifest, provider,
  and durability identity; and
- treat an uncertain acceptance outcome as unknown, not failed or successful.

Exact replay must be idempotent. Reuse of an identity with different artifact
or manifest content must conflict.

### RPO, retention, and fallback

The reported RPO is the elapsed time since the newest accepted recovery point.
It is not a global mutation watermark.

Archived must expose:

- configured RPO target and actual age;
- retention policy and currently retained points;
- selected restore point and fallback depth;
- stale, missing, corrupt, or incompatible points; and
- any interval lost by fallback.

Automatic fallback may choose only a policy-allowed compatible point. It must
report the selected point and actual lost interval.

### Explicit restore

Restore must:

1. select an immutable recovery point;
2. verify artifact and manifest integrity against capture-time digests stored
   in continuity or lifecycle authority independently of the publication
   provider receipt;
3. validate runtime, schema, plugin, and component compatibility;
4. resolve the declared dependency closure;
5. materialize each component through its owner-defined restore contract;
6. perform allowed migrations in declared order;
7. validate the complete result; and
8. keep admission closed until validation completes.

An incomplete required restore must not silently become a clean start.

### Archived conformance

Conformance proves:

- complete state-surface accounting and explicit exclusions;
- bounded online capture under timeout, cancellation, saturation, and I/O
  pressure;
- immutable publication and retrieval with exact replay and conflict cases;
- corruption, incompatibility, retention, and fallback behavior;
- explicit restore of every required captured surface; and
- reported RPO equality with the actual accepted-point age.

## P — Portable

### Guarantee

Portable adds generation-fenced replacement and restored startup on fresh
compute. The logical runtime can move away from its source process, machine,
container, or cell while preserving one recovery lineage.

Portable does not require the runtime to become absent during ordinary idle
periods and does not by itself guarantee retained-ingress wake.

Replacement without a completed planned handoff, including crash relocation,
restores the newest policy-allowed accepted recovery point under a new
generation. It reports the interval since that point as lost and does not imply
clean final-state capture.

### Exclusive generation authority

Portable must maintain one durable authority that:

- grants one current runtime generation;
- rejects stale start, restart, wake, health-recovery, and autoscaling paths;
- fences root-work admission and recovery-lineage advancement;
- records source and destination generations separately; and
- fails closed when authority is unknown.

No friendly deployment name, process-local registry generation, provider
process, or carrier incarnation may substitute for runtime generation
authority.

### Planned handoff

A successful planned replacement must:

1. close admission to new root work;
2. identify and drain or fence writable owners;
3. complete conventional clean shutdown;
4. capture the final closed persisted state;
5. obtain immutable acceptance for that exact final recovery point;
6. bind destruction authorization to the source generation and accepted
   identity; and
7. allocate and restore a new generation.

`safeToDestroy` must not be inferred from readiness, local capture, provider
invocation, or process exit. A failed or unknown final publication must not
authorize destruction.

### Restore hold and fresh compute

Before any original restore target is created, the lifecycle owner must acquire
a durable restore hold for the logical runtime and destination generation. The
hold must:

- block every launcher and adapter start path;
- survive holder failure without TTL-based reopening;
- reject stale generations and mismatched restore identities; and
- admit exactly one startup matching the committed restore result.

The destination must use a new runtime generation and record the immutable
source checkpoint separately. The recovery point owner must equal the
destination logical runtime's trusted tenant-cell owner identity; receipt or
request data cannot override that binding.

### Dependency closure and readiness

Portable requires every restore dependency to be captured, externally
re-resolved, reconstructed, or blocking. Missing compatible runtime support,
plugins, credentials, identity, workspace, or shared capabilities prevent the
Portable claim.

Readiness remains closed through:

- artifact and manifest validation;
- compatibility and migration;
- dependency resolution;
- identity reconstruction;
- component activation; and
- scheduler reconciliation.

A Hosting Profile cannot weaken this restored-startup gate.

### Portable conformance

Conformance includes every Archived test and additionally proves:

- fresh-compute restore with a new generation;
- stale-generation rejection across publication, handoff, restore, start, and
  admission;
- final clean-shutdown capture and exact acceptance;
- no destruction authorization on invalidating warning, failure, or unknown
  outcome;
- durable restore hold across every start path and holder failure;
- dependency-closure behavior for captured, external, reconstructed, and
  blocking dependencies; and
- readiness remaining closed through complete restored validation; and
- durable quarantine for stale generation, changed provider provenance,
  corrupt retrieval, contradictory restore evidence, and unsafe replay.

## E — Elastic

### Guarantee

Elastic adds hibernation and wake. No runtime process needs to exist while
idle. Retained ingress or an OpenClaw-owned semantic deadline provisions a new
Portable runtime, which restores and becomes ready before work delivery.

Elastic v1 does not make a Channel wake-capable unless that Channel owner proves
the retained-ingress contract.

### Hibernation

Elastic hibernation must:

1. begin from a valid Portable runtime generation;
2. durably retain or reject new ingress before sleep authorization;
3. close admission and drain or fence active work;
4. complete the Portable final-handoff requirements;
5. atomically accept the final recovery point and wake intent;
6. record the earliest semantic wake deadline and reason class;
7. authorize destruction only while sleep authorization remains valid; and
8. leave no required checkpoint, deadline, ingress, or retry state solely in
   the retiring process; and
9. durably externalize required audit and lifecycle-event records before
   destruction authorization.

Newly queued work must revoke sleep authorization before compute destruction.

### Wake-capable ingress

Each ingress allowed to wake absent compute must independently prove:

- durable pre-ack retention;
- stable deduplication identity;
- atomic sleep revocation and wake request;
- retry and terminal-failure observability;
- generation-fenced delivery;
- delivery only after restored readiness; and
- no loss when multiple events coalesce into one compute wake.

An enabled ingress that requires a resident process either keeps compute
resident or blocks Elastic validation. It must not be silently omitted from the
claim.

### Scheduler deadlines

OpenClaw remains authoritative for cron definitions, due decisions, catch-up,
and duplicate suppression. Before hibernation, OpenClaw must publish the
earliest semantic `nextRequiredAt` deadline and reason class.

The host may apply cold-start lead time but must not reinterpret cron semantics.
After restore, OpenClaw must reconcile missed schedules and durably queue
policy-allowed catch-up before admission opens.

### Wake and delivery order

Elastic wake follows this order:

```text
retained ingress or semantic deadline becomes due
  -> revoke sleep authorization
  -> coalesce provisioning
  -> allocate a new runtime generation
  -> retrieve the accepted recovery point
  -> restore and validate dependency closure
  -> reconcile scheduler state
  -> publish continuity and owner readiness
  -> open admission
  -> deliver retained work to the current generation
```

A failed restore keeps ingress retained. Delivery to a stale, unready, or
quarantined generation fails closed.

### Elastic conformance

Conformance includes every Portable test and additionally proves:

- final recovery-point and wake-intent atomicity;
- sleep revocation racing new ingress;
- successful wake with no resident OpenClaw process;
- durable Teams, host/API, cron, or other explicitly conforming wake sources;
- event coalescing without lost individual delivery;
- retained delivery only after readiness to the current generation;
- scheduler reconciliation and duplicate suppression before admission;
- visible missed deadlines, provisioning failures, and retry exhaustion; and
- quarantine blocking wake, generation grant, delivery, and implicit reset
  while preserving retained work and recovery evidence.

## Validation and level claims

Configuration selects one level:

```text
conventional | archived | portable | elastic
```

Validation must evaluate every requirement at or below the selected level.
Missing required capability, unresolved provider, unsupported state surface,
incompatible dependency, non-wake-capable enabled ingress, or unknown authority
must fail the selected-level validation.

An implementation may compute a lower effective level for diagnostics, but it
must not:

- start a managed operation that relies on the selected level;
- report the selected level as ready;
- silently rewrite configuration to the lower level; or
- discard blocking evidence.

An explicit operator change may select a lower level. That change does not
erase existing lineage, retained work, quarantine, or audit history.

## Readiness, Status, and Doctor

Continuity publishes owner-scoped criteria:

```text
openclaw.continuity-archived
openclaw.continuity-portable
openclaw.continuity-elastic
openclaw.continuity-recovery-point-current
```

Each criterion reports only continuity-owned truth. Channel, credential,
workspace, ingress, and lifecycle owners report their own criteria. A Hosting
Profile composes the end-to-end posture without duplicating CAPE policy.

Status must expose:

- selected and effective level;
- configuration and policy provenance;
- latest local capture and accepted recovery point;
- configured and actual RPO;
- current runtime and provider generations;
- lifecycle phase and active blockers;
- selected restore source, fallback depth, and compatibility result;
- unresolved dependency-closure requirements;
- earliest semantic wake deadline and accepted wake registration;
- required and advisory readiness posture; and
- last failure, retryability, and operator action.

Doctor must report:

- unknown or unclassified state surfaces;
- missing, ambiguous, stale, or incompatible publication providers;
- corrupt, stale, or insufficiently retained recovery points;
- restore incompatibility or unresolved dependencies;
- missing generation or restore-hold authority;
- unsupported enabled wake sources;
- missed or unregistered semantic deadlines; and
- selected-level requirements that are not satisfied.

Status and Doctor must not expose credentials, protected artifact metadata, or
provider secrets.

## Failure and quarantine rules

Failures must identify the operation, logical runtime, relevant generation and
checkpoint, affected component or provider, retryability, outcome certainty,
and required operator action.

Unknown publication, destruction, restore, generation, delivery, or
disaster-reset authority fails closed.

Quarantine is required when evidence is contradictory, stale, corrupt,
identity-mismatched, or unsafe to replay. Quarantine must:

- prevent wake, generation grant, restore commit, admission, retained delivery,
  and implicit disaster reset as applicable;
- preserve artifacts, retained work, lineage, and diagnostic evidence; and
- require explicit authenticated and audited operator reconciliation or an
  authenticated audited disaster reset.

Auditable lifecycle facts must be durably persisted outside any single runtime
process before destruction is authorized. Their integrity and ordering must be
verifiable, and downgrade, quarantine, or disaster reset must not erase them
before the configured audit-retention boundary.

## Policy boundaries

The following may be configurable policy:

- durability and encryption boundary;
- RPO target;
- retention and fallback limits;
- replacement triggers;
- idle threshold and minimum residency;
- cold-start lead and wake SLO; and
- audited disaster-reset permission.

The following are mandatory capability or safety requirements and must not be
weakened by policy:

- required state-surface accounting;
- manifest and artifact integrity;
- native online capture semantics;
- generation fencing for replacement;
- immutable final acceptance before destruction;
- restore validation before readiness;
- retained-ingress durability for wake-capable sources; and
- OpenClaw ownership of scheduler semantics and duplicate suppression;
- fail-closed handling of unknown authority; and
- quarantine prevention of wake, generation grant, restore commit, admission,
  retained delivery, and implicit disaster reset.

## Compatibility and evolution

State CAPE v1 is a capability standard, not a serialized artifact schema.
Artifact, manifest, provider, lifecycle, and owner contracts retain their own
versions.

A future CAPE revision may add requirements but must not silently redefine a v1
level. Implementations must expose the CAPE specification version used for a
level claim. New required capabilities or weakened guarantees require an
explicit new specification version.

## Conformance report

A conformance report must identify:

- implementation and OpenClaw versions;
- CAPE specification version;
- selected level and tested effective level;
- state inventory and explicit exclusions;
- publication and durability implementation;
- independent capture-integrity authority used during restore;
- host and lifecycle authority, when required;
- enabled wake sources, when required;
- policy values used by the test;
- every required test result;
- unsupported or advisory capabilities; and
- failure-injection coverage; and
- audit persistence and quarantine-exit authorization evidence.

Passing a happy-path backup and restore is insufficient. The report must cover
the failure, replay, stale-generation, corruption, timeout, crash-window,
readiness, and race cases required by the claimed level.
