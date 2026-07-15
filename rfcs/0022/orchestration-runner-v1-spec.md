# Auditable Skills Orchestration Runner v1 Addendum Specification

This document is the implementer-facing orchestration-runner addendum for RFC
0022. It builds on `auditable-skills-v1-spec.md` and defines how OpenClaw can
compose auditable managed skill runs without requiring Lobster or binding the
core contract to any one workflow engine.

Status: draft addendum, tied to RFC 0022.

## Scope

This addendum defines:

- the boundary between OpenClaw and an orchestration runner;
- minimum workflow and step identity and lifecycle;
- managed skill step requests and result envelopes;
- pause, resume, cancellation, and accounting invariants;
- a minimal OpenClaw core runner profile;
- a Lobster adapter profile;
- capability discovery, conformance, and migration.

This addendum does not define:

- a portable workflow format in `SKILL.md`;
- a new general expression language;
- fan-out, joins, loops, or dynamic graph mutation in the core profile;
- a second receipt, session, task, usage, cost, or policy store;
- a requirement to install Lobster;
- replacement of advanced Lobster pipeline behavior.

## Design invariant

OpenClaw core owns the execution facts. A runner owns workflow decisions.

| OpenClaw core | Runner |
| --- | --- |
| Skill resolution and effective policy | Step ordering and dependency readiness |
| Managed child dispatch | Branching and retry policy when supported |
| Invocation and run lineage | Pause and resume orchestration |
| Tool receipts and session correlation | Workflow-level status |
| Observed model usage and captured cost | Aggregation and limit decisions |

The shared result envelope lets a core runner, Lobster, or a future plugin
consume the same managed-run facts. A runner must not parse assistant prose to
manufacture successful outcomes or usage.

## Compatibility

- Direct managed skill invocation does not require a runner.
- An installation without Lobster may use the core runner profile.
- An installation with Lobster may continue to use Lobster through an adapter.
- A runner may support capabilities beyond this addendum without changing the
  core result envelope.
- Unknown runner capabilities must not be assumed.
- A workflow requiring an unavailable capability must fail validation before
  dispatch.

## Runner capabilities

A runner exposes a stable id and explicit capabilities.

```ts
type SkillRunnerCapabilitiesV1 = {
  runnerId: string;
  sequential: true;
  dependencies: boolean;
  conditions: boolean;
  retries: boolean;
  approvals: boolean;
  durableResume: boolean;
  structuredInput: boolean;
  parallel: boolean;
  loops: boolean;
};
```

The v1 core runner profile requires only `sequential`, `dependencies`,
cancellation, failure propagation, and accounting. Lobster may advertise
conditions, retries, approvals, durable resume, structured input, and other
capabilities it actually supports.

## Normalized workflow plan

The normalized plan is an OpenClaw runtime contract, not portable Agent Skills
metadata. A caller, Claw-local policy, UI, plugin, or adapter may produce it.

```ts
type SkillWorkflowPlanV1 = {
  schemaVersion: 1;
  planId: string;
  revision?: string;
  steps: SkillWorkflowStepV1[];
  limits?: {
    maxInputTokens?: number;
    maxOutputTokens?: number;
    maxTotalTokens?: number;
    maxCostUsd?: number;
  };
};

type SkillWorkflowStepV1 = {
  id: string;
  skill: string;
  needs?: string[];
  input?: Record<string, unknown>;
  requiredReceiptTypes?: string[];
};
```

Step ids must be unique within a workflow. `needs` must reference existing
steps and the graph must be acyclic. The core runner may execute only one ready
step at a time. It must not infer conditions or data mappings from prose.

`input` is explicit caller-selected structured input. Implementations must
apply existing secret, size, and policy handling. `requiredReceiptTypes` is an
optional completion gate: a completed managed run satisfies it only when its
recorded receipts contain every exact type. A declaration in `SKILL.md` does
not satisfy this gate.

Example:

```yaml
schemaVersion: 1
planId: support-resolution
revision: "1"
steps:
  - id: verify
    skill: verify-customer
  - id: resolve
    skill: resolve-case
    needs: [verify]
    requiredReceiptTypes: [case.resolved]
  - id: notify
    skill: notify-customer
    needs: [resolve]
```

The YAML is illustrative. This addendum standardizes the normalized semantics,
not a file extension or authoring syntax.

`planId` identifies the logical workflow definition. The runtime assigns a
unique `workflowId` to each execution. Before dispatch, it validates the plan,
computes or records an immutable plan revision or digest, and binds that value
to the execution. Resume must use the same validated plan revision.

Implementations must bound step count, dependency count, input size, receipt
gate count, string length, and graph-validation work. Plan validation must not
perform blocking network I/O on the Gateway event loop.

## Workflow and step lifecycle

```ts
type WorkflowStatusV1 =
  | "pending"
  | "running"
  | "waiting"
  | "completed"
  | "failed"
  | "cancelled";

type StepStatusV1 =
  | "pending"
  | "ready"
  | "running"
  | "waiting"
  | "completed"
  | "failed"
  | "skipped"
  | "cancelled";
```

The core profile uses these transitions:

- Workflow: `pending -> running -> completed | failed | cancelled`.
- Step: `pending -> ready -> running -> completed | failed | cancelled`.
- A downstream step remains `pending` until all `needs` are `completed`.
- A required receipt mismatch makes the owning step `failed` with a structured
  reason.
- When a step fails, the core profile fails the workflow and does not dispatch
  remaining steps.
- Cancellation prevents new dispatch and requests cancellation of the active
  managed run through existing OpenClaw primitives.
- Cancellation received before dispatch must settle without starting a skill;
  cancellation while waiting for completion must reach the active managed run.

`waiting` and `skipped` are available to adapters with approval, input,
condition, or resume semantics. The core profile need not produce them in v1.

Terminal workflow and step states must settle once. Repeated cancel, resume, or
completion delivery must be idempotent.

## Managed step request

For each ready skill step, the runner asks OpenClaw to invoke one managed skill.

```ts
type ManagedSkillStepRequestV1 = {
  schemaVersion: 1;
  workflowId: string;
  stepId: string;
  attempt: number;
  skill: string;
  parentSessionKey: string;
  input?: Record<string, unknown>;
  requiredReceiptTypes?: string[];
};
```

OpenClaw resolves the exact skill artifact, effective child policy, isolation,
model, tools, credentials, and sandbox at dispatch time. A runner may request a
skill by name but must not bypass those checks.

`attempt` starts at 1 and increases for runner-requested retries. The tuple
`workflowId`, `stepId`, and `attempt` is an idempotency key for dispatch. A
runner retry must not silently reuse an earlier failed attempt's invocation id.

## Managed step result

OpenClaw returns a normalized result derived from the core Auditable Skills
records.

```ts
type ManagedSkillStepResultV1 = {
  schemaVersion: 1;
  workflowId: string;
  stepId: string;
  attempt: number;
  invocationId: string;
  runId: string;
  status: "completed" | "failed" | "cancelled";
  skill: {
    name: string;
    source: string;
    version?: string;
    digest: string;
  };
  receipts: Array<{
    type: string;
    version?: number;
    subject?: { type: string; id: string };
    data?: Record<string, unknown>;
  }>;
  usage?: {
    inputTokens?: number;
    outputTokens?: number;
    cacheReadTokens?: number;
    cacheWriteTokens?: number;
    totalTokens?: number;
  };
  cost?: {
    usd: number;
    basis: "provider-billed" | "catalog-estimate" | "mixed";
  };
  error?: {
    code: string;
    message: string;
  };
};
```

The result must contain observed receipts only. It must not substitute declared
`outcomes`. Usage and cost are omitted when unavailable.

Error `code` is a stable machine-readable value. Error `message` is sanitized
for the runner's trust boundary. Raw local paths, credentials, provider payloads,
and unredacted exception text must remain in appropriately protected local
diagnostics rather than crossing through this result.

## Accounting

OpenClaw supplies observed run facts. The active runner owns workflow totals and
limit decisions through the following invariants:

1. Each distinct contributing `runId` is counted at most once.
2. Retries with distinct run ids remain incurred spend.
3. A pause or process restart must not discard already counted runs.
4. Resume must not count a previously accepted run again.
5. Cancellation reports spend already incurred.
6. Missing usage or cost remains absent, not zero.
7. Mixed cost basis is reported when an aggregate combines billed and estimated
   costs.
8. Limits come from the caller or Claw policy, never skill metadata.

A runner that cannot persist accounting across a pause must reject workflows
requiring durable resume before dispatch.

The core runner may keep its workflow state in OpenClaw's existing TaskFlow and
runtime-state primitives. It must not create another model-usage or receipt
ledger.

## Pause and resume

An adapter advertising `durableResume` must persist enough runner-owned state
to recover:

- workflow id and validated plan revision;
- current step and attempt;
- accepted result run ids;
- accumulated usage, cost, and basis;
- pending approval or structured-input request;
- a single-use or revision-bound resume token.

Resume must validate workflow identity and revision before dispatch. Expired,
replayed, deleted, or revision-mismatched resume state must fail without
starting another skill.

OpenClaw core owns managed-run facts referenced by the checkpoint. The runner
owns the checkpoint and decision to continue.

## Core runner profile

The recommended first core implementation is deliberately small:

- validates a static acyclic plan;
- runs one ready managed skill at a time;
- waits for the managed run to settle;
- gates completion on exact observed receipt types when configured;
- stops on failure or cancellation;
- aggregates observed usage and cost once per run;
- records workflow and step status through existing TaskFlow/runtime state;
- exposes inspect and cancel operations.

The core profile does not implement expressions, approvals, structured-input
pauses, loops, fan-out, joins, or arbitrary retry policies. Those capabilities
may remain in Lobster or another runner.

## Lobster adapter profile

Lobster can implement the same boundary without changing its pipeline language.

| Shared contract | Lobster mapping |
| --- | --- |
| Workflow id and status | Lobster run and persisted resume identity |
| Managed step request | Embedded OpenClaw managed-skill action |
| Managed step result | Structured command result and audit projection |
| Observed receipts | Result data used by existing conditions |
| Run usage and cost | Native `CostTracker` contribution |
| Workflow limit | Existing `cost_limit` |
| Waiting and resume | Existing approval and structured-input state |
| Cancellation | Existing cancel and incurred-cost reporting |

The adapter must preserve the normalized result envelope across any Lobster
serialization boundary. It must not require OpenClaw to maintain a parallel
workflow accounting store.

## Runner selection

The caller selects a runner explicitly or accepts a configured default.

```ts
type SkillWorkflowRunnerSelectionV1 = {
  runner?: "core" | string;
  require?: Array<keyof Omit<SkillRunnerCapabilitiesV1, "runnerId">>;
};
```

`core` names the built-in minimal profile. Other ids are implementation- or
plugin-defined, for example a Lobster adapter id. Selection must fail before
dispatch when the chosen runner is unavailable or lacks a required capability.

Audit output records the selected runner id and version when available. The
runner id is execution provenance, not part of `SKILL.md`.

## Migration plan

The migration keeps the existing Lobster proof working while removing Lobster
as a prerequisite for basic composition.

### Phase 1: extract the shared contracts

Move workflow/step identity, the managed step request and result envelope, and
accounting normalization into OpenClaw-owned interfaces. Preserve current
Lobster behavior behind an adapter.

Exit criterion: the existing RFC 0022 support workflow passes through the
adapter without changing its receipts, lineage, or totals.

### Phase 2: add the core sequential runner

Implement the core profile over existing managed invocation and TaskFlow/runtime
state. Support dependencies, exact receipt gates, failure, cancellation, and
accounting only.

Exit criterion: the support workflow runs without Lobster and produces the same
managed-run records and total spend.

### Phase 3: make runner selection explicit

Expose runner capability discovery and selection. Route simple plans to the
core runner. Route plans requiring approvals, structured input, conditions, or
other advanced features to Lobster when installed.

Exit criterion: unavailable capabilities fail validation before any managed
skill starts.

### Phase 4: move shared reporting above adapters

Build workflow audit output from normalized step results and accepted run ids.
Keep runner-specific checkpoint details behind the adapter.

Exit criterion: audit consumers can compare core and Lobster executions through
one workflow/step/run/receipt/usage contract.

### Phase 5: remove the hard dependency

Change the RFC 0022 core implementation and tests so Lobster is optional.
Retain Lobster-specific tests for advanced capabilities and accounting
continuity across its pause/resume paths.

Exit criterion: uninstalling Lobster removes advanced runner capabilities but
does not remove direct managed invocation, receipts, audit queries, or the core
sequential runner.

## Conformance

A conforming runner must prove:

1. It rejects an invalid or cyclic plan before dispatch.
2. It dispatches a step only after all dependencies complete.
3. It uses the existing managed invocation boundary.
4. It gates on observed receipts, not declared outcomes or prose.
5. It counts each contributing run id once.
6. It retains incurred spend from failed and retried attempts.
7. Cancellation prevents new dispatch and reports incurred spend.
8. Repeated terminal delivery is idempotent.
9. Requested unsupported capabilities fail before dispatch.
10. Resume, when advertised, survives a process boundary without duplicate
    dispatch or accounting.

The same fixture should run against every conforming runner profile. A useful
baseline is `verify-customer -> resolve-case -> notify-customer`, with one
accepted receipt path, one missing-receipt failure, one cancellation, and one
accounting assertion.
