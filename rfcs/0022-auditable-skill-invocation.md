---
title: Auditable Skill Invocation and Managed Skill Runs
authors:
  - Gio Lodi
created: 2026-07-13
last_updated: 2026-07-13
status: draft
issue:
rfc_pr:
---

# Proposal: Auditable Skill Invocation and Managed Skill Runs

## Summary

Add an incremental, OpenClaw-native path from observing skill use to running a
small ordered set of skill steps. OpenClaw first records explicit skill
invocations and model reads without changing execution. Later phases enrich the
invocation record, add one canonical harness invocation primitive, allow one
skill to invoke another, and finally introduce durable sequential skill runs.
Every phase is independently useful and preserves the semantics established by
the phase before it.

## Motivation

Skills are an important OpenClaw extension surface, but today an operator cannot
reliably answer basic questions about their use:

- Was a skill explicitly invoked, or did the model only read its instructions?
- Which installed skill source and content version were used?
- Which session, turn, model, and tools were involved?
- Did one skill request another skill?
- How much usage belongs to the surrounding turn, and when can it be attributed
  to one isolated skill step?
- If several skill actions form one task, which action is current, complete, or
  failed?

Existing transcripts and tool events contain parts of those answers, but they
do not provide a stable skill-level identity. Inferring invocation after the
fact from a file path is also semantically weak: reading `SKILL.md` makes the
instructions available to the model, but it does not prove that the model used
them or that OpenClaw explicitly invoked the skill.

The first requirement is therefore truthful observation, not orchestration.
OpenClaw should distinguish an explicit invocation from a model read, attach
the skill identity already known by the loader, and export the resulting event
through its existing trajectory surface. Once that contract is stable, the
same invocation identity can support parent-child calls and managed steps
without introducing a separate execution model.

This staged approach also avoids prematurely committing OpenClaw to a general
workflow language. Sequential skill runs, isolated child execution, model
selection, budgets, parallelism, and conditions have different correctness and
security requirements. They should be added only when the preceding primitive
has shipped and can be validated independently.

## Goals

- Distinguish explicit skill invocation from model access to skill instructions.
- Give every explicit invocation a stable identity within its session and turn.
- Record the resolved skill name, source, and content version without rereading
  mutable skill state after the event.
- Export skill audit events through OpenClaw's existing sanitized trajectory
  path.
- Correlate explicit invocation with terminal turn outcome, duration, model,
  and usage while describing the attribution scope honestly.
- Establish one harness-owned skill invocation primitive used by explicit user
  commands and future internal callers.
- Allow one skill invocation to request another without widening permissions or
  silently changing execution environments.
- Add a durable, ordered skill-run model only after invocation semantics are
  stable.
- Preserve compatibility at every phase so that an implementation can stop
  after any accepted phase and still provide useful behavior.

## Non-Goals

- Defining a general workflow language.
- Adding expressions, conditional routing, data queries, or computed gates.
- Adding parallel fan-out, joins, loops, or dynamic step generation.
- Assigning exclusive per-skill token usage when multiple skills share one
  agent turn.
- Treating every `SKILL.md` read as proof that the skill was invoked.
- Capturing full prompts, skill inputs, outputs, tool arguments, or secrets by
  default.
- Changing skill discovery, installation, eligibility, precedence, or prompt
  formatting in the first audit phase.
- Allowing skill metadata to widen sandbox, tool, model, or credential policy.
- Replacing sessions, trajectories, background tasks, or child-agent runtime.
- Requiring all phases to ship in one implementation series.

## Proposal

### Design principles

The feature follows five rules.

1. **Observe before managing.** The first phase records existing behavior and
   does not add a new execution path.
2. **Name events truthfully.** An explicit invocation and a model read are
   different facts and remain different event types.
3. **Snapshot identity at the boundary.** Audit records carry the resolved
   source and version known when the skill enters the turn. Export does not
   infer identity from later filesystem state.
4. **Reuse one invocation identity.** User commands, harness calls, nested
   calls, and managed steps converge on the same invocation receipt.
5. **Add one execution capability at a time.** Same-turn calls precede durable
   runs; sequential runs precede isolated steps; isolated steps precede
   exclusive usage attribution and budgets.

```mermaid
flowchart LR
    A["Phase 1\nobserve skill use"]
    B["Phase 2\nenrich invocation receipts"]
    C["Phase 3\ncanonical harness invocation"]
    D["Phase 4\nparent and child skill calls"]
    E["Phase 5\ndurable sequential skill runs"]
    F["Later\nisolation, models, budgets"]

    A --> B --> C --> D --> E --> F
```

Each arrow is a compatibility boundary. A later phase extends the record; it
does not reinterpret events emitted by an earlier phase.

### Phase 1: Observe skill use

Phase 1 adds two audit facts to the existing trajectory event stream.

#### Explicit invocation

`skill.invocation.started` records that OpenClaw explicitly supplied a resolved
skill for use in a turn. Initial explicit sources include a user skill command
and a harness invocation path. The event is emitted before the skill content is
supplied to the turn.

```json
{
  "type": "skill.invocation.started",
  "invocationId": "skillinv_01...",
  "sessionId": "session_01...",
  "turnId": "turn_01...",
  "timestamp": "2026-07-13T20:00:00.000Z",
  "trigger": "user-command",
  "skill": {
    "name": "security-review",
    "version": "sha256:8db4...",
    "source": "workspace"
  }
}
```

The resolved source should use the canonical bounded source vocabulary already
owned by skill loading. The content version should use the existing skill
prompt version when available. Absence of an optional version does not block
the invocation, but it remains visible in the record.

At the terminal boundary of that turn, OpenClaw emits exactly one corresponding
terminal event:

- `skill.invocation.completed`
- `skill.invocation.failed`
- `skill.invocation.cancelled`
- `skill.invocation.timed_out`

The terminal event repeats the invocation identity and records the turn
outcome. Phase 1 does not claim that every tool action between the two events
was caused by the skill.

#### Model access

`skill.accessed` records a successful model-initiated read of a `SKILL.md` that
belongs to the resolved skill snapshot for the current turn.

```json
{
  "type": "skill.accessed",
  "sessionId": "session_01...",
  "turnId": "turn_01...",
  "timestamp": "2026-07-13T20:00:03.000Z",
  "trigger": "model-read",
  "skill": {
    "name": "security-review",
    "version": "sha256:8db4...",
    "source": "workspace"
  }
}
```

This event is observational and does not create an invocation ID. It means that
the instructions were successfully read, not that the model followed them.

A read is eligible for this event only when OpenClaw can match the successful
read to a skill in the current resolved snapshot. An arbitrary path ending in
`SKILL.md` is insufficient. Failed or blocked reads retain their existing tool
events and do not emit `skill.accessed`.

#### Phase 1 persistence and export

Phase 1 writes no new canonical state store. Events use the existing runtime
trajectory path and its retention policy. Sanitized trajectory export includes
the new events and applies the same local-path and sensitive-data redaction as
other trajectory events.

The initial event contains no full skill content, prompt text, tool arguments,
environment values, or file contents. Its purpose is identity and correlation.

#### Phase 1 acceptance criteria

- An explicit user skill command emits one started event and exactly one
  terminal event with the same invocation ID.
- A successful model read of a resolved skill emits `skill.accessed` and does
  not emit an explicit invocation event.
- A failed, blocked, or unrelated file read does not emit `skill.accessed`.
- Skill source and version come from the turn's resolved skill snapshot.
- Trajectory export retains the events while preserving existing redaction.
- Disabling trajectory capture preserves current skill behavior.
- Existing skill prompts and model-visible skill listings remain byte-stable.

### Phase 2: Enrich invocation receipts

Phase 2 adds a terminal `SkillInvocationReceipt` assembled from facts OpenClaw
already knows at stable lifecycle boundaries.

```ts
type SkillInvocationReceipt = {
  invocationId: string;
  sessionId: string;
  turnId: string;
  parentInvocationId?: string;
  skill: {
    name: string;
    version?: string;
    source: string;
  };
  trigger: "user-command" | "harness" | "skill" | "managed-step";
  model?: {
    requested?: string;
    effectiveProvider?: string;
    effectiveModel?: string;
  };
  outcome: "completed" | "failed" | "cancelled" | "timed_out";
  startedAt: string;
  completedAt: string;
  durationMs: number;
  usage?: {
    scope: "turn" | "step";
    attribution: "shared" | "exclusive";
    inputTokens?: number;
    outputTokens?: number;
    cacheReadTokens?: number;
    cacheWriteTokens?: number;
    totalTokens?: number;
  };
  trajectoryRef?: string;
};
```

The shape above is illustrative TypeScript. The implementation should use the
canonical OpenClaw usage and model reference types rather than introduce
parallel token or model vocabularies.

When an invocation shares a normal agent turn, usage is recorded as
`scope: "turn"` and `attribution: "shared"`. It must not be presented as the
exclusive cost of the skill. If several skills are invoked or accessed in one
turn, each explicit receipt may reference the same shared turn usage.

Full inputs and outputs remain outside the default receipt. A later metadata
extension may allow a skill to declare bounded audit labels or named evidence,
but metadata cannot disable the foundational identity and lifecycle events or
request capture of secret values.

#### Phase 2 acceptance criteria

- Receipts are derived from lifecycle facts rather than post-run filesystem
  inference.
- Requested and effective models remain distinguishable when both are known.
- Shared turn usage is never labeled exclusive.
- Missing provider usage produces an omitted or explicitly unavailable usage
  field rather than a zero-cost claim.
- Receipt export applies existing trajectory redaction.

### Phase 3: Add one harness invocation primitive

Phase 3 introduces one canonical harness operation for explicit skill
invocation. The exact public API name is an implementation decision; this RFC
uses `invokeSkill` for clarity.

```ts
await harness.invokeSkill({
  skillName: "security-review",
  additionalInstructions: "Review the current repository.",
  parentInvocationId,
});
```

The operation:

1. resolves the skill from the current eligible skill snapshot;
2. snapshots its canonical identity and prompt version;
3. emits `skill.invocation.started`;
4. uses OpenClaw's canonical explicit skill-invocation formatting;
5. runs in the current session, turn policy, model, sandbox, and tool boundary;
6. emits exactly one terminal event and receipt.

The existing explicit user command should converge on this primitive. OpenClaw
should not maintain separate invocation semantics for user commands and
internal harness callers.

Phase 3 remains same-session and same-turn-policy. It does not spawn a child
agent, change the model, reserve a budget, or persist a managed run.

#### Phase 3 acceptance criteria

- The explicit user command and direct harness call produce the same event and
  receipt contract.
- Resolution fails before invocation starts when the skill is missing,
  ineligible, disabled for the requested surface, or ambiguous.
- Invocation cannot widen the current tool, sandbox, credential, or model
  policy.
- Existing skill formatting remains the single content assembly path.

### Phase 4: Allow parent and child skill calls

Phase 4 allows an active explicit invocation to request another skill through
the canonical harness primitive.

An optional metadata declaration advertises the skills that an author expects
to call:

```yaml
metadata:
  openclaw:
    invokes:
      - repository-inventory
      - security-review
```

`invokes` is declarative permission and discovery metadata. Listing a skill
does not run it. Mentioning another skill in prose also does not create an
executable dependency.

A child call records `parentInvocationId` and uses trigger `skill`. The initial
implementation runs under the same session, effective model, sandbox, and tool
policy as its parent.

```text
skillinv_parent: repository-audit
└── skillinv_child: repository-inventory
```

OpenClaw enforces:

- a bounded maximum invocation depth;
- cycle detection over the active invocation ancestry;
- eligibility and declared-invocation checks;
- no tool, sandbox, credential, or model-policy widening;
- one terminal receipt for every started child invocation.

The initial cycle identity is the resolved skill identity within the active
ancestry. A skill cannot recursively call itself through aliases or repeated
names that resolve to the same installed skill.

#### Phase 4 acceptance criteria

- A child receipt references its parent invocation.
- Undeclared, missing, ineligible, cyclic, or over-depth calls fail before the
  child invocation starts.
- Parent failure or cancellation cannot leave an untracked active child.
- Child calls inherit current execution policy without widening privileges.

### Phase 5: Add durable sequential skill runs

Phase 5 introduces a `SkillRun`: one durable owner for an ordered list of
managed skill steps. This is the first phase that manages progress across
invocations.

A skill may point to a companion run descriptor from its OpenClaw metadata:

```yaml
metadata:
  openclaw:
    run:
      version: 1
      source: skill-run.yaml
```

The first descriptor supports only a static ordered list:

```yaml
version: 1
name: repository-audit
steps:
  - id: inventory
    skill: repository-inventory
  - id: review
    skill: security-review
```

The descriptor does not support conditions, expressions, parallelism, loops,
dynamic steps, per-step models, or budgets in this phase.

OpenClaw validates the complete descriptor before starting the run. Validation
includes unique step IDs, resolvable skills, eligible invocation relationships,
bounded step count, and absence of recursive run references.

The durable model records at least:

```ts
type SkillRun = {
  runId: string;
  ownerSessionId: string;
  rootInvocationId: string;
  descriptorVersion: string;
  status: "pending" | "running" | "completed" | "failed" | "cancelled";
  currentStepId?: string;
  createdAt: string;
  updatedAt: string;
};

type SkillRunStep = {
  runId: string;
  stepId: string;
  position: number;
  skillName: string;
  skillVersion?: string;
  invocationId?: string;
  status: "pending" | "running" | "completed" | "failed" | "cancelled";
};
```

Canonical managed state belongs in OpenClaw's shared SQLite state database.
Trajectory events remain the detailed diagnostic stream and receipts retain
their trajectory references. The runtime must not introduce a second JSON or
JSONL state store for managed progress.

The first failure policy is stop-on-failure. Completed steps remain recorded;
pending steps do not start. Resume may retry the failed step only when an
operator explicitly requests it and OpenClaw can preserve the prior attempt's
receipt.

#### Phase 5 acceptance criteria

- The complete static descriptor validates before the first step starts.
- Steps start in declared order and at most one step runs at a time.
- Every running step has a corresponding explicit invocation and receipt.
- A failed step prevents later steps from starting.
- Restart recovery distinguishes an active step from an interrupted step and
  never silently marks an interrupted invocation successful.
- Status can identify the current step and link to completed receipts.
- Replacing or editing a skill after run start does not rewrite the version
  recorded for an already-started step.

### Later extensions

Later RFC amendments or follow-up RFCs may add:

1. isolated child execution for one step;
2. requested and effective per-step model selection;
3. exclusive step-level usage receipts;
4. run and step token limits;
5. retry policy and idempotency metadata;
6. bounded parallel steps and joins;
7. approval holds;
8. conditional steps and expressions.

These are deliberately ordered. Per-step model and token accounting become
truthful only after a step owns an isolated child run. Parallelism requires
budget reservation, cancellation, and join semantics that sequential execution
does not need. Expressions require a separate language, validation, and data
access contract.

### Security and privacy

Skill audit data can expose user intent and local environment details even when
it contains no prompt text. Implementations must therefore follow the existing
trajectory access, retention, redaction, and export boundaries.

The foundational event records bounded identifiers and lifecycle facts. It
does not record:

- skill content;
- full user prompts or additional instructions;
- tool arguments or results;
- environment values or credentials;
- arbitrary local paths;
- unbounded metadata supplied by a skill.

Local skill paths may be used internally for matching a successful read to the
resolved snapshot, but exported events should identify the skill by canonical
source metadata rather than expose the local path.

Nested invocation and managed-step metadata cannot grant permissions. Runtime
policy is the intersection of the caller's current policy, the target skill's
eligibility, and administrator policy. A child call that requires unavailable
capabilities fails before it starts.

### Compatibility and rollout

Skill authors do not need to change existing `SKILL.md` files for Phases 1-3.
Existing skills receive audit identity when OpenClaw can resolve them from the
current snapshot.

Unknown OpenClaw metadata keys remain subject to the existing forward-
compatibility behavior. Implementations must not expose `invokes` or `run`
metadata to older runtimes as an assurance that nested calls or managed runs
will occur.

The event schema should be additive and versioned through the existing
trajectory schema conventions. Consumers must tolerate unknown event types and
missing optional fields.

Each phase should ship behind its own implementation readiness decision. The
RFC does not require a persistent feature flag once a phase is stable, but an
experimental gate may be appropriate for the first managed-run implementation.

## Rationale

### Why invocation and access are separate

Treating every `SKILL.md` read as an invocation would produce attractive but
false audit results. Models may inspect several skills before selecting an
approach, reread a version after a prompt refresh, or read instructions without
following them. `skill.accessed` preserves the useful observation without
claiming causality.

Explicit invocations have a stronger boundary because OpenClaw intentionally
supplies one resolved skill through a known command or harness operation. That
boundary can support lifecycle events and receipts.

### Why audit events precede metadata

Metadata without runtime evidence describes author intent, not executed
behavior. Starting with events establishes which facts OpenClaw can observe
reliably. Later metadata can constrain or enrich execution without becoming the
source of truth for what occurred.

### Why one harness primitive precedes nested calls

If user commands, internal callers, and managed steps each implement skill
resolution and prompt assembly independently, their audit and security
semantics will drift. Converging them first makes nested invocation a new caller
of an existing primitive instead of a second execution system.

### Why sequential runs precede richer orchestration

A static ordered list is sufficient to prove durable run identity, step
receipts, restart behavior, and failure handling. Parallelism, conditional
routing, loops, and expressions add independent scheduling and safety
questions. Deferring them keeps the initial managed surface small enough to
test against real OpenClaw sessions and trajectories.

### Why shared-turn usage is not divided among skills

Provider usage is generally reported for a request or turn, not for individual
instruction sources inside that request. Dividing shared usage by skill count,
text length, or tool activity would produce invented precision. Exclusive
attribution becomes valid when an isolated step owns the provider run.

### Why managed progress uses SQLite

Audit trajectories and managed state have different jobs. Trajectories provide
detailed diagnostic history. A managed run needs indexed, transactional state
for ownership, current step, cancellation, and restart recovery. OpenClaw's
shared SQLite state database is the canonical location for that state; a new
sidecar state format would create competing recovery and migration semantics.

## Unresolved questions

- Should Phase 1 terminal events attach to the explicit invocation directly or
  reference a canonical turn-terminal outcome record?
- Which existing bounded skill-source vocabulary should appear in exported
  events, and how should plugin-bundled skills be represented?
- Should `skill.accessed` deduplicate repeated reads of the same skill version
  within one turn, or record every successful access?
- What maximum nested invocation depth provides useful composition without
  encouraging recursive agent behavior?
- Should `invokes` be required for every child call or support an administrator
  policy that allows any eligible skill?
- What is the minimum safe restart contract for a same-session managed step
  whose process ended after provider completion but before the terminal receipt
  was committed?
- Should managed run descriptors remain companion files or eventually become a
  separate installed artifact type?
- Which later capabilities belong in amendments to this RFC, and which should
  require separate RFCs?
