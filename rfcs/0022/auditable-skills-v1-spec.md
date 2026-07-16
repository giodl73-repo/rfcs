# Auditable Skills v1 Core Specification

This document is the implementer-facing core specification for RFC 0022,
Auditable Skills. The RFC explains the motivation, ownership model, and rollout
plan. This file defines the portable skill declarations and runtime evidence
contracts that skill authors, harnesses, tools, and audit consumers can build
against.

Status: draft, tied to RFC 0022.

## Scope

This core specification defines:

- optional `SKILL.md` metadata for declared outcomes, child skills, and
  isolation intent;
- typed receipts emitted by successful tools;
- a configurable receipt-store boundary and a shared local SQLite profile;
- managed skill invocation and exact executed-skill identity;
- parent, child, session, and run correlation;
- normalized run usage and captured cost;
- minimum query, sanitization, and failure behavior;
- producer, harness, and audit-consumer conformance.

This core specification does not define:

- a portable workflow language;
- workflow step syntax, branching, retries, approvals, or resume storage;
- package installation or Claw lifecycle;
- model, tool, credential, or child-skill grants;
- CRM accounts, cases, queues, SLAs, or assignment;
- provider pricing catalogs or retention policy;
- tamper-evident logs, signatures, regulatory attestations, or non-repudiation;
- authoritative budgets inside skill metadata.

Workflow-runner integration is defined separately in
[`orchestration-runner-v1-spec.md`](orchestration-runner-v1-spec.md).

## Normative language

The terms **must**, **must not**, **should**, **should not**, and **may** are
normative. An implementation may expose different internal types, storage, and
APIs when its externally observable behavior follows this specification.

## Ownership model

Auditable Skills v1 separates three authorities:

| Owner | Owns | Does not establish |
| --- | --- | --- |
| Skill package | Declared outcomes, possible child skills, isolation intent | Runtime facts, permissions, budgets, or successful effects |
| Claw or caller | Allowed skill graph, model and execution policy, limits | Evidence that an effect occurred |
| Harness | Executed identity, invocation lineage, tool evidence, usage, cost, status | Business meaning beyond producer-defined receipt fields |

An implementation must not treat package metadata as proof that an outcome
occurred. It must not treat metadata as a permission grant.

## Compatibility and evolution

Version 1 uses these compatibility rules:

- Skills without the metadata in this specification remain valid ordinary
  skills.
- Implementations may ignore all three metadata keys.
- Unknown metadata keys must not prevent ordinary skill discovery or
  instruction loading.
- Unknown or malformed values must not break ordinary instruction loading. A
  managed invocation that relies on a malformed recognized field must reject
  that invocation with a clear diagnostic rather than silently weakening the
  requested behavior.
- Receipt producers may add optional fields within `data` without changing the
  core version.
- A breaking change to a core record requires a new `schemaVersion`.
- Audit readers must reject unsupported core schema versions rather than
  reinterpret them as v1.

Implementations may accept namespaced aliases while the metadata vocabulary is
incubating. The author-facing v1 names are the direct names below.

## Skill declaration contract

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

## Receipt contract

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
request-time network I/O. A recorder or sanitizer failure must be contained so
it cannot crash the Gateway or change the tool result.

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

The harness records full receipts through a storage-neutral receipt-store
boundary. The v1 boundary must support idempotent record, get by receipt ID,
exact-filter list, and count operations. It must index exact receipt type and
should index subject, agent, session key, run, invocation, and skill identity.

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
sharing is not supported. A future remote provider may implement the same
store boundary without changing receipt producers or workflow results.

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
- apply forward migrations atomically before accepting writes;
- expose health diagnostics for path, permissions, lock timeout, corruption,
  and unsupported schema without including receipt payloads;
- use a consistent SQLite snapshot mechanism for backup and export rather than
  copying a live database file;
- keep database, journal, and temporary files private to the OpenClaw account.

Disabling new receipt recording must not make existing records unreadable.

## Managed invocation contract

Every explicit managed skill invocation receives one stable invocation ID. An
isolated child also records its parent invocation and parent run when present.

```ts
type ExecutedSkillIdentityV1 = {
  name: string;
  source: string;
  version?: string;
  digest: string;
};

type SkillInvocationV1 = {
  schemaVersion: 1;
  invocationId: string;
  parentInvocationId?: string;
  runId?: string;
  parentRunId?: string;
  sessionId: string;
  sessionKey?: string;
  skill: ExecutedSkillIdentityV1;
  status: "pending" | "running" | "completed" | "failed" | "cancelled";
  startedAt?: string;
  completedAt?: string;
  error?: {
    code: string;
    message: string;
  };
};
```

The installed package identity is authoritative for version and digest when
available. A self-declared version is descriptive only. Workspace skills use a
canonical source identity and content digest.

Invocation state must settle once into `completed`, `failed`, or `cancelled`.
Retries that consume model usage must remain observable as attempts within the
owning run or as separately identified runs. A retry must not erase incurred
usage.

`runId` is absent when invocation is rejected before dispatch. It is required
once a run starts. `startedAt` is required for `running` and later states;
`completedAt` is required for terminal states. A `failed` invocation requires a
stable error code and sanitized message. Terminal state is chosen by one atomic
settlement; a late completion or cancellation callback must not rewrite it.

Existing trajectory invocation events normalize as follows: a started event is
`running`; completion status `success` is `completed`, `error` is `failed`, and
`interrupted` is `cancelled`. Implementations may retain the source status in a
diagnostic field, but audit consumers use the normalized lifecycle above.

Managed invocation reuses ordinary OpenClaw session, policy, sandbox, tool,
credential, and model boundaries. The managed path must not create broader
authority than an equivalent direct run.

## Run usage and cost contract

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

The provider aggregate is preferred when present. Otherwise `total` is
the sum of available normalized billable buckets according to OpenClaw's
provider normalization rules. Failed, retried, and timed-out attempts are
included whenever the provider reported usage.

Cost and its basis are captured with the run. Historical audit output must not
silently change when catalog pricing changes later.

## Audit run projection

An audit consumer should be able to obtain one versioned run projection that
joins invocation identity, observed receipts, model identity, usage, and cost.

```ts
type AuditableSkillRunV1 = {
  schemaVersion: 1;
  sessionId: string;
  sessionKey?: string;
  runId: string;
  firstEventAt: string;
  lastEventAt: string;
  status?: "completed" | "failed" | "cancelled";
  invocations: SkillInvocationV1[];
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
present only when a terminal run status is known and uses the same normalized
lifecycle vocabulary as managed invocations.

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

An implementation conforming as an audit provider must support exact filtering
of observed receipts by `type`. It should additionally support filtering by:

- receipt subject type and id;
- skill name and digest;
- invocation and run id;
- session key;
- tool name and tool-call id;
- time range.

The query surface must return the originating run and session correlation so an
operator or later agent can revisit the work thread. It must distinguish
declared outcomes from observed receipts.

`list` must impose a maximum limit and deterministic ordering with `receiptId`
or store sequence as the final tie-breaker. Cursors are opaque and scoped to
the normalized filter and ordering; a cursor must not be accepted with a
different query. Invalid cursors fail explicitly rather than restarting at the
first page. An empty `agentIds` list matches no agents.

Audit providers should support grouping observed receipts by exact type and
time window as a reporting projection. CLI, Gateway, plugin, UI, workflow, and
export surfaces must reuse the same query boundary rather than opening SQLite
or scanning trajectory files directly.

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

The v1 envelope provides correlation, not tamper evidence. Products that claim
regulatory attestation or modification detection need a separately specified
integrity, signing, and verification layer.

## Conformance

### Skill author

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

### Harness

A conforming harness:

- validates metadata without breaking ordinary skill loading;
- enforces the effective child graph and isolation requirement before dispatch;
- records exact executed-skill identity and invocation lineage;
- admits receipts only from successful tools;
- attributes usage to runs and preserves cost basis;
- exposes originating run and session correlation.

### Required test vectors

A conforming implementation should prove at least:

1. A skill without v1 metadata loads normally.
2. Unknown metadata does not break ordinary loading.
3. `isolation: required` rejects before dispatch when isolation is unavailable.
4. A successful tool records a valid `payment.authorized` receipt.
5. A failed tool records no success receipt.
6. A malformed receipt is omitted with a diagnostic.
7. Exact type filtering returns the originating run and session.
8. Parent and child invocation lineage survives completion and cleanup.
9. Reported provider usage includes incurred failed or retried attempts.
10. Missing usage and unavailable cost remain absent rather than becoming zero.
11. A malformed recognized metadata field rejects managed invocation without
    breaking ordinary skill loading.
12. A pre-dispatch rejection has no `runId`, while a started run does.
13. Oversized receipt data is omitted without changing the successful tool
    result.
14. Competing completion and cancellation callbacks settle one terminal state.
15. Two local agents write to one configured store and an all-agent exact-type
    count returns both records.
16. A trajectory reference contains the receipt ID and correlation but no
    producer `data`; `get` resolves the full data from the receipt store.
17. Repeating one harness source identity with equal content is idempotent;
    different content produces a conflict.
18. Session or trajectory rotation does not delete the canonical receipt.
19. A database with a newer unsupported schema is rejected with an actionable
    health diagnostic and no fallback store is created.
20. List pagination is stable when multiple receipts share an occurrence time,
    and count does not materialize producer data.

## Example: support work thread

An email channel maps a provider conversation to a stable OpenClaw session. A
support skill declares `customer.verified` and `case.resolved`. The verification
and case tools emit those receipts only after their respective operations
succeed. OpenClaw records the exact skill, invocation, model run, usage, cost,
and originating session.

An operator can later filter `case.resolved`, count resolutions by skill digest,
inspect a resolution code in receipt data, and reopen the originating session.
No separate CRM schema is required for that retained operational history. A
producer may place an external case id in `subject` when a separate system owns
the authoritative case.
