---
title: Auditable Skills
authors:
  - Gio Lodi
created: 2026-07-13
last_updated: 2026-07-16
status: draft
issue:
rfc_pr: https://github.com/giodl73-repo/rfcs/pull/6
---

# Auditable Skills

## Summary

Define a portable foundation for skills whose effects can be understood after a
model run ends. A skill may declare the outcomes it can produce, the other
skills it may use, and its isolation needs. A runtime records the durable
evidence it actually observes, together with business context, exact
managed-run identity, token usage, and cost. These primitives make completed
work searchable and auditable and form a natural stepping stone to workflows
without turning the Agent Skills format into a workflow engine.

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
`payment.authorized`, or `invoice.paid`. OpenClaw records the exact skill on its
existing child-run record and uses the native run ID to join model usage, USD
cost when available, and evidence actually observed in one configurable
receipt store shared by the Gateway's agents. When skills are composed into a
workflow, a runner aggregates that observed usage and applies caller-owned
limits. The current proof uses Lobster; the contract also permits a minimal
OpenClaw core runner or another conforming adapter.

The central invariant is:

> `SKILL.md` declares intent. A Claw or caller supplies policy. The OpenClaw
> harness records facts.

Skills remain useful without a Claw. A Claw strengthens the model by supplying
exact package identity, installed-agent provenance, an allowed skill graph, and
orchestration budget policy. This RFC depends on the composition and lifecycle
boundaries in [RFC 0016: Claws](https://github.com/openclaw/rfcs/pull/27) when a
Claw is present; it does not duplicate Claw installation, update, or removal.

The first workflow milestone does not require a new general step engine.
OpenClaw can use Lobster for typed pipelines and TaskFlow for durable lifecycle,
or provide a small core sequential runner over the same managed-skill contract.
The missing work is to connect those paths to policy-filtered OpenClaw actions,
then make managed skill calls measurable with honest spend and enforceable
limits.

An operator should be able to answer:

- Which exact skill revision ran?
- Who or what invoked it?
- Which native child run performed it?
- How many tokens and US dollars did it consume?
- Was the cost provider-billed or catalog-estimated?
- Which business outcome did its tools prove?
- Which session, outcome subject, and Claw did it belong to?
- Did it remain within its orchestration budget?

For example, an isolated refund skill may produce this run summary:

```json
{
  "managedSkill": {
    "invocationId": "skill_456",
    "skillName": "issue-refund",
    "skillDigest": "sha256:abc123"
  },
  "runId": "run-123",
  "usage": {
    "input": 3200,
    "output": 480,
    "total": 3680
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
skill name, digest, managed invocation ID, run ID, usage, and cost basis are
harness facts. Neither substitutes for the other.

## Goals

- Let successful tool results assert typed, filterable business receipts.
- Record full receipts once in a configurable store shared across local agents.
- Add a portable, optional `SKILL.md` declaration for managed orchestration.
- Record exact skill identity on the native child run and retain parent-run
  lineage.
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

The implementer-facing v1 core contract is captured in
[`0022/auditable-skills-v1-spec.md`](0022/auditable-skills-v1-spec.md). The
runner-neutral composition boundary and optional Lobster-to-core migration are
captured separately in
[`0022/orchestration-runner-v1-spec.md`](0022/orchestration-runner-v1-spec.md).
This RFC remains the design rationale and rollout plan; the sidecar specs are
the concise metadata, receipt, invocation, accounting, runner, and conformance
references.

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
- managed invocation, run, parent run, child session, and Claw identity;
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

The harness records the full receipt once in a configured shared receipt store
and adds:

- record timestamp and ID;
- session and run identity;
- tool name and tool-call ID;
- skill invocation identity when present;
- provider and model correlation from the run.

The ordinary trajectory records only an `audit.receipt.recorded` reference
containing the receipt ID and small correlation fields. It does not duplicate
producer `data`. By default, OpenClaw uses one local
`~/.openclaw/state/receipts.sqlite` across all agents on the Gateway. Operators
can configure another local path—for example, a database dedicated to a team of
agents—while keeping the producer and query contracts unchanged. SQLite is a
single-host profile, not a network-filesystem or multi-host database.

The receipt does not own token usage. Audit projections join it to managed-run
identity and spend through the native run ID. Optional invocation fields on a
receipt are denormalized correlation, not a second join key or lifecycle store.

Malformed receipt data is not recorded as evidence. Receipt recording failure
does not rewrite the underlying tool outcome, but it remains observable as an
audit diagnostic.

### Managed run and exact skill identity

Every accepted managed skill call receives one stable invocation ID, one native
child run ID, and one child session key. The immutable skill descriptor belongs
on OpenClaw's existing child-run record:

```ts
type ManagedSkillDescriptor = {
  invocationId: string;
  skillName: string;
  skillSource?: string;
  skillDigest: string;
  parentRunId?: string;
  declarations?: SkillExecutionHints;
};
```

The native child run already owns running, completion, failure, cancellation,
duration, cleanup, model, and session lifecycle. Auditable Skills does not copy
that state into a parallel invocation state machine. A parent managed
invocation can be resolved through `parentRunId` and the parent run's own
descriptor rather than duplicating `parentInvocationId` on every child.

An invocation rejected before dispatch returns a structured error and creates
no managed-run record. Once dispatch is accepted, `runId` is the canonical join
across the managed descriptor, session usage, trajectory facts, and receipts.

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

Managed child invocation reuses OpenClaw's existing skill snapshot,
`sessions_spawn`, child-session, subagent-registry, and model-selection paths.
It does not create a second executor or ledger. Parent and child calls retain
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

The provider aggregate is preferred when present. Otherwise a simple run total
is input plus output; cache read and cache write remain separate dimensions and
must not be confused with OpenClaw's context-window `totalTokens` snapshot.
Failed, retried, and timed-out attempts are included whenever the provider
reported usage.

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

The native subagent view is a useful first inspection surface for managed-run
identity, input/output usage, cache snapshot, and estimated cost. Workflow
limits require the cumulative observed usage for each contributing run; they
must not use the context-window `totalTokens` snapshot as spend or treat a
missing cumulative value as zero.

The implementation should carry cumulative usage through the terminal native
run lifecycle or completion result and snapshot it on the exact retained run
record. `runId` is the accounting identity. A later session turn or steer
replacement must not overwrite or inherit an earlier run's usage. Exact
historical USD remains optional until its amount and basis are captured with
the run; a mutable session estimate is not a durable cost receipt.

### Workflow accounting and limits

OpenClaw does not introduce a second workflow budget ledger. A completed
managed skill run projects its observed usage, cost, and unambiguous model
identity into a normalized step result. The selected runner owns workflow
aggregation and limit enforcement. A native usage check may aggregate an
explicit set of completed managed `runId` values and apply a caller-owned token
ceiling between steps; it is a decision primitive, not a stored budget. The
Lobster proof maps the same run result into its native command-result shape,
where `CostTracker` owns durable aggregation and `cost_limit` owns enforcement.
A conforming core runner can consume either boundary without Lobster.

Accounting follows the workflow lifecycle:

1. Each completed managed skill step reports its observed run usage once.
2. The selected runner aggregates those step results into one workflow summary.
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
- join outcomes to managed-run identity, parent and descendant runs, model
  identity, normalized usage, captured cost, and workflow limit state;
- compare a skill's declared `outcomes` with observed evidence to find runs
  where expected evidence is missing or an unexpected outcome was recorded.

For example, an operator could find every `payment.refunded` outcome for a
particular subject, count refunds by skill revision, or audit completed runs
that declared `customer.notified` but recorded no matching evidence. Version 1
does not automatically treat a missing declared outcome as a failed run; it
makes the discrepancy visible to policy and reporting layers.

The first implementation projects existing trajectory and session facts while
resolving full outcome evidence from the shared receipt store. This is one
purpose-built business-evidence store, not a duplicate CRM or workflow ledger.
Public CLI, Gateway, and UI surfaces may evolve independently around the same
record contracts.

The first product surface should make receipts feel like a normal OpenClaw
resource rather than a special trajectory filter. Exact command names are not
normative, but a useful CLI shape is:

```text
openclaw receipts --type case.resolved
openclaw receipts --count --type invoice.paid
openclaw receipts --id <receipt-id>
```

The CLI, Gateway API, plugins, and workflow runners should call the same
storage-neutral `get`, `list`, and `count` boundary. None should open SQLite or
scan session databases directly.

### From a run to a durable work history

Consider an OpenClaw agent supporting customers over email. The channel maps an
email conversation to a stable session, so each thread already has durable
identity and ordered history.

The support skill may declare `customer.verified` and `case.resolved` as
possible outcomes. Those declarations help the caller plan and govern the
work, but they do not claim that either result occurred. If the agent verifies
the customer and resolves the issue, the responsible tools record those
outcomes with their evidence. The harness adds the exact skill revision,
managed child-run identity, model usage, cost, and workflow accounting facts.

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

### Operational readiness

The initial local SQLite profile is useful before it becomes a regulated audit
system, but a production implementation still needs ordinary data-store
operations:

- explicit retention and bounded cleanup independent of session rotation;
- schema-version checks and atomic migrations;
- health diagnostics for an inaccessible, locked, corrupt, or newer database;
- backup and export through consistent snapshots rather than live file copies;
- access control and redaction appropriate for evidence such as authorization
  codes;
- bounded receipt size, query limits, and stable pagination.

OpenClaw Doctor should report store health and actionable recovery guidance. A
store failure remains contained from the completed tool result, but it must be
visible; the runtime must not silently fall back to a per-agent store or claim
that evidence was retained when it was not. Tamper evidence and regulatory
attestation remain separate future layers.

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
OpenClaw can use Lobster for advanced steps, branching, approvals, and resume,
or a minimal core runner for static sequential composition; TaskFlow retains
durable identity and lineage, and existing sessions and tool policy retain
execution authority. Both runner paths reuse the receipt and accounting
contract in this RFC rather than introducing a second harness.

### Agent Skills interoperability path

The standards opportunity is intentionally smaller than the OpenClaw product
surface. An Agent Skills proposal would standardize only optional declarative
vocabulary such as `outcomes`, `uses-skills`, and `isolation`, plus the rule
that declarations are not evidence or authority. Receipt storage, session
identity, model accounting, and workflow execution remain harness concerns.

Before proposing community vocabulary, the same example skill should be read
by at least two independent harness implementations or compatibility fixtures.
They should agree on metadata parsing and declared intent while remaining free
to use different receipt stores, invocation engines, and query surfaces. Until
then, implementations may incubate namespaced aliases without placing
OpenClaw-specific orchestration names in portable `SKILL.md` files.

## Implementation plan

The fork series deliberately proved assumptions one step at a time. Five
OpenClaw slices have now been rebuilt as compact, independently reviewable
evidence. The earlier Lobster workflow experiments remain archived evidence,
not a required landing stack:

| Evidence | What it proved |
| --- | --- |
| [OpenClaw #97](https://github.com/giodl73-repo/openclaw/pull/97) | Shared durable receipts, trajectory references, exact query, and count. |
| [OpenClaw #98](https://github.com/giodl73-repo/openclaw/pull/98) | Portable declarations, exact skill digest, and native managed child-run identity. |
| [OpenClaw #100](https://github.com/giodl73-repo/openclaw/pull/100) | One runner-neutral result joining exact native status and durable receipts. |
| [OpenClaw #109](https://github.com/giodl73-repo/openclaw/pull/109) | Cumulative retry-aware usage retained on the exact native run and exposed by the managed result. |
| [OpenClaw #113](https://github.com/giodl73-repo/openclaw/pull/113) | Explicit managed run IDs deduplicated into one usage total with an optional caller-owned between-step token ceiling. |
| [OpenClaw #114](https://github.com/giodl73-repo/openclaw/pull/114) | One host-owned managed-skill dispatch function shared by `sessions_spawn` and future core controllers. |
| [Lobster #1](https://github.com/giodl73-repo/lobster/pull/1) | Accounting continuity across pause and resume. |

Upstream work should proceed in rounds so maintainers can accept the core
boundary without first accepting a workflow engine:

1. **RFC and core receipts.** Review this RFC and sidecars, then land one small
   OpenClaw vertical slice containing trusted tool receipts, the configurable
   shared store, trajectory references, and storage-neutral `get`, `list`, and
   `count`. This round has no skill metadata or workflow dependency.
2. **Managed invocation and result.** Add the optional Agent Skills
   declarations, exact executed-skill identity on the native subagent record,
   parent-run lineage, and one runner-neutral native result after the receipt
   boundary settles.
   Before claiming long-term skill attribution, define retention or export for
   the run-to-skill association alongside the receipt retention claim.
3. **Exact-run accounting.** Carry cumulative observed usage through the native
   completion lifecycle, retain it with the exact run, and expose it as an
   optional result field. Capture cost and basis with the run before presenting
   exact historical USD or enforcing USD limits.
4. **Between-step token budgets.** Aggregate explicit completed managed run IDs
   once, reuse the native session-tree visibility boundary, and optionally
   compare the exact total with a caller-owned ceiling. Missing accounting must
   stop the decision rather than becoming zero. Do not store budgets or claim
   preflight reservation.
5. **Agent-driven sequence proof.** Let a parent agent invoke one managed skill
   at a time with `sessions_spawn`, inspect its native result and required
   receipt types, then check exact accumulated usage before continuing. The
   parent may choose an allowed model per direct step. This proves useful
   composition with existing primitives, but makes no durable workflow or
   automatic restart claim.
6. **Managed dispatch seam.** Extract one non-model, host-owned managed skill
   dispatch function and make `sessions_spawn` call it. The first proof keeps
   trusted skill resolution, managed identity, current-agent and background-run
   restrictions, and native subagent dispatch in one path. It adds no plugin
   permission or workflow state.
7. **Idempotent runners.** Add host-derived workflow, step, and attempt
   idempotency to that boundary. Only then add a deterministic TaskFlow
   controller and optional Lobster adapter. Keep Lobster's pause/resume
   accounting change in its owning repository.

Each round should be reviewable and useful on its own. The current workflow
proof uses Lobster because it already provides the needed advanced lifecycle;
it validates the runner-neutral contract rather than establishing a permanent
hard dependency.

## Reference success scenario

The first release proof should be one repeatable support-email fixture rather
than a broad workflow showcase:

1. A provider conversation maps to a stable OpenClaw session.
2. A trusted verification tool records `customer.verified` with a subject and
   authorization code in the shared receipt store.
3. An authorized later run, including one owned by another local agent, finds
   that receipt by exact type or subject without scanning the original session
   database.
4. A resolution tool records `case.resolved`; its trajectory contains only the
   receipt reference.
5. An operator lists both outcomes, counts resolutions across agents, shows the
   full evidence by receipt ID, and reopens the originating session.
6. The workflow proof gates resolution on the observed verification receipt and
   reports the child runs' token usage once.

The proof succeeds only if full producer `data` exists in the receipt store,
not in the trajectory reference, and failed or malformed tool results create no
success evidence.

## Acceptance criteria

The first complete series is successful when:

1. A successful tool can emit `payment.authorized` with an authorization code.
2. Failed tools and malformed receipts produce no success evidence.
3. Receipts are recorded, sanitized, retained, and filterable by type.
4. Multiple local agents can share one configured receipt database, and
   trajectory rotation does not duplicate or delete its full receipt payloads.
5. A skill can declare outcomes, other skills it may use, and isolation intent
   in standard-compatible string metadata.
6. Ordinary skills without the metadata remain backward compatible.
7. One accepted managed call records exact skill identity on its native child
   run without copying child lifecycle into a second state machine.
8. One declared isolated child skill runs through existing OpenClaw session
    and policy primitives with native child and parent-run lineage.
9. Each isolated run reports normalized tokens and captured cost with its basis
    when available.
10. Shared turns are labelled shared rather than divided among skills.
11. A workflow total counts each contributing managed run once.
12. Workflow accounting survives approval and structured-input pauses.
13. A caller-provided workflow cost limit is enforced by the selected runner;
    the current Lobster proof uses its existing `cost_limit`.
14. Skill metadata cannot set or widen that limit.
15. When a Claw is present, reports include authoritative Claw and skill package
    revision identity from Claw provenance.
16. The reference support scenario can list, count, and show full receipts
    across local agents and return their originating session correlation.
17. Store health, unsupported schema, retention, and backup behavior are
    documented before the SQLite profile is presented as production-ready.
18. If managed-run identity expires before a retained receipt, readers report
    identity as unavailable rather than reconstructing it from prose.
19. A caller can name completed managed run IDs, count each once, and receive a
    token-limit decision without creating another budget or usage ledger.
20. A parent agent can sequence managed skills using native results, receipt
    gates, per-step allowed models, and exact token decisions without a new
    executor.
21. A durable TaskFlow runner cannot land until non-model dispatch reuses the
    same managed `sessions_spawn` admission path rather than duplicating it.

## A natural stepping stone to workflows

This proposal deliberately stops short of defining workflows, but it establishes
the reusable primitives a workflow layer would otherwise need to invent:

- `outcomes` provides names for expected completion results;
- `uses-skills` provides potential composition edges between skills;
- `isolation` provides honest execution and accounting boundaries;
- managed child-run identity and parent-run lineage identify each execution;
- observed receipts provide evidence for completion gates;
- normalized usage and workflow limits provide spend controls.

Before a deterministic runner exists, a parent OpenClaw agent can provide the
smallest useful composition profile: invoke one managed skill, read its native
result and receipts, check exact accumulated usage, and decide whether to
invoke the next skill. That works with the existing invocation lifecycle and
the primitives in this series. It does not claim a durable workflow identity,
automatic replay, or restart-safe next-step dispatch.

OpenClaw does not need a second invocation lifecycle, workflow receipt, usage,
or session store. The configured receipt store remains the one canonical source
of full outcome evidence. TaskFlow already owns durable flow identity and linked
child tasks. Once a trusted host dispatch seam can invoke managed skills through
the existing admission path, a minimal core runner can cover static sequential
dependencies, failure, cancellation, and totals. Lobster can remain the
advanced runner for typed
pipelines, conditions, retries, branching, approvals, and resume. Both consume
the same managed-skill result and reuse the same session, evidence, usage, cost,
limit, and outcome primitives.

Fan-out, joins, model selection, reservations, and computed gates can evolve in
those existing layers. They remain runtime capabilities, not portable
`SKILL.md` metadata.

## Prior art and dependencies

- [Agent Skills specification](https://agentskills.io/specification)
- [OpenTelemetry GenAI token usage conventions](https://github.com/open-telemetry/semantic-conventions/blob/main/docs/gen-ai/gen-ai-metrics.md)
- [RFC 0016: Claws](https://github.com/openclaw/rfcs/pull/27)
- [Auditable Skills v1 core specification](0022/auditable-skills-v1-spec.md)
- [Orchestration runner v1 addendum](0022/orchestration-runner-v1-spec.md)

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
