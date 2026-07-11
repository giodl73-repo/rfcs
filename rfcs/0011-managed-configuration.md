---
title: Managed Configuration
authors:
  - Gio Lodi
created: 2026-07-10
last_updated: 2026-07-10
status: draft
issue:
rfc_pr: https://github.com/giodl73-repo/rfcs/pull/1
---

# Proposal: Managed Configuration

## Summary

Add a core configuration-authority model in which a host supplies a sparse,
read-only managed configuration and an operator supplies the normal OpenClaw
configuration. OpenClaw validates both through its existing schema, rejects
conflicts or attempts to weaken managed boundaries, and produces one effective
configuration with inspectable provenance. This gives Docker, Kubernetes,
managed hosting platforms, and future OCC deployments a supported alternative
to generated files, environment rewrites, and host-specific post-processing.

## Motivation

OpenClaw currently has one ordinary configuration authority. A hosting platform
that must enforce deployment posture while preserving operator customization
therefore has to generate config fragments, project environment variables,
rewrite files in a particular order, remove stale generated values, and keep
custom logic synchronized with OpenClaw's schema.

The missing distinction is not a new `hosting` config section. It is authority
over fields in the existing configuration schema:

```text
OpenClaw defaults
  + host-managed configuration
  + operator configuration
  -> admission and cross-field validation
  -> effective configuration
  -> runtime
```

Without an upstream contract, hosts silently overwrite operator values or carry
private strictness logic. Operators cannot reliably explain why a value is
effective, and support teams cannot reproduce the deployment from declared
inputs. Repeated host patches then track OpenClaw config internals instead of a
stable composition contract.

Managed Configuration also gives OCC a clean future boundary: OCC can compile
admitted desired state into a managed document while OpenClaw remains the owner
of schema validation, field semantics, and effective runtime configuration.

## Goals

- Load one sparse managed document and one operator document using the normal
  OpenClaw configuration schema.
- Let presence in the managed document declare authority over that field.
- Reject conflicts with structured findings instead of silently overwriting.
- Support exact authority for every schema field.
- Support a small closed set of OpenClaw-defined bounded strictness rules.
- Run normal cross-field validation against the composed effective config.
- Expose redacted effective values, provenance, control mode, and stable config
  identity for diagnostics and conformance.
- Keep the managed source immutable through normal operator config APIs.
- Make unsafe startup composition failures visible through status/readiness.
- Allow hosts to replace private config writers with a release-tested contract.

## Non-Goals

- A new top-level `hosting` section containing copies of existing settings.
- General-purpose arbitrary config layering or numeric priorities.
- A policy expression language or host-defined comparison functions.
- Silent managed-value precedence.
- Dynamic OCC reconciliation in the first version.
- Secret delivery, trusted identity, lifecycle, state synchronization, or
  plugin installation.
- Depending on the optional Policy plugin for runtime correctness.
- Making every field support a monotonic "stricter" relationship.

## Proposal

### Inputs and ownership

The first version accepts:

1. one read-only managed configuration document;
2. the ordinary operator configuration document;
3. OpenClaw defaults and existing runtime-derived values where already
   supported.

Both documents use the existing OpenClaw schema. A leaf present in the managed
document activates its schema-defined control rule. A missing managed leaf
remains operator-controlled.

Arrays are whole-field values unless a specific built-in bounded rule applies.
Objects are traversed according to the normal schema; declaring one child does
not implicitly claim unrelated siblings.

### Authority rules

#### Exact authority

The operator must omit the field or provide the same normalized value. A
different value is an admission error.

```json
{
  "path": "gateway.auth.mode",
  "reason": "ControlledByHost",
  "managedValue": "trusted-proxy",
  "operatorValue": "token",
  "control": "exact"
}
```

Exact authority is the default and works for every schema field without adding
field-specific policy logic.

#### Bounded authority

OpenClaw may mark a small closed set of fields with a monotonic comparator. The
operator may preserve or tighten the managed boundary but cannot weaken it.

Initial comparator classes are:

| Comparator | Composition rule |
| --- | --- |
| Allow-set ceiling | Operator set must be a subset of the managed set |
| Deny-set floor | Operator set must be a superset of the managed set |
| Maximum limit | Operator value must be equal or lower |
| Minimum requirement | Operator value must be equal or stricter |
| Required protection | Operator cannot disable it |
| Disabled risky capability | Operator cannot re-enable it |

Comparator assignment and ordering are owned by the OpenClaw schema. Hosts
cannot attach a comparator to an arbitrary field or redefine "stricter."

### Effective configuration

Composition happens once before runtime consumers observe configuration:

```text
parse inputs
  -> schema validation of each source
  -> authority admission
  -> effective composition
  -> existing cross-field validation
  -> immutable effective snapshot
```

Gateway, plugins, tools, sessions, and state modules consume the same effective
snapshot. They do not independently merge managed and operator values.

The effective result has a stable identity derived from normalized, redacted
configuration and source/control metadata. Secret values are never included in
diagnostic output or hashes in a way that exposes plaintext.

### Findings and provenance

Validation returns structured findings suitable for CLI, status, doctor, admin
UI, and automation:

```json
{
  "valid": false,
  "findings": [
    {
      "path": "tools.exec.ask",
      "reason": "WeakerThanManagedRequirement",
      "managedValue": "always",
      "operatorValue": "never",
      "control": "minimum-requirement"
    }
  ]
}
```

Effective inspection reports provenance without exposing secrets:

```json
{
  "path": "tools.alsoAllow",
  "value": ["read", "sessions_list"],
  "authority": "operator",
  "managedBoundary": ["read", "sessions_list", "exec"],
  "control": "allow-set-ceiling"
}
```

At minimum, OpenClaw should support machine-readable operations equivalent to:

- validate managed plus operator inputs;
- inspect the effective configuration;
- explain one path's value, authority, and comparator;
- report the effective configuration identity through status.

Exact command names and loading flags should follow existing config CLI and
Gateway conventions during implementation.

### Mutation behavior

Normal operator config mutation APIs write only the operator document. A write
that would violate managed authority fails before persistence. The managed
source is not writable through those APIs.

Config reload follows the existing hot-reload/restart classification after a
new effective snapshot passes admission. Invalid managed composition does not
partially activate. At startup, failure remains visible and prevents readiness
when the runtime cannot safely operate under the declared host boundary.

### Chaining

The implementation should avoid hard-coding a merge engine that can never
support more than one managed source. A later extension may accept an ordered
chain such as:

```text
platform -> tenant -> agent/team -> operator
```

Each downstream authority may preserve or tighten inherited bounded values and
may not override exact authority. Chaining is not part of the initial product
surface or support promise.

### Policy plugin relationship

Core owns composition, authority metadata, comparators, findings, and startup
enforcement. The optional Policy plugin may reuse the comparator and finding
machinery for diagnostics, repair suggestions, or conformance checks, but core
must not depend on that plugin.

### Conformance

Release tests should cover:

- exact conflicts across representative scalar, object, and array fields;
- every supported bounded comparator and normalization edge case;
- source immutability and operator write rejection;
- redaction and stable effective identity;
- cross-field validation after composition;
- reload/restart behavior;
- invalid-startup readiness/status behavior;
- compatibility when a newer managed document references an unsupported field
  or comparator.

Hosting Profiles may declare Managed Configuration as an optional capability,
but profiles do not own its semantics.

### Implementation sequence

Keep the initial stack small:

1. Add managed/operator documents, exact authority, effective composition,
   structured findings, provenance, identity, and focused docs/tests.
2. Add the first closed set of schema-owned bounded comparators with shared
   tests and optional Policy-plugin reuse.

## Rationale

### Why not generated overlays?

Generated overlays express precedence, not authority. They silently overwrite
conflicts and require every host to reproduce OpenClaw normalization and
cross-field validation behavior.

### Why not a new hosted config section?

The controlled settings already have canonical homes. Copying them into a
hosting section creates two schemas and forces runtime modules to understand
hosting. Authority metadata composes existing fields without changing their
semantic owner.

### Why reject instead of managed-wins?

Silent precedence hides operator intent and configuration drift. Structured
admission makes the conflict actionable and preserves one explainable effective
state.

### Why keep comparators closed?

Arbitrary host comparators become a policy language and make conformance
impossible. OpenClaw can safely promise monotonic composition only where it owns
the field and ordering.

### Why core rather than a plugin?

Configuration authority must apply before optional plugins and runtime modules
activate. A plugin cannot safely be the enforcement dependency for its own
loading configuration or for Gateway startup.

## Unresolved questions

- Which existing OpenClaw config loading API and CLI commands should expose the
  managed source?
- Which fields, if any, should receive bounded comparators in the first release?
- How should schema evolution report a managed field that is unknown to an
  older OpenClaw release?
- Should effective identity include source document identities separately from
  the normalized effective hash?
- Which redacted provenance fields belong in `status` versus a dedicated config
  inspection call?
- What storage and ownership guidance should hosts follow for durable operator
  configuration across container replacement?
