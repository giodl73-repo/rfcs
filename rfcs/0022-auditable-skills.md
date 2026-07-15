---
title: Auditable Skills
authors:
  - Gio Lodi
created: 2026-07-13
last_updated: 2026-07-15
status: draft
issue:
rfc_pr: https://github.com/giodl73-repo/rfcs/pull/6
---

# Auditable Skills

## Summary

Define a portable foundation for skills whose effects can be understood after a
model run ends. A skill may declare the outcomes it can produce, the other
skills it may use, and its isolation needs. A runtime records the durable
evidence it actually observes, together with business context, exact execution
lineage, token usage, and cost. These primitives make completed work searchable
and auditable and form a natural stepping stone to workflows without turning
the Agent Skills format into a workflow engine.

## Motivation

An agent handling a customer email should not lose the business context when a
reply is sent or a model run ends. The conversation may continue over several
messages, involve several skills, and produce results such as a verified
customer, an authorized refund, or a resolved case. An operator or a later
agent should be able to find that thread again and understand what actually
happened.

Agent Skills already describes how reusable capabilities are packaged. An
**auditable skill** adds a clean separation between declared capability and
observed effect. The package may say what it intends to accomplish, while the
runtime retains evidence of what actually happened and enough correlation to
find, count, explain, and cost that work later.

This RFC adds a small vocabulary for what a skill may accomplish, which other
skills it may use, and whether its work needs an isolated run. A successful
tool call may then emit typed evidence such as `inventory.sent`,
`payment.authorized`, or `invoice.paid`. OpenClaw records the exact skill
invocation, child-run lineage, model usage, USD cost when available, and
evidence actually observed. When skills are composed into a workflow, Lobster
aggregates that observed usage and applies limits through its existing workflow
accounting primitives.

The central invariant is:

> `SKILL.md` declares intent. A Claw or caller supplies policy. The OpenClaw
> harness records facts.

Skills remain useful without a Claw. A Claw strengthens the model by supplying
exact package identity, installed-agent provenance, an allowed skill graph, and
orchestration budget policy. This RFC depends on the composition and lifecycle
boundaries in [RFC 0016: Claws](https://github.com/openclaw/rfcs/pull/27) when a
Claw is present; it does not duplicate Claw installation, update, or removal.

The first workflow milestone is not a new step engine. OpenClaw already has
Lobster for typed pipelines and TaskFlow for durable lifecycle. The missing
work is to connect those primitives to policy-filtered OpenClaw actions, then
make managed skill calls measurable with honest spend and enforceable limits.

An operator should be able to answer:

- Which exact skill revision ran?
- Who or what invoked it?
- Which model calls and child runs did it create?
- How many tokens and US dollars did it consume?
- Was the cost provider-billed or catalog-estimated?
- Which business outcome did its tools prove?
- Which session, outcome subject, and Claw did it belong to?
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
- Roll observed managed-run usage into workflow totals and existing workflow
  limits.
- Make retained work threads discoverable through existing session identity and
  typed outcome subjects.
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
- workflow usage and limit state when the run belongs to a workflow.

The declaration and the evidence remain separate so audit consumers can compare
expected and observed behavior.

### Portable declarations for auditable skills

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

### Durable outcome evidence

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
- provider and model correlation from the run.

The receipt does not own token usage. Audit projections join it to run-level
usage and spend through the invocation and run identity.

Malformed receipt data is not recorded as evidence. Receipt recording failure
does not rewrite the underlying tool outcome, but it remains observable as an
audit diagnostic.

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

### Workflow accounting and limits

OpenClaw does not introduce a second workflow budget ledger. A completed
managed skill run projects its observed input and output tokens, plus an
unambiguous model identity when available, into Lobster's native command-result
shape. Lobster's existing `CostTracker` owns workflow aggregation and its
existing `cost_limit` owns workflow enforcement.

Accounting follows the workflow lifecycle:

1. Each completed managed skill step reports its observed run usage once.
2. Lobster aggregates those step results into one workflow summary.
3. The summary survives approval and structured-input pauses and resumes.
4. Cancellation reports cost already incurred when resume state is available.
5. The caller may set a workflow limit; skill metadata cannot set or widen it.

This boundary keeps business receipts independent from accounting. A receipt
can be searched and audited by type, while tokens and cost remain attached to
the run and workflow that consumed them. Provider-billed and estimated cost
bases remain visible rather than being collapsed into false precision.

OpenClaw may continue to enforce ordinary agent or descendant-run limits at its
own execution boundary. This RFC does not define a new portable budget schema
or require those policies to share storage with Lobster.

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
schema process after standalone invocation and workflow accounting settle.

### Query and reporting

The stable contract is a versioned record and filter model, not a particular
prototype CLI flag. Stable outcome types turn runtime evidence into ordinary
audit dimensions. An implementation should make it possible to:

- search and filter observed outcomes by exact type, subject, skill, skill
  revision, run, session, tool, and time;
- count and group outcomes by type, status, subject, skill or Claw revision,
  model, and time window;
- join outcomes to invocation lifecycle, parent and descendant runs, model
  identity, normalized usage, captured cost, and workflow limit state;
- compare a skill's declared `outcomes` with observed evidence to find runs
  where expected evidence is missing or an unexpected outcome was recorded.

For example, an operator could find every `payment.refunded` outcome for a
particular subject, count refunds by skill revision, or audit completed runs
that declared `customer.notified` but recorded no matching evidence. Version 1
does not automatically treat a missing declared outcome as a failed run; it
makes the discrepancy visible to policy and reporting layers.

The first implementation may project existing trajectory and session data.
It does not require a separate business ledger. Public CLI, Gateway, and UI
surfaces may evolve independently around the same record contracts.

### From a run to a durable work history

Consider an OpenClaw agent supporting customers over email. The channel maps an
email conversation to a stable session, so each thread already has durable
identity and ordered history.

The support skill may declare `customer.verified` and `case.resolved` as
possible outcomes. Those declarations help the caller plan and govern the
work, but they do not claim that either result occurred. If the agent verifies
the customer and resolves the issue, the responsible tools record those
outcomes with their evidence. The harness adds the exact skill revision,
invocation and child-run lineage, model usage, cost, and workflow accounting
facts.

The session now serves as more than a transcript. It is a retained work thread
that another agent or operator can revisit:

1. find the session by its existing channel and thread identity, or find an
   outcome by its exact type and subject;
2. read the ordered observed outcomes;
3. inspect evidence such as authorization or resolution codes; and
4. trace the skills, model runs, spend, and policy boundaries that produced
   them.

A Claw can package the skills, outcome vocabulary, policies, budgets, and
operator views for that support capability. OpenClaw retains the live
operational history. For teams that need only this level of continuity, the
combination may be sufficient without a separate case-tracking application.
Integrations can put external record IDs in producer-owned receipt subjects
without adding a second session-association model to OpenClaw core.

The proposal does not define accounts, contacts, assignment queues, SLAs,
forms, or authoritative customer data. Long-term revisitability also depends
on explicit retention, backup, and export policy. Those are product and
deployment concerns built on the record contract, not additional `SKILL.md`
metadata.

### Failure behavior

- A failed tool call emits no success receipt.
- A malformed receipt is not recorded as business evidence.
- Receipt recording failure does not rewrite the tool's success or failure.
- Unknown skill execution hints do not break ordinary skill loading.
- A managed child request outside the effective declared and allowed graph is
  rejected before dispatch.
- `isolation: required` fails before dispatch when isolation is unavailable.
- Usage is omitted when the provider reports none; it is never invented.
- Cost is omitted when neither billed nor estimated cost is available.
- Workflow accounting survives pause and resume without double counting.
- A configured workflow cost limit remains caller policy and cannot be widened
  by skill metadata.
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

The metadata uses the Agent Skills string-valued metadata map rather than a new
manifest.
Standalone skills remain portable, implementations may ignore the extension,
and Claws continue to own packaging and lifecycle. A later standards proposal
can promote the fields after real interoperability proof.

Finally, the proposal does not add workflow syntax to Agent Skills metadata.
OpenClaw can use Lobster for steps, branching, approvals, and resume; TaskFlow
for durable identity, status, waits, and lineage; and existing sessions and
tool policy for execution. Managed skill calls can then reuse the receipt and
accounting contract in this RFC rather than introducing a second harness.

## Implementation plan

The prototype series deliberately proved assumptions one at a time. An
upstream-shaped implementation should consolidate them into coherent vertical
slices rather than preserve every intermediate state. The reviewer-facing
series contains this RFC and five implementation slices.

### 1. Record business work

Carry typed receipts through successful tool results and make them searchable
through existing session/run correlation plus exact outcome type. This proves
that an operator can revisit the originating work thread, count outcomes by
type, and inspect evidence such as a payment authorization code.

Consolidated proof: [giodl73-repo/openclaw#97](https://github.com/giodl73-repo/openclaw/pull/97).

### 2. Manage skill invocation

Consume the portable metadata in a real invocation path. Record exact skill
identity, lifecycle, and parent/child lineage while reusing existing OpenClaw
session and policy boundaries.

Consolidated proof: [giodl73-repo/openclaw#98](https://github.com/giodl73-repo/openclaw/pull/98).

### 3. Compose managed skills into a workflow

Let embedded Lobster invoke policy-filtered managed OpenClaw skills. The
support-case proof waits for each child to finish, branches on recorded
evidence, pauses for approval, and passes only selected evidence into the next
skill. Lobster owns pipeline semantics, TaskFlow owns durable flow identity,
and OpenClaw owns managed execution and evidence.

Consolidated proof: [giodl73-repo/openclaw#100](https://github.com/giodl73-repo/openclaw/pull/100).

### 4. Preserve workflow accounting across pauses

Keep Lobster's existing `CostTracker` summary across approval and structured
input pauses, expose the summary through embedded and CLI envelopes, and retain
incurred accounting on cancellation. This adds no new store or pricing model.

Consolidated proof: [giodl73-repo/lobster#1](https://github.com/giodl73-repo/lobster/pull/1).

### 5. Report and constrain workflow spend

Project completed managed-run usage into Lobster's native result convention so
the workflow total includes actual OpenClaw child work. Preserve `_meta.cost`
through the OpenClaw runner and demonstrate the existing workflow
`cost_limit`, without attributing tokens to individual receipts.

Consolidated proof: [giodl73-repo/openclaw#109](https://github.com/giodl73-repo/openclaw/pull/109).

## Acceptance criteria

The first complete series is successful when:

1. A successful tool can emit `payment.authorized` with an authorization code.
2. Failed tools and malformed receipts produce no success evidence.
3. Receipts are recorded, sanitized, retained, and filterable by type.
4. A skill can declare outcomes, other skills it may use, and isolation intent
   in standard-compatible string metadata.
5. Ordinary skills without the metadata remain backward compatible.
6. One explicit invocation records exact skill identity and lifecycle.
7. One declared isolated child skill runs through existing OpenClaw session
    and policy primitives with complete parent/child lineage.
8. Each isolated run reports normalized tokens and captured cost with its basis
    when available.
9. Shared turns are labelled shared rather than divided among skills.
10. A workflow total counts each contributing managed run once.
11. Workflow accounting survives approval and structured-input pauses.
12. A caller-provided workflow `cost_limit` uses Lobster's existing enforcement.
13. Skill metadata cannot set or widen that limit.
14. When a Claw is present, reports include authoritative Claw and skill package
    revision identity from Claw provenance.

## A natural stepping stone to workflows

This proposal deliberately stops short of defining workflows, but it establishes
the reusable primitives a workflow layer would otherwise need to invent:

- `outcomes` provides names for expected completion results;
- `uses-skills` provides potential composition edges between skills;
- `isolation` provides honest execution and accounting boundaries;
- invocation and parent/child lineage identify each execution;
- observed receipts provide evidence for completion gates;
- normalized usage and workflow limits provide spend controls.

OpenClaw does not need a second durable ordered-step object. TaskFlow already
owns flow identity, state, waits, revisions, and linked child tasks. Lobster
already supplies typed JSON pipelines, conditions, retries, branching,
approvals, and resume. A Lobster step can reference an OpenClaw tool today and,
next, a managed skill invocation while reusing the same session, evidence,
usage, cost, limit, and outcome primitives.

Fan-out, joins, model selection, reservations, and computed gates can evolve in
those existing layers. They remain runtime capabilities, not portable
`SKILL.md` metadata.

## Prior art and dependencies

- [Agent Skills specification](https://agentskills.io/specification)
- [RFC 0016: Claws](https://github.com/openclaw/rfcs/pull/27)

## Unresolved questions

- Should `outcomes`, `uses-skills`, and `isolation` be proposed as Agent Skills
  community vocabulary after implementation proof, or incubate under temporary
  namespaced aliases first?
- What canonical digest represents a mutable workspace skill across platforms?
- Where should local Claw runtime graph and budget policy live without making
  model or credential choices portable package data?
- When is catalog-estimated cost sufficiently stable for hard USD enforcement,
  and how should mixed billed and estimated runs behave?
- When should a declared-versus-observed receipt mismatch become a strict
  managed-run failure rather than an audit warning?
