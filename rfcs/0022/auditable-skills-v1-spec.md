# Auditable Skills Receipt Core and Extension Profiles Specification

This document is the implementer-facing specification for RFC 0022, Auditable
Skills. The RFC explains the motivation, ownership model, and rollout plan.
This file defines a standalone Receipt Core profile and records later extension
profiles without making them prerequisites for receipt conformance.

Status: draft, tied to RFC 0022. Receipt Core is the only Round 1 profile.

## Scope

The **Receipt Core** profile defines:

- typed receipts emitted by successful tools;
- portable record, get, list, and count semantics;
- a configurable shared local SQLite implementation profile;
- agent, session, run, tool, and tool-call correlation;
- minimum query, sanitization, boundedness, and failure behavior;
- receipt producer and Receipt Core harness conformance.

This document also records three **later extension profiles**:

- Skill Declaration: optional `SKILL.md` outcomes, child skills, and isolation
  intent;
- Managed Skill Identity: native child-run identity, exact executed-skill
  identity, and parent lineage;
- Run Accounting: normalized run usage, captured cost, and a joined audit-run
  projection.

An implementation may claim any later profile only when it also claims Receipt
Core and satisfies that extension's requirements and test vectors. Receipt Core
conformance does not require any later profile.

This specification does not define:

- a portable workflow language;
- workflow step syntax, branching, retries, approvals, or resume storage;
- package installation or Claw lifecycle;
- model, tool, credential, or child-skill grants;
- CRM accounts, cases, queues, SLAs, or assignment;
- provider pricing catalogs or retention policy;
- tamper-evident logs, signatures, regulatory attestations, or non-repudiation;
- authoritative budgets inside skill metadata.

Future workflow-runner research is preserved separately in
[`orchestration-runner-v1-spec.md`](orchestration-runner-v1-spec.md). It is not
part of Receipt Core.

## Normative language

The terms **must**, **must not**, **should**, **should not**, and **may** are
normative within the profile being claimed. An implementation may expose
different internal types, storage, and APIs when its externally observable
behavior follows the claimed profile.

## Ownership model

Auditable Skills v1 separates three authorities:

| Owner | Owns | Does not establish |
| --- | --- | --- |
| Skill package | Declared outcomes, possible child skills, isolation intent | Runtime facts, permissions, budgets, or successful effects |
| Claw or caller | Allowed skill graph, model and execution policy, limits | Evidence that an effect occurred |
| Harness | Executed identity on native runs, tool evidence, usage, cost, status | Business meaning beyond producer-defined receipt fields |

An implementation must not treat package metadata as proof that an outcome
occurred. It must not treat metadata as a permission grant.

## Compatibility and evolution

Version 1 uses these compatibility rules:

- Skills without the metadata in this specification remain valid ordinary
  skills.
- Implementations may ignore all three metadata keys.
- Unknown metadata keys must not prevent ordinary skill discovery or
  instruction loading.
- Unknown or malformed values must not break ordinary instruction loading.
  Implementations should emit a bounded diagnostic when a recognized value is
  ignored. A caller that requires a declaration for policy must fail closed
  when that normalized declaration is absent; it must not treat malformed data
  as permission.
- Receipt producers may add optional fields within `data` without changing the
  core version.
- A breaking change to a core record requires a new `schemaVersion`.
- Audit readers must reject unsupported core schema versions rather than
  reinterpret them as v1.

Implementations may accept namespaced aliases while the metadata vocabulary is
incubating. The author-facing v1 names are the direct names below.

## Skill Declaration extension profile (later)

This profile is not part of Receipt Core or Round 1.

Auditable Skills uses the Agent Skills string-valued `metadata` map.

```yaml
---
name: issue-refund
description: Verify a refund request and issue an approved customer refund.
metadata:
  outcomes: "customer.verified payment.refunded"
  uses-skills: "verify-customer check-refund-policy"
  isolation: "required"
---
```

### Metadata fields

| Field | Type | Required | Semantics |
| --- | --- | --- | --- |
| `outcomes` | string | No | Whitespace-separated receipt types the skill intends to produce. |
| `uses-skills` | string | No | Whitespace-separated skill names the skill may request as managed children. |
| `isolation` | string | No | `shared`, `preferred`, or `required`. |

```ts
type SkillExecutionHintsV1 = {
  outcomes?: string[];
  usesSkills?: string[];
  isolation?: "shared" | "preferred" | "required";
};
```

List values are split on one or more Unicode whitespace characters. Empty
tokens are discarded. Implementations should preserve declaration order while
removing exact duplicates for policy and reporting.

Outcome identifiers and skill names must not contain whitespace. Outcome
identifiers should be stable, producer-owned dotted names such as
`payment.authorized`, not generic state words such as `done`.

Implementations must bound metadata value length and parsed list size before
using these fields for planning or policy.

### Isolation values

| Value | Required behavior |
| --- | --- |
| `shared` | The skill may execute in the current run. Usage remains shared. |
| `preferred` | The harness should create an isolated child run when supported and may fall back to shared execution. |
| `required` | The harness must reject managed invocation before dispatch when it cannot create an isolated child run. |

An isolated run is the minimum exclusive accounting boundary. An
implementation must not divide a shared model turn among skills and present the
result as exclusive attribution.

### Effective child graph

`uses-skills` narrows possible composition. A managed child may run only when
it is allowed by all applicable layers:

1. the parent declaration, when the caller requires declared children;
2. Claw or direct-caller policy;
3. skill visibility and model-invocation policy;
4. ordinary tool, sandbox, credential, and model restrictions.

A declaration must never widen any of these layers.

## Receipt Core profile

A receipt is producer-owned evidence attached to a completed successful tool
result.

```ts
type SkillReceiptV1 = {
  type: string;
  version?: number;
  subject?: {
    type: string;
    id: string;
  };
  data?: Record<string, unknown>;
};
```

Example:

```json
{
  "type": "payment.authorized",
  "version": 1,
  "subject": {
    "type": "invoice",
    "id": "INV-1042"
  },
  "data": {
    "authorizationCode": "AUTH-9482"
  }
}
```

### Receipt fields

| Field | Type | Required | Semantics |
| --- | --- | --- | --- |
| `type` | non-empty string | Yes | Primary exact-match business outcome dimension. |
| `version` | positive integer | No | Producer schema version for this receipt type. |
| `subject.type` | non-empty string | Conditional | Producer-owned kind of affected object. |
| `subject.id` | non-empty string | Conditional | Stable producer-owned object identifier. |
| `data` | JSON object | No | Bounded producer-defined evidence. |

`subject` is valid only when both child fields are present. OpenClaw does not
interpret subject or data business meaning.

### Receipt admission

The harness must admit a receipt only when:

- the owning tool call completed successfully;
- the receipt passes structural validation and configured size limits;
- sanitization and redaction complete before durable recording.

Validation, sanitization, and recording must be bounded and must not perform
request-time network I/O. The implementation must cap receipts admitted from
one tool result and bound total size and store lock wait for that result. A
recorder or sanitizer failure must be contained so it cannot crash the Gateway
or change the tool result.

Failed tools must not emit success receipts. Model prose, skill declarations,
and assistant claims must not be converted into receipts without an explicit
trusted producer boundary.

Malformed receipts are omitted and produce an observable audit diagnostic.
Receipt-recording failure must not rewrite the underlying tool result.

### Recorded receipt envelope

The harness wraps each admitted producer receipt in trusted execution
correlation before durable recording.

```ts
type RecordedSkillReceiptV1 = {
  receiptSchema: "openclaw-audit-receipt";
  schemaVersion: 1;
  receiptId: string;
  sequence: number;
  type: string;
  version?: number;
  occurredAt: number;
  agentId: string;
  sessionId: string;
  sessionKey?: string;
  runId: string;
  invocationId?: string;
  skillName?: string;
  skillDigest?: string;
  toolName: string;
  toolCallId: string;
  subject?: { type: string; id: string };
  data?: Record<string, unknown>;
};
```

`occurredAt` is Unix time in milliseconds. `sessionId` identifies the
transcript instance; `sessionKey`, when present, identifies the stable logical
route or thread. The receipt producer supplies only the `SkillReceiptV1`
fields. Receipt ID, sequence, time, agent, tool, tool-call, session, run, and
optional invocation and skill correlation are harness facts and must not be
accepted from the producer as authoritative correlation.

`runId` is the canonical join to the originating run. Optional invocation and
skill fields belong to the Managed Skill Identity extension. When present,
they are denormalized convenience fields and must match the managed descriptor
associated with the same run.

The recorded receipt is the canonical full business-evidence record. A normal
trajectory contains only this bounded reference:

```ts
type TrajectoryReceiptReferenceV1 = {
  type: "audit.receipt.recorded";
  data: {
    receiptId: string;
    type: string;
    version?: number;
    subject?: { type: string; id: string };
    invocationId?: string;
    skillName?: string;
    skillDigest?: string;
    toolName: string;
    toolCallId: string;
  };
};
```

The reference preserves ordered run history without copying receipt `data`
into session telemetry. Consumers that need full evidence resolve `receiptId`
through the receipt store.

### Receipt store

The receipt contract has portable idempotent record, get by receipt ID,
exact-filter list, and count semantics. It must index exact receipt type and
should index subject, agent, session key, run, invocation, and skill identity.
The local SQLite profile may implement those operations directly; Receipt Core
does not require a provider interface before a second storage implementation
exists.

An OpenClaw installation defaults to one shared local SQLite receipt database:

```json5
{
  audit: {
    receipts: {
      enabled: true,
      store: {
        type: "sqlite",
        path: "~/.openclaw/state/receipts.sqlite",
      },
    },
  },
}
```

Every agent served by that installation uses the configured store, so an
operator can search and count outcomes across agents without scanning every
session database. SQLite v1 is a single-host profile. The database must remain
on storage local to the Gateway; direct network-filesystem or multi-host SQLite
sharing is not supported. A future remote provider should preserve the same
producer and query semantics. Introducing that provider is the point at which a
shared implementation interface should be extracted.

The receipt store has its own retention, backup, and access policy. Session or
trajectory rotation must not delete its full receipts. Deleting a receipt may
leave a historical trajectory reference unresolved; implementations must not
reconstruct full evidence from model prose or other untrusted content.

Recording uses a harness-owned source identity such as agent, session, run,
tool call, and receipt position. Repeating the same source identity with the
same normalized receipt must return the existing record. Reusing it with
different content must fail as an idempotency conflict. A producer cannot
choose this identity or overwrite an existing receipt.

The store must bound record size, transaction and lock wait, and in-memory
queueing. Receipt persistence must not add an unbounded wait to the Gateway's
tool-result path. A store error remains observable but must not crash the
Gateway or rewrite the completed tool result. The implementation must not
silently fall back to a different per-agent or in-memory store.

The SQLite profile must:

- reject a database schema newer than the running implementation supports;
- create or migrate its schema in an explicit transaction before accepting
  writes;
- expose health diagnostics for path, permissions, lock timeout, corruption,
  and unsupported schema without including receipt payloads;
- use a consistent SQLite snapshot mechanism for backup and export rather than
  copying a live database file;
- keep database, journal, and temporary files private to the OpenClaw account.

Disabling new receipt recording must not make existing records unreadable.

## Managed Skill Identity extension profile (later)

This profile is not part of Receipt Core or Round 1.

Every accepted managed skill call receives one stable invocation ID and starts
one ordinary child run. Auditable Skills adds immutable skill identity to that
native record; it does not create a parallel invocation lifecycle.

```ts
type ExecutedSkillIdentityV1 = {
  name: string;
  source?: string;
  version?: string;
  digest: string;
};

type ManagedSkillDescriptorV1 = {
  invocationId: string;
  skillName: string;
  skillSource?: string;
  skillDigest: string;
  parentRunId?: string;
  executionHints?: SkillExecutionHintsV1;
};

type ManagedSkillRunV1 = {
  schemaVersion: 1;
  runId: string;
  childSessionKey: string;
  managedSkill: ManagedSkillDescriptorV1;
};
```

The installed package identity is authoritative for version and digest when
available. A self-declared version is descriptive only. Workspace skills use a
canonical source identity and full content digest.

`ExecutedSkillIdentityV1` is the normalized reporting form of the native
descriptor. Package version may be joined from retained install provenance
when available; name and full digest identify the executed content without it.
The existing child-run record remains authoritative for status, timestamps,
duration, error, cancellation, cleanup, model selection, and session identity.
Consumers resolve those fields by `runId`; they must not require a second
pending/running/terminal invocation record. A managed parent is resolved by
following `parentRunId` to the parent run's descriptor, so
`parentInvocationId` need not be copied onto every child.

A request rejected before child dispatch returns a structured error and
creates no `ManagedSkillRunV1`. Once accepted, `runId` and `childSessionKey`
are required. Retries that start distinct native runs retain distinct run IDs
and incurred usage.

Managed dispatch reuses the caller's current skill snapshot and ordinary
OpenClaw session, subagent, policy, sandbox, tool, credential, and model
boundaries. The managed path must not create broader authority than an
equivalent direct child run.

## Run Accounting extension profile (later)

This profile is not part of Receipt Core or Round 1.

Usage belongs to the run that consumed it.

```ts
type NormalizedRunUsageV1 = {
  input?: number;
  output?: number;
  cacheRead?: number;
  cacheWrite?: number;
  reasoningTokens?: number;
  total?: number;
};

type RunCostV1 = {
  usd: number;
  basis: "provider-billed" | "catalog-estimate" | "mixed";
};
```

Token values must be finite, non-negative integers. Cost must be a finite,
non-negative number. Missing usage or pricing must be omitted, never invented
as zero.

The provider aggregate is preferred when present. Otherwise `total` is input
plus output. Cache read and cache write remain separate dimensions unless the
provider's normalized aggregate explicitly includes them. `total` must not be
populated from OpenClaw's context-window `totalTokens` snapshot. Failed,
retried, and timed-out attempts are included whenever the provider reported
usage.

The harness should carry cumulative usage on the terminal native-run lifecycle
or completion result and snapshot it on the exact retained run record. The
native `runId` is the accounting identity. Consumers must not reconstruct an
older run from mutable session usage after a later turn, steer, or replacement
run. A replacement run starts with no inherited usage.

Cost and its basis are captured with the run. Historical audit output must not
silently change when catalog pricing changes later.

An operator-facing native subagent view may expose the current session usage
and cost snapshot beside managed identity. A workflow accounting result must
use cumulative observed usage for the contributing run, including reported
failed or retried attempts, and must not substitute a context-window token
snapshot. If the cumulative value cannot be established, it is unavailable
rather than zero.

Likewise, a mutable session cost estimate is not exact historical run cost. An
implementation may expose token usage before cost, but it must omit exact-run
cost until the amount and basis can be captured with that run.

### Managed-run usage check

A harness may expose a read-only check over an explicit set of retained managed
run IDs:

```ts
type ManagedRunUsageCheckV1 = {
  runIds: string[];
  usage: NormalizedRunUsageV1 & { total: number };
  budget?: {
    maxTokens: number;
    remainingTokens: number;
    decision: "within_limit" | "limit_reached";
  };
};
```

The check must:

1. apply the harness's existing run-visibility boundary;
2. accept only terminal runs with managed skill identity;
3. deduplicate by exact `runId` before aggregation;
4. prefer the retained provider total and otherwise use retained input plus
   output, leaving cache buckets separate;
5. return `accounting_unavailable` with the affected run ID when required usage
   cannot be established;
6. treat `maxTokens` as caller-owned input for this decision, not durable skill
   metadata or a new budget ledger; and
7. report `limit_reached` when observed total is equal to or greater than the
   ceiling.

This check occurs between completed runs. It cannot reserve future tokens or
guarantee that one active run will not exceed the ceiling.

## Joined audit-run projection (later)

This projection belongs to the Run Accounting extension profile.

An audit consumer should be able to obtain one versioned run projection that
joins managed-run identity, observed receipts, model identity, usage, and cost.

```ts
type AuditableSkillRunV1 = {
  schemaVersion: 1;
  sessionId: string;
  sessionKey?: string;
  runId: string;
  firstEventAt: string;
  lastEventAt: string;
  status?: "completed" | "failed" | "cancelled";
  managedRun: ManagedSkillRunV1;
  models: Array<{
    provider?: string;
    modelId?: string;
  }>;
  accountingScope: "exclusive" | "shared";
  usage?: NormalizedRunUsageV1;
  cost?: RunCostV1;
  receipts: RecordedSkillReceiptV1[];
};
```

The projection joins receipt-store records with existing trajectory, session,
and usage facts. It does not duplicate full receipt payloads into a workflow or
usage ledger.

`firstEventAt` and `lastEventAt` bound the observed run history. `status` is
present only when a terminal native run status is known.

`exclusive` means one isolated managed invocation owns the run's usage and
cost. `shared` means multiple skills or invocations may have contributed. Audit
consumers must not divide shared usage among those skills or present it as
exclusive step cost. Multiple model entries preserve provider fallback and
multi-model execution rather than selecting one arbitrary identity.

When RFC 0016 Claw provenance is available, the projection should also include
the authoritative Claw id, version when known, and digest. That extension must
come from install provenance, not skill metadata.

## Query requirements

The stable query contract has three operations:

- `get(receiptId)` returns one full receipt or a typed not-found result;
- `list(query)` returns a bounded, stably ordered page of full receipts;
- `count(query)` returns the number of records matching the same filters
  without materializing receipt payloads.

```ts
type ReceiptFilterV1 = {
  type?: string;
  subject?: { type: string; id?: string };
  agentIds?: string[];
  sessionKey?: string;
  runId?: string;
  invocationId?: string;
  skill?: { name: string; digest?: string };
  tool?: { name?: string; callId?: string };
  occurredAfter?: number;
  occurredBefore?: number;
};

type ReceiptQueryV1 = ReceiptFilterV1 & {
  order?: "oldest" | "newest";
  limit: number;
  cursor?: string;
};

type ReceiptPageV1 = {
  receipts: RecordedSkillReceiptV1[];
  nextCursor?: string;
};
```

`count` accepts `ReceiptFilterV1`; pagination fields do not affect the count.

A Receipt Core implementation must support exact filtering of observed
receipts by `type`. It should additionally support filtering by:

- receipt subject type and id;
- run id;
- session key;
- tool name and tool-call id;
- time range.

An implementation claiming Managed Skill Identity should additionally support
exact skill name and digest plus invocation ID filters.

The query surface must return the originating run and session correlation so an
operator or later agent can revisit the work thread. It must distinguish
declared outcomes from observed receipts.

`list` must impose a maximum limit and deterministic ordering with `receiptId`
or store sequence as the final tie-breaker. Cursors are opaque and scoped to
the normalized filter and ordering; a cursor must not be accepted with a
different query. Invalid cursors fail explicitly rather than restarting at the
first page. An empty `agentIds` list matches no agents.

Receipt Core implementations should support grouping observed receipts by exact
type and time window as a reporting projection. CLI, Gateway, plugin, UI,
workflow, and export surfaces must reuse the same query semantics rather than
scanning trajectory files directly. The local SQLite profile may expose those
operations from its SQLite module without a separate provider interface.

Every public query surface must apply its existing caller, agent, session, and
plugin authorization before calling the store. Supplying `agentIds` is a
filter, not an authority grant. `get`, `list`, and `count` must use the same
visibility rules so counts cannot reveal records whose full receipts the caller
could not read. A workflow adapter receives receipts only for the managed run
it is resolving.

## Retention, sanitization, and export

Implementations must apply existing secret and sensitive-data handling before
durable recording and export. Receipt producers should record the minimum
evidence needed for later verification.

Producer data may contain authorization codes and other sensitive business
evidence. In the local SQLite profile, any operating-system user who can read
the configured database can read that evidence. Operators therefore own file
access, backup access, export authorization, and retention policy for the
receipt store.

Retention, backup, and export policy are deployment concerns. An implementation
must not claim durable revisitability beyond its configured retention window.
Receipt retention is independent of transcript and trajectory retention. A
retained receipt remains searchable after session telemetry rotates, although
the transcript needed to reconstruct conversational context may no longer be
available. An external system remains authoritative for business objects it
owns.

Retention cleanup must be bounded and observable. It must delete canonical
receipt records without rewriting trajectories; an old trajectory reference
may therefore resolve as not found. Export must preserve receipt schema
version, receipt ID, exact type, occurrence time, and harness correlation.
Exports containing producer `data` require the same or stronger authorization
and redaction policy as direct `get` and `list` operations.

Managed child-run metadata may have a different retention window from full
receipts. An implementation that claims later skill-level attribution must
retain or export the `runId` to managed-skill association for that claimed
window. If the association has expired, readers report identity as unavailable;
they must not infer it from transcript prose or a receipt producer's data.

The v1 envelope provides correlation, not tamper evidence. Products that claim
regulatory attestation or modification detection need a separately specified
integrity, signing, and verification layer.

## Conformance profiles

### Skill Declaration extension

A conforming skill author:

- uses only string metadata values;
- treats `outcomes` as declarations rather than evidence;
- does not place secrets, tokens, cost, receipts, budgets, permissions, or run
  state in metadata;
- chooses stable outcome identifiers.

### Receipt producer

A conforming receipt producer:

- emits receipts only from a trusted completed successful operation;
- supplies a stable `type`;
- bounds and sanitizes subject and data;
- does not report model usage or harness lineage as producer-owned evidence.

### Receipt Core harness

A conforming Receipt Core harness:

- admits receipts only from successful tools;
- caps and bounds receipt work per tool result;
- contains validation and storage failure without changing the tool result;
- exposes originating run and session correlation.

### Receipt Core required test vectors

A conforming Receipt Core implementation should prove at least:

1. A successful tool records a valid `payment.authorized` receipt.
2. A failed tool records no success receipt.
3. A malformed receipt is omitted with a diagnostic.
4. Exact type filtering returns the originating run and session.
5. Oversized receipt data is omitted without changing the successful tool
   result.
6. Two local agents write to one configured store and an all-agent exact-type
   count returns both records.
7. A trajectory reference contains the receipt ID and correlation but no
   producer `data`; `get` resolves the full data from the receipt store.
8. Repeating one harness source identity with equal content is idempotent;
   different content produces a conflict.
9. Session or trajectory rotation does not delete the canonical receipt.
10. A database with a newer unsupported schema is rejected with an actionable
    health diagnostic and no fallback store is created.
11. List pagination is stable when multiple receipts share an occurrence time,
    and count does not materialize producer data.
12. The maximum admitted receipts from one result are committed as one bounded
    batch; overflow is ignored with an observable diagnostic and cannot extend
    lock wait per omitted receipt.

### Later extension test vectors

An implementation claiming a later profile should additionally prove:

1. **Skill Declaration:** a skill without v1 metadata and a skill with unknown
   metadata load normally; malformed recognized metadata does not become
   permission; `isolation: required` rejects before dispatch when unavailable.
2. **Managed Skill Identity:** pre-dispatch rejection creates no managed-run
   record; an accepted call has one native run and child session; parent lineage
   survives configured retention; completion and cancellation use native state.
3. **Run Accounting:** failed and retried attempts remain in reported usage;
   missing usage or cost remains absent; explicit managed run IDs are
   deduplicated for caller-owned token decisions; expired run-to-skill identity
   is reported unavailable rather than reconstructed.

## Example: support work thread

An email channel maps a provider conversation to a stable OpenClaw session. A
support skill declares `customer.verified` and `case.resolved`. The verification
and case tools emit those receipts only after their respective operations
succeed. OpenClaw records the exact skill on the native child run and retains
its model usage, cost, and originating session.

An operator can later filter `case.resolved`, count resolutions by skill digest,
inspect a resolution code in receipt data, and reopen the originating session.
No separate CRM schema is required for that retained operational history. A
producer may place an external case id in `subject` when a separate system owns
the authoritative case.
