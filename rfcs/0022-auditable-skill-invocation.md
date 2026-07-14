---
title: Skill Receipts and Orchestration
authors:
  - Gio Lodi
created: 2026-07-13
last_updated: 2026-07-14
status: draft
issue:
rfc_pr: https://github.com/giodl73-repo/rfcs/pull/6
---

# Proposal: Skill Receipts and Orchestration

## Summary

Add a small, portable skill declaration and an OpenClaw-native evidence model
so operators can see which exact skill ran, what its tools proved, how many
tokens and US dollars it consumed, and whether it stayed within a shared
orchestration budget, without introducing a second workflow engine.

## Motivation

A skill may declare the outcomes it can produce, the other skills it may use,
and whether it requires an isolated run. A successful tool call may
emit a typed receipt such as `inventory.sent`, `payment.authorized`, or
`invoice.paid`. OpenClaw records the exact skill invocation, child-run lineage,
model usage, USD cost when available, budget consumption, and receipts actually
observed.

The central invariant is:

> `SKILL.md` declares intent. A Claw or caller supplies policy. The OpenClaw
> harness records facts.

Skills remain useful without a Claw. A Claw strengthens the model by supplying
exact package identity, installed-agent provenance, an allowed skill graph, and
orchestration budget policy. This RFC depends on the composition and lifecycle
boundaries in [RFC 0016: Claws](https://github.com/openclaw/rfcs/pull/27) when a
Claw is present; it does not duplicate Claw installation, update, or removal.

The first orchestration milestone is not a general step engine. It is one
audited, measurable child skill call with honest spend and enforceable limits.

An operator should be able to answer:

- Which exact skill revision ran?
- Who or what invoked it?
- Which model calls and child runs did it create?
- How many tokens and US dollars did it consume?
- Was the cost provider-billed or catalog-estimated?
- Which business outcome did its tools prove?
- Which session, business record, and Claw did it belong to?
- Did it remain within its orchestration budget?

For example, an isolated refund skill may produce this run summary:

```json
{
  "skill": {
    "name": "issue-refund",
    "digest": "sha256:abc123"
  },
  "invocationId": "inv-456",
  "runId": "run-123",
  "usage": {
    "inputTokens": 3200,
    "outputTokens": 480,
    "cacheReadTokens": 1200,
    "totalTokens": 4880
  },
  "cost": {
    "usd": 0.0184,
    "basis": "catalog-estimate"
  },
  "receipts": [
    {
      "type": "payment.refunded",
      "data": {
        "authorizationCode": "REF-9482"
      }
    }
  ]
}
```

The authorization code is business evidence supplied by the payment tool. The
skill name, digest, invocation ID, run ID, usage, and cost basis are harness
facts. Neither substitutes for the other.

## Goals

- Let successful tool results assert typed, filterable business receipts.
- Add a portable, optional `SKILL.md` declaration for managed orchestration.
- Record exact skill identity, invocation lifecycle, and parent/child lineage.
- Attribute tokens and USD cost honestly at the model-run boundary.
- Aggregate orchestration spend without counting a run more than once.
- Enforce a shared root budget across isolated descendant skill runs.
- Associate sessions and receipts with one optional primary business record.
- Reuse OpenClaw tools, sessions, trajectories, usage normalization, model cost,
  child sessions, policy, sanitization, and state accessors.
- Reuse Claw package identity and provenance when a Claw owns the agent.

## Non-goals

- A business schema registry in OpenClaw core.
- A payment ledger, inventory system, CRM, or invoice state machine.
- Inferring receipts from model prose.
- Assigning an invented portion of a shared model turn to each skill it read.
- Letting skill metadata grant tools, credentials, models, or permissions.
- Letting a skill set its own authoritative budget.
- A new general workflow language or executor.
- Ordered steps, expressions, conditions, loops, joins, retries, or parallel
  fan-out in the first implementation.
- Replacing Agent Skills, Claws, sessions, trajectories, plugins, or child
  agents.

## Proposal

### Three ownership layers

The design has three layers with different trust and lifecycle boundaries.

#### Skill declaration

`SKILL.md` describes capability and intent:

- outcomes the skill intends to produce;
- other skills it may request;
- whether managed execution should be isolated.

The declaration is reviewable package data. It is not evidence that an outcome
occurred, and it cannot widen runtime authority.

#### Claw or caller policy

A Claw or direct caller owns execution policy:

- the installed skill packages and exact versions;
- the allowed parent/child graph;
- token and optional USD limits;
- model choices when a later policy surface supports them;
- whether declaration mismatches warn or fail.

When a Claw is present, OpenClaw uses RFC 0016 package and installed-agent
provenance rather than inventing another bundle or dependency identity. A
standalone skill invocation receives equivalent local policy from its caller or
OpenClaw configuration.

#### Harness evidence

OpenClaw records what actually happened:

- canonical skill source, version when known, and content or package digest;
- invocation, parent invocation, run, parent run, session, and Claw identity;
- provider, model, normalized tokens, captured USD cost, and cost basis;
- status, duration, errors, and child runs;
- receipts actually emitted by successful tools;
- active session business context;
- budget charges and exhaustion.

The declaration and the evidence remain separate so audit consumers can compare
expected and observed behavior.

### Portable skill execution hints

OpenClaw follows the [Agent Skills specification](https://agentskills.io/specification),
which permits an optional string-valued `metadata` map. Skill authors should
not need to learn OpenClaw's internal orchestration or receipt vocabulary. This
RFC proposes three small, implementation-neutral hints:

```yaml
---
name: issue-refund
description: Verify a refund request and issue an approved customer refund.
metadata:
  outcomes: "payment.refunded"
  uses-skills: "verify-customer check-refund-policy"
  isolation: "required"
---
```

All values remain strings, as required by Agent Skills. `outcomes` and
`uses-skills` are whitespace-separated lists because receipt type identifiers
and skill names cannot contain spaces. Other implementations may ignore these
hints or implement the same behavior.

The harness may normalize the hints internally:

```ts
type SkillExecutionHints = {
  outcomes?: string[];
  usesSkills?: string[];
  isolation?: "shared" | "preferred" | "required";
};
```

Unknown keys and values do not prevent ordinary skill discovery or instruction
loading. Each hint can evolve independently without placing a JSON schema
inside YAML frontmatter.

These names are proposed as Agent Skills community vocabulary, not as ownership
claims over the global metadata namespace. During incubation, implementations
may accept namespaced aliases for compatibility. The author-facing target is
the direct vocabulary above.

#### Outcome declarations

`outcomes` lists business outcomes the skill intends to produce during
successful managed execution. It is useful for planning, inspection, and
comparing declared behavior with observed evidence.

It does not create a receipt, mark a run successful, or authorize the model to
claim that the event occurred. Actual receipts must still come from completed
successful tool calls.

A run that finishes without a declared receipt may report the mismatch. Version
1 does not automatically turn that mismatch into a failed business operation.
Strict receipt gates belong with later ordered-step semantics.

#### Child skill declarations

`uses-skills` lists the skill names this skill may request as managed children.
It is deliberately different from a package `requires` field: a possible child
call is not necessarily an installation dependency or prerequisite. The
effective child set is the intersection of:

- the parent skill declaration;
- Claw or caller policy;
- skill visibility and model-invocation policy;
- normal tool, sandbox, credential, and model restrictions.

Metadata can narrow authority. It cannot make a hidden or prohibited skill
invocable.

#### Isolation declarations

`isolation` has these meanings:

- `shared`: the skill may use the current run; usage and cost remain shared.
- `preferred`: use an isolated child run when supported.
- `required`: reject managed invocation when OpenClaw cannot create an isolated
  child run.

Isolation is the honest accounting boundary. A child run can own its model
usage exclusively; several skills used within one model turn cannot.

#### Fields that do not belong in skill metadata

Skill metadata must not contain:

- actual tokens, cost, receipts, authorization codes, or run status;
- credentials or resolved secret values;
- permission grants;
- authoritative token or USD budgets;
- claims that an external mutation succeeded;
- runtime-generated model choices or lineage.

Those values are caller policy or harness evidence and become stale or unsafe
when self-declared by a skill package.

### Receipt contract

The producer-owned receipt remains small and generic:

```ts
type SkillReceipt = {
  type: string;
  version?: number;
  subject?: {
    type: string;
    id: string;
  };
  data?: Record<string, unknown>;
};
```

`type` is the primary business filter and should be namespaced enough to remain
meaningful outside one tool, such as `payment.authorized` rather than
`completed`. `version`, `subject`, and `data` belong to the producer's schema.
OpenClaw does not interpret their business meaning.

The harness records the receipt in an existing trajectory envelope and adds:

- record timestamp and ID;
- session and run identity;
- tool name and tool-call ID;
- skill invocation identity when present;
- active `regarding` snapshot when present;
- provider and model correlation from the run.

The receipt does not own token usage. Audit projections join it to run-level
usage and spend through the invocation and run identity.

Malformed receipt data is not recorded as evidence. Receipt recording failure
does not rewrite the underlying tool outcome, but it remains observable as an
audit diagnostic.

### Session business context

The OpenClaw session remains the conversation or activity stream. One optional
primary association identifies the business record that the session is about:

```ts
type SessionRegarding = {
  system: string;
  type: string;
  id: string;
  key?: string;
};
```

The identity is `system`, `type`, and `id`. Optional `key` is a human-facing
reference such as a case or invoice number. Core treats the fields as opaque.

For example, a mail plugin may correlate an email thread to an OpenClaw session,
match or create a Dataverse case, and set:

```json
{
  "system": "dataverse",
  "type": "incident",
  "id": "500xx0000012345",
  "key": "CASE-18427"
}
```

Channel thread identity and business record identity remain separate. Subject
text is not an identity. Plugins and tools own matching and external creation;
OpenClaw owns session persistence, audited set/replace/clear transitions, and
receipt snapshots.

This follows the Dataverse Set Regarding pattern without importing a CRM object
model into core. `SkillReceipt.subject` remains the object a particular event
concerns and is not an alias for the session's primary `regarding` value.

Regarding is useful for support, finance, sales, and operations, but it is not a
prerequisite for standalone receipts or skill invocation.

### Invocation and exact skill identity

Every explicit managed skill call receives one stable invocation ID. An
isolated child additionally receives parent invocation and parent run IDs.

Runtime evidence should identify the exact executed artifact:

```ts
type ExecutedSkillIdentity = {
  name: string;
  source: string;
  version?: string;
  digest: string;
};
```

The installed package version and digest are authoritative when available. A
self-declared version is descriptive only. Workspace skills use a canonical
source identity and content digest.

When a Claw owns the installed agent, the run also records:

```ts
type ClawExecutionIdentity = {
  clawId: string;
  clawVersion?: string;
  clawDigest: string;
};
```

This identity comes from RFC 0016 install provenance. It lets reports compare
cost and outcomes by exact skill and Claw revision without putting runtime data
back into `SKILL.md` or the Claw manifest.

Managed child invocation reuses OpenClaw's existing child-session and gateway
agent paths. It does not create a second executor. Parent and child calls retain
normal policy and do not widen tool, sandbox, credential, or model access.

### Spend accounting

People need to know how much a skill costs. Spend is therefore part of the first
orchestration milestone, not a later analytics feature.

#### Run ownership

Model usage belongs to the run that consumed it:

- an isolated child skill owns its run usage exclusively;
- an inline skill used during a shared model turn does not receive an invented
  fraction of that turn;
- the containing run reports shared usage when exclusive attribution is not
  available;
- an orchestration total sums each contributing run exactly once.

Receipts correlate with spend through invocation and run IDs. They do not own
tokens or cost.

#### Normalized tokens

OpenClaw reuses its normalized provider usage buckets:

- input;
- output;
- cache read;
- cache write;
- total.

The provider aggregate is preferred when present. Otherwise total usage is the
sum of the normalized billable buckets. Failed, retried, and timed-out attempts
are included whenever the provider reported usage.

#### USD cost

OpenClaw reuses existing model cost calculation and provider-reported cost:

```ts
type RunCost = {
  usd: number;
  basis: "provider-billed" | "catalog-estimate" | "mixed";
};
```

- `provider-billed` means the provider supplied an authoritative billed total.
- `catalog-estimate` means OpenClaw applied configured model pricing to the
  normalized usage.
- `mixed` applies only to an aggregate containing more than one basis.

The amount and basis are captured at execution time. Historical audit output
must not silently change when model catalog pricing changes later. Cost is
omitted when neither provider billing nor a usable catalog estimate exists.

An orchestration report should include per-run, per-model, and total spend so a
Claw can show which skill revisions produced which outcomes at what cost.

### Shared orchestration budgets

A budget belongs to one isolated root skill run and covers its managed
descendants. The canonical root child session owns the durable counter.

```ts
type OrchestrationBudget = {
  schemaVersion: 1;
  rootRunId: string;
  tokenLimit: number;
  tokensUsed: number;
  createdAt: number;
  updatedAt: number;
  exhaustedAt?: number;
};
```

Version 1 enforces a token limit. USD spend remains available in the run and
orchestration projections described above, including its billing or estimate
basis. A hard USD limit may be added once mixed billed/estimated enforcement
semantics are accepted; reporting cost does not wait for that decision.

The budget contract is:

1. The caller or Claw establishes one limit on the root isolated invocation.
2. Descendants inherit an opaque owner-session and root-run reference.
3. Descendants cannot redefine or increase the limit.
4. Each completed model attempt charges its full observed usage atomically.
5. A call that crosses the remaining limit is recorded in full.
6. Exhaustion stops the next model or managed child-skill action.

A budget cannot predict the exact size of one provider call, so it cannot
prevent a final overshoot. It prevents additional spend after the overshooting
call is observed. Initial enforcement is sequential; reservations for parallel
fan-out are outside version 1.

The durable counter is the enforcement source of truth. Retention-bounded
trajectory summaries remain audit and reporting views, not the hard budget
ledger. Accounting failure in a managed budgeted run must stop before another
paid action rather than silently drifting from observed spend.

### Claw integration

RFC 0016 defines a Claw as one complete installed agent with exact package
references and durable provenance. This RFC adds runtime measurement without
changing that ownership model.

When a Claw owns the agent:

- installed skill package identity supplies authoritative version and digest;
- Claw provenance supplies the agent and Claw execution identity;
- Claw or local operator policy may restrict the allowed skill graph;
- Claw or local operator policy may establish root token and cost limits;
- audit summaries group invocations, spend, and receipts by Claw revision.

The effective runtime policy remains an intersection with ordinary OpenClaw
policy. Installing a Claw or declaring `uses-skills` never grants new tools,
credentials, channel access, models, or child skills.

The initial RFC 0022 implementation does not require a new portable RFC 0016
manifest field. A later Claw runtime-policy field may be added through the Claws
schema process after the standalone invocation and budget contracts settle.

### Query and reporting

The stable contract is a versioned record and filter model, not a particular
prototype CLI flag. OpenClaw should expose:

- receipts filtered by business type, subject, run, session, tool, time, and
  `regarding` identity;
- run summaries joining invocation lifecycle, receipts, model identity, usage,
  and captured cost;
- orchestration summaries joining unique parent and descendant runs;
- budget state showing limits, actual consumption, and exhaustion;
- Claw and skill revision filters when provenance is available.

The first implementation may project existing trajectory and session data.
It does not require a separate business ledger. Public CLI, Gateway, and UI
surfaces may evolve independently around the same record contracts.

### Failure behavior

- A failed tool call emits no success receipt.
- A malformed receipt is not recorded as business evidence.
- Receipt recording failure does not rewrite the tool's success or failure.
- An invalid `regarding` value does not change the session association.
- Replacing or clearing `regarding` records an audited transition.
- Unknown skill execution hints do not break ordinary skill loading.
- A managed child request outside the effective declared and allowed graph is
  rejected before dispatch.
- `isolation: required` fails before dispatch when isolation is unavailable.
- Usage is omitted when the provider reports none; it is never invented.
- Cost is omitted when neither billed nor estimated cost is available.
- A stale budget root reference cannot charge another root's counter.
- An exhausted budget prevents the next paid or managed child action.
- Sanitization and redaction apply before durable recording and export.

## Rationale

The design separates declarations from evidence because skill packages are not
trusted witnesses for their own outcomes or cost. Tool receipts provide domain
evidence, while the harness provides execution identity and spend. A Claw or
caller remains the correct owner for limits and policy.

Run-level accounting is the smallest honest attribution boundary. Assigning
tokens directly to a receipt or to every skill read during a shared turn would
produce precise-looking but false numbers. Isolated child sessions already give
OpenClaw an exclusive execution boundary, so the proposal reuses them.

The metadata is a namespaced Agent Skills extension rather than a new manifest.
Standalone skills remain portable, implementations may ignore the extension,
and Claws continue to own packaging and lifecycle. A later standards proposal
can promote the fields after real interoperability proof.

Finally, the proposal stops at one measurable, budgeted child call. Ordered
steps and richer orchestration can reuse the same primitives later if demand
justifies them; they do not need to be accepted to make skills auditable now.

## Implementation plan

The prototype series deliberately proved assumptions one at a time. An
upstream-shaped implementation should consolidate them into coherent vertical
slices rather than preserve every intermediate state.

### 1. Record and query typed receipts

Carry typed receipts through successful tool results, record them in existing
trajectory data, and support the first business-type query. Prove
`payment.authorized` with an authorization code and prove that failed calls
record no success receipt.

Consolidated proof: [giodl73-repo/openclaw#88](https://github.com/giodl73-repo/openclaw/pull/88).

### 2. Add audited session regarding

Set, replace, clear, and read one primary session association; audit real
changes. Snapshot the active association onto later receipts and support exact
identity filters.

Consolidated proof: [giodl73-repo/openclaw#89](https://github.com/giodl73-repo/openclaw/pull/89).

### 3. Consume skill execution hints in explicit invocation

Parse and normalize `outcomes`, `uses-skills`, and `isolation` with a real
consumer. Record explicit invocation lifecycle and exact skill identity. Do
not land an inert metadata contract with no production path.

Consolidated proof: [giodl73-repo/openclaw#90](https://github.com/giodl73-repo/openclaw/pull/90),
including the production consumer for the friendly string hints and the exact
full skill digest.

### 4. Invoke a declared child skill with lineage

Run one named declared child through existing child-session primitives. Record
parent invocation and run lineage and enforce `isolation: required`.

Consolidated proof: [giodl73-repo/openclaw#91](https://github.com/giodl73-repo/openclaw/pull/91).

### 5. Report run and orchestration spend

Report normalized tokens and captured USD cost per run, model, skill revision,
and orchestration. Preserve shared versus exclusive attribution and aggregate
each run once.

Consolidated proof: [giodl73-repo/openclaw#92](https://github.com/giodl73-repo/openclaw/pull/92),
including captured USD cost and provider-billed, catalog-estimated, or mixed
cost basis.

### 6. Enforce one shared root budget

Create the root owner, inherit it through descendants, charge complete observed
attempts, and stop before the next model or child-skill action once exhausted.
USD spend continues to come from the run projection in slice 5.

Consolidated proof: [giodl73-repo/openclaw#93](https://github.com/giodl73-repo/openclaw/pull/93),
including durable ownership, atomic charging, overshoot accounting, and both
admission boundaries.

Direct tool-dispatch parity, `skill.used` read diagnostics, richer query
presentation, and ordered steps can follow independently. They are not required
to validate the first managed child-call contract.

## Acceptance criteria

The first complete series is successful when:

1. A successful tool can emit `payment.authorized` with an authorization code.
2. Failed tools and malformed receipts produce no success evidence.
3. Receipts are recorded, sanitized, retained, and filterable by type.
4. A session can be associated with a case or other opaque business record.
5. Regarding set, replace, and clear transitions are audited.
6. Later receipts snapshot and filter by the active regarding identity.
7. A skill can declare outcomes, other skills it may use, and isolation intent
   in standard-compatible string metadata.
8. Ordinary skills without the metadata remain backward compatible.
9. One explicit invocation records exact skill identity and lifecycle.
10. One declared isolated child skill runs through existing OpenClaw session
    and policy primitives with complete parent/child lineage.
11. Each isolated run reports normalized tokens and captured USD cost with its
    basis when available.
12. Shared turns are labelled shared rather than divided among skills.
13. An orchestration total counts each contributing run once.
14. One durable root budget is inherited by descendants and cannot be widened.
15. A completed call is charged fully, including failed attempts and overshoot.
16. Exhaustion stops the next model or managed child action.
17. When a Claw is present, reports include authoritative Claw and skill package
    revision identity from Claw provenance.

## Later orchestration

After the first managed child-call and budget contracts are accepted, OpenClaw
may add a small durable ordered-step object. A step may reference a declared
skill and observed receipt, but it must reuse the same invocation, session,
usage, cost, and budget primitives.

Expressions, conditions, retries, model overrides, parallel fan-out, joins,
loops, reservations, and computed gates remain later work. They are not
prerequisites for useful receipts, measurable skills, or budgeted child calls.

## Prior art and dependencies

- [Agent Skills specification](https://agentskills.io/specification)
- [RFC 0016: Claws](https://github.com/openclaw/rfcs/pull/27)
- [Dynamics 365 Set Regarding](https://learn.microsoft.com/en-us/dynamics365/outlook-app/user/track-message-or-appointment)
- [Dataverse Email and RegardingObjectId](https://learn.microsoft.com/en-us/power-apps/developer/data-platform/reference/entities/email)
- [Automatic email-to-case creation](https://learn.microsoft.com/en-us/dynamics365/customer-service/administer/automatically-create-case-from-email)
- [Email reply correlation and automatic case creation](https://learn.microsoft.com/en-us/troubleshoot/dynamics-365/customer-service/email/incoming-email-not-converted-case)

## Unresolved questions

- Should `outcomes`, `uses-skills`, and `isolation` be proposed as Agent Skills
  community vocabulary after implementation proof, or incubate under temporary
  namespaced aliases first?
- What canonical digest represents a mutable workspace skill across platforms?
- Where should local Claw runtime graph and budget policy live without making
  model or credential choices portable package data?
- When is catalog-estimated cost sufficiently stable for hard USD enforcement,
  and how should mixed billed and estimated runs behave?
- Which session lifecycle transitions preserve or clear `regarding`?
- When should a declared-versus-observed receipt mismatch become a strict
  managed-run failure rather than an audit warning?
