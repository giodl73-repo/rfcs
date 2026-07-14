---
title: Skill Receipts and Orchestration
authors:
  - Gio Lodi
created: 2026-07-13
last_updated: 2026-07-13
status: draft
issue:
rfc_pr: https://github.com/giodl73-repo/rfcs/pull/6
---

# Proposal: Skill Receipts and Orchestration

## Summary

Add a small OpenClaw-native receipt primitive, then use it as the foundation
for auditable skill orchestration.

A successful tool call can assert one or more typed receipts such as
`inventory.sent`, `payment.authorized`, or `invoice.paid`. The tool or plugin
owns the meaning and payload of each type. OpenClaw owns execution identity,
timestamps, session and run correlation, model and usage correlation,
sanitization, storage, and query.

The OpenClaw session remains the conversation or activity stream. An optional
`regarding` association identifies the one primary business record that the
session is about, such as a case, opportunity, invoice, shipment, or incident.
Receipts recorded during the session snapshot that association in their
OpenClaw-owned envelope. This follows the useful Dataverse distinction between
an activity and the record it is regarding without importing a CRM object
model into core.

Orchestration comes later. Once receipts can be recorded and queried, a skill
run can use them as evidence that a step completed. OpenClaw can then add
ordered steps, child skill invocation, per-step model selection, and budgets
without introducing a second execution system.

## Motivation

OpenClaw already knows when tools run, which session and model are active, and
how many tokens a provider turn uses. What it does not have is a small domain
fact that says what a successful call accomplished.

A generic tool result can say that an API call returned successfully, but the
useful receipt may be more specific:

- inventory was sent;
- a payment was authorized;
- an invoice was paid;
- a message was accepted by a provider;
- a deployment was created.

For a payment, the provider authorization code is stronger evidence than a
model summary. For inventory, the shipment or transfer identifier is the
useful receipt. OpenClaw should preserve those facts without defining payment,
inventory, or invoicing schemas in core.

Typed receipts also give later orchestration a clean completion boundary. A
step can wait for `payment.authorized` without parsing prose or treating every
successful tool call as equivalent.

## Goals

- Let a successful tool result assert typed, filterable receipts.
- Keep receipt meaning and type-specific data owned by the tool or plugin.
- Let a plugin or tool set one typed primary `regarding` association on an
  existing OpenClaw session.
- Add OpenClaw-owned execution correlation when a receipt is recorded.
- Reuse existing tool results, sessions, trajectories, provider usage, child
  sessions, and model selection primitives.
- Add orchestration one capability at a time after receipt recording is useful
  on its own.
- Track token usage honestly: shared turn usage remains shared; isolated child
  runs may be attributed exclusively.

## Non-goals

- A business schema registry in OpenClaw core.
- An arbitrary external-reference or relationship graph.
- A payment ledger, inventory system, or invoice state machine.
- A new general workflow language.
- Expressions, conditions, loops, joins, or dynamic fan-out in the initial
  implementation.
- Inferring business completion from model prose.
- Claiming exclusive per-skill token usage when several skills share one turn.
- Replacing OpenClaw sessions, trajectories, tools, plugins, or child agents.

## Regarding and receipt contracts

The contracts are intentionally small.

```ts
type SessionRegarding = {
  system: string;
  type: string;
  id: string;
  key?: string;
};

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

`SessionRegarding` is a singular primary association, not a general relation
list. Its identity is the tuple of `system`, `type`, and `id`. `key` is an
optional human-facing business key such as a case, invoice, or order number.
Core treats all four fields as opaque strings.

For example, an email support session may be regarding a Dataverse case:

```json
{
  "system": "dataverse",
  "type": "incident",
  "id": "500xx0000012345",
  "key": "CASE-18427"
}
```

The channel thread and the business record remain separate identities. A mail
plugin may use provider message correlation to select the OpenClaw session,
then use domain criteria to match or create a case and set the session's
`regarding` association. Subject text is not an identity.

`type` is the primary filter key. It should be namespaced enough to remain
meaningful outside one tool, for example `payment.authorized` rather than
`completed`.

`version` belongs to the producer's schema. OpenClaw does not interpret it.

`subject` is the optional producer-owned object that the particular business
event concerns. It is distinct from `regarding`, which is the OpenClaw-owned
snapshot of the session's primary business context. A payment receipt may have
the payment or invoice as its subject while the surrounding session is
regarding an order, case, or customer. The two values may match but are not
aliases and neither is inferred from the other.

`data` contains type-specific evidence. For example:

```json
{
  "type": "payment.authorized",
  "version": 1,
  "subject": {
    "type": "invoice",
    "id": "inv-123"
  },
  "data": {
    "authorizationCode": "auth-456",
    "providerPaymentId": "pay-789"
  }
}
```

The producer supplies the receipt. The harness adds the record envelope:

- record ID;
- timestamp;
- session and run ID;
- tool name and tool-call ID;
- effective provider and model when available;
- the active session `regarding` snapshot when available;
- skill invocation or step ID when available;
- usage reference or usage scope when available.

These correlation fields are harness facts and cannot be supplied by the
receipt producer.

Setting, replacing, or clearing `regarding` is explicit and auditable. The
session stores the current value. Trajectory events record the change, and a
receipt stores the value active when the receipt was emitted so later changes
do not rewrite history.

## Ownership boundary

The tool or plugin owns:

- receipt type;
- schema version;
- subject meaning;
- type-specific evidence;
- the decision that the business event actually occurred;
- criteria for matching or creating an external business record;
- the decision to set, replace, or clear a session's `regarding` association.

OpenClaw owns:

- accepting receipts only from completed successful calls;
- validating and persisting the canonical `regarding` shape on the session;
- auditing `regarding` changes and snapshotting the active value on receipts;
- correlation and timestamps;
- sanitization and redaction;
- retention and export;
- filtering and query;
- later skill-run and step linkage;
- usage scope and token accounting.

This keeps OpenClaw generic while still making sessions and receipts
operationally useful across support, sales, finance, logistics, and operations.

## Prior-art pattern: Dataverse Set Regarding

Microsoft Dataverse stores an email as an activity and uses the polymorphic
`RegardingObjectId` lookup to associate that activity with one primary row.
Dynamics 365 calls the user operation **Set Regarding**. Automatic email-to-case
rules can correlate an incoming reply, match or create a case, and associate the
email activity with that case.

OpenClaw should reuse the pattern, not the Dataverse schema:

- the OpenClaw session is the conversation/activity stream;
- channel-native thread correlation selects that session;
- a plugin or tool applies domain criteria and sets `regarding`;
- receipts and usage remain events within the session and snapshot the active
  association;
- core stores and filters explicit `system`, `type`, `id`, and `key` fields
  rather than requiring a polymorphic business-object registry.

References:

- [Link and track an email or appointment with Set Regarding](https://learn.microsoft.com/en-us/dynamics365/outlook-app/user/track-message-or-appointment)
- [Dataverse email table and RegardingObjectId](https://learn.microsoft.com/en-us/power-apps/developer/data-platform/reference/entities/email)
- [Automatically create a case from an email](https://learn.microsoft.com/en-us/dynamics365/customer-service/administer/automatically-create-case-from-email)
- [Email reply correlation and automatic case creation](https://learn.microsoft.com/en-us/troubleshoot/dynamics-365/customer-service/email/incoming-email-not-converted-case)

## Incremental implementation

### Phase 1: Carry receipts on tool results

Add an optional receipt collection to the existing structured tool result.
Preserve it through after-tool hooks and tool-result middleware.

This phase does not add storage or orchestration. It proves the producer
contract and compatibility boundary.

Prototype: `giodl73-repo/openclaw#66`.

### Phase 2: Record successful receipts

Project valid receipts from successful tool results into OpenClaw's existing
trajectory stream. Record a stable event such as `audit.receipt`; keep the
business receipt type in event data as the primary domain filter.

Trajectory already supplies timestamp, session, run, provider, and model
correlation. The receipt projection adds tool name and tool-call ID. Failed
calls do not produce receipts.

Prototype: `giodl73-repo/openclaw#67`.

### Phase 3: Query receipts

Add a narrow query surface over recorded receipts. The first filters should be:

- receipt type;
- subject type and ID;
- session or run ID;
- tool name;
- time range.

This phase should reuse the existing trajectory or state storage selected by
OpenClaw. It should not introduce a separate business ledger.

Prototype: `giodl73-repo/openclaw#68` begins with receipt-type filtering on the
existing session trajectory query.

### Phase 4: Record skill use

Project OpenClaw's existing trusted `skill.used` diagnostic fact into the
trajectory. Preserve the native distinction between reading skill instructions
and explicitly activating a skill command. Do not record trusted skill file
paths in the public trajectory payload.

Prototype: `giodl73-repo/openclaw#69`.

### Phase 5: Set session regarding

Add a narrow core seam to get, set, replace, and clear the optional primary
`regarding` association on an existing session. Reuse the session accessor and
trajectory storage rather than creating a business-record store.

The association belongs to the logical session entry identified by
`sessionKey`, not to one model run or one rotating transcript `sessionId`.

The operation records a trajectory event such as `session.regarding.set` or
`session.regarding.cleared`. A plugin, tool, or skill owns the criteria and the
external mutation; OpenClaw owns only the canonical association and its audit
history.

The proving scenario is an email session associated with a case, but no
support-specific field or case behavior belongs in core.

### Phase 6: Correlate receipts, skills, regarding, and usage

When OpenClaw has an explicit skill invocation identity, attach it to receipts
produced during that invocation. Record the effective model and link existing
provider usage. Snapshot the active `regarding` value on each receipt envelope
and allow queries by `regarding.system`, `regarding.type`, `regarding.id`, and
`regarding.key`.

Usage must declare its scope:

- `shared` when several skills or actions use the same agent turn;
- `exclusive` only when a step runs in an isolated child session whose usage
  can be measured independently.

This phase enables reporting token spend by run without inventing token data.

### Phase 7: Manage ordered steps

Add a small durable run object with ordered steps. A step may complete from a
tool result and its receipts. Initial step execution is sequential.

A step records:

- step ID and run ID;
- requested skill or action;
- expected receipt type when applicable;
- status and terminal outcome;
- child session ID when isolated;
- model and usage scope;
- emitted receipt references.

No expressions or arbitrary conditions are required for this phase.

### Phase 8: Invoke another skill

Allow one managed step to invoke another skill through the same OpenClaw
session and child-agent primitives used elsewhere. The harness may select a
different model for an isolated child step and records its usage separately.

Parent and child calls do not widen tool, sandbox, credential, or model policy.

### Later: Budgets and richer orchestration

Once isolated step usage is reliable, a run or step may set a token budget.
Budget enforcement should reuse OpenClaw's existing usage normalization and
goal/session budget primitives.

Expressions, parallel fan-out, joins, retries, and computed gates can be
considered later. They are not prerequisites for useful receipts or ordered
skill runs.

## Failure behavior

- A failed tool call emits no success receipt.
- A malformed receipt is ignored rather than recorded as evidence.
- Receipt recording failure must not rewrite the underlying tool outcome.
- An invalid `regarding` value is rejected without changing the current session
  association.
- Replacing or clearing `regarding` records an explicit audited transition.
- A step waiting for a receipt remains incomplete if the expected receipt was
  not recorded.
- Usage is omitted or marked shared when exclusive attribution is unavailable.
- Redaction applies before durable recording and export.

## Acceptance criteria

The receipt and regarding implementation series is successful when:

- a tool can return `payment.authorized` with an authorization code;
- after-tool hooks and middleware preserve that receipt;
- a successful call records a correlated receipt event;
- a failed call records no success receipt;
- recorded receipts can be filtered by their business `type`;
- a plugin or tool can set and read one primary `regarding` association on a
  session;
- replacing or clearing that association is audited;
- subsequent receipts snapshot the active association and can be filtered by
  its explicit fields;
- existing trajectory sanitization and retention apply;
- no new business-specific schema or workflow engine is added.

The first orchestration series is successful when:

- a durable run can execute a small ordered step list;
- a step can reference receipts emitted during its execution;
- a child skill can run through existing OpenClaw child-session primitives;
- model and token usage are recorded with honest shared or exclusive scope;
- a token budget can stop an isolated step without changing receipt semantics.

## Unresolved questions

- Should the public tool-result property be named `receipts`, `audit`, or
  `records`?
- Should the first query surface read trajectory exports directly or project
  receipts into OpenClaw's shared SQLite state?
- Should the first `regarding` mutation seam be a session accessor capability,
  a sessions command/API operation, or both?
- Which session lifecycle transitions should preserve or clear `regarding`?
- Which explicit skill invocation boundary should provide the first stable
  skill invocation ID?
- Which receipt fields require field-level redaction beyond existing payload
  sanitization?
