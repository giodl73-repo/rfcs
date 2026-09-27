# Host Integration Readiness v1 Specification

This document defines how RFC 0020 host integration contracts participate in
OpenClaw readiness. It depends on the canonical condition and provider contract
in [RFC 0018: Readiness Conditions and
Providers](https://github.com/openclaw/rfcs/pull/33). It may be composed by the
optional profiles in [RFC 0023: Standard Hosting
Profiles](https://github.com/openclaw/rfcs/pull/37).

Status: draft addendum, tied to RFC 0020 and dependent on RFC 0018.

## Scope

This specification defines:

- how bundle declarations refer to owner readiness criteria;
- how owner activation snapshots become canonical conditions;
- who owns success, failure, requirement, and selection;
- generation and cache invalidation rules;
- allowed plugin-provider use;
- bundle, credential, policy, and dispatcher readiness behavior;
- Status and Doctor separation; and
- conformance fixtures.

It does not define:

- another readiness condition schema or evaluator;
- a standard Hosting Profile;
- fleet-global readiness;
- liveness, restart, placement, rollout, or scheduler policy;
- live credential, provider, or network probes;
- host-written owner success.

## Core Invariant

Readiness is an observation of owner authority, not a host activation API.

```text
owner prepares and activates one binding generation
  -> owner publishes one immutable activation snapshot
  -> core-owned criterion adapter reads that snapshot
  -> RFC 0018 emits True, False, or Unknown
  -> operator selection or an opt-in profile classifies requirement
  -> every readiness projection consumes the same canonical result
```

The bundle, external host, dispatcher, readiness provider, Hosting Profile, and
fleet aggregator cannot override the semantic owner's result.

## Dependencies

RFC 0018 owns:

- `ReadinessCondition` and `ReadinessResult`;
- `True`, `False`, and `Unknown`;
- required and advisory aggregation;
- criterion identity and selection;
- bounded evaluation, cancellation, coalescing, and caching;
- HTTP, health, status, and CLI projection.

RFC 0023 may select RFC 0020 criteria in an operator profile. RFC 0020 does not
add host integration to the standard profile catalog.

## Terminology

- **Owner snapshot**: immutable, bounded evidence from one semantic owner's
  current activation.
- **Criterion adapter**: core-owned read-only mapping from an owner snapshot to
  one canonical readiness condition.
- **Criterion id**: stable owner-defined identity declared by a bundle
  contribution and resolved through the owner's contract.
- **Requirement**: RFC 0018 `required` or `advisory` classification selected by
  an operator or profile.
- **Current evidence**: evidence whose owner, bundle, policy, and binding
  generations still match active authority.

## Ownership

The semantic owner owns:

- criterion meaning;
- the activation predicate;
- success, non-ready, and unknown reasons;
- owner-generation comparison;
- whether the current binding can accept work.

The external host plugin owns only observations about facts it controls, such
as whether its external dispatcher process is admitted. It cannot claim that a
model provider, Channel, mailbox, credential binding, or request policy is
ready.

Readiness core owns:

- evaluation bounds and cancellation;
- requirement and aggregation;
- result ordering and projection;
- invalid-result and timeout conversion;
- cache invalidation.

The operator or selected Hosting Profile owns whether a selectable criterion
is required or advisory.

## Bundle Criterion Declarations

Each bundle contribution contains `readinessCriteria`, a bounded list of exact
RFC 0018 selector ids.

The field:

- declares which owner criteria the contribution may affect;
- resolves against the active RFC 0018 criterion catalog;
- remains metadata until owner configuration selects and activates the
  contribution;
- does not register executable provider code;
- does not select the criterion;
- does not make the criterion required;
- does not report success.

The bundle contribution `required` field controls registration completeness
and membership in the explicitly selected `openclaw.host-bindings-ready`
aggregate. It never changes an individual referenced criterion's RFC 0018
`required` or `advisory` classification.

Core-owned criteria use an RFC 0018 `openclaw.<criterion-id>` selector.
Plugin-owned observational facts use
`plugin.<plugin-id>.<criterion-id>`. A plugin criterion becomes resolvable only
after the activated plugin registry publishes its provider descriptor.
Selectors are enumerable without evaluating providers.

The manifest rejects non-canonical ids, recursive
`openclaw.host-bindings-ready` references, more than 64 selectors on one
contribution, or more than 64 unique selectors across the bundle. Unknown
owner/kind pairs remain invalid even if their selectors are syntactically
valid.

An unknown selector remains visible as `unresolvedReadinessCriteria` in
host-integration Status and Doctor. If the contribution is required and its
owner otherwise reports ready, the aggregate `HostBindingsReady` condition
fails closed with `RequiredCriterionUnknown`. An unresolved selector on an
optional contribution remains visible without blocking the aggregate.

## Canonical Condition Mapping

Criterion adapters emit the RFC 0018 shape:

```ts
type ReadinessCondition = {
  type: string;
  status: "True" | "False" | "Unknown";
  requirement: "required" | "advisory";
  reason: string;
  message: string;
};
```

The adapter determines status:

- `True`: the selected contribution is resolved, its owner binding is
  activated for current generations, and the owner expects it to accept work.
- `False`: the owner authoritatively observed a non-ready state, such as an
  incompatible selected contribution, denied policy, or unavailable required
  physical route.
- `Unknown`: current truth cannot be established, including activation in
  progress, stale generations, missing snapshots, bounded evaluation timeout,
  or unavailable observation state.

An absent observation is never `True`.

`degraded` is not a readiness status. Reduced service that still accepts the
workload remains `True` and is explained in Status or Doctor. Reduced service
that violates the serving contract is `False` or `Unknown` according to owner
evidence. A transient per-operation overload does not flap readiness by itself;
the owner changes its activation snapshot only when it no longer expects to
accept work.

The requirement field is applied by RFC 0018 after operator or profile
selection. An owner, bundle, or host cannot self-promote its condition to
required.

### Condition Ordering

RFC 0020 extends the RFC 0018 condition order with one bucket for remaining
core-owned host-integration conditions. The bucket follows
`GatewayResponding` and `PluginsLoaded` and precedes remaining plugin
conditions. Conditions within it sort by `selectorId`.

## Optional Generation Metadata

Authenticated or local projections may attach:

```ts
type HostIntegrationReadinessMetadata = {
  owner: string;
  kind: string;
  contributionId: string;
  ownerGeneration: string;
  hostBundleGeneration: string;
  policyGeneration?: string;
  bindingIncarnation?: string;
};
```

This metadata is diagnostic. RFC 0018 consumers must be able to ignore it.
Values are bounded and redacted.

When RFC 0023 projects `RuntimeActivationSummary.hostIntegrationGeneration`
from RFC 0020 evidence, the value is the current `hostBundleGeneration`.
Owner, policy, and binding generations remain separate diagnostic fields.

Evidence is current only when every applicable generation and incarnation
matches active authority. A mismatch emits `Unknown`; it must not reuse a
prior `True` value.

## Invalidation

Readiness cache entries for affected criteria invalidate immediately when:

- owner configuration generation changes;
- host bundle generation changes;
- selected contribution identity or version changes;
- traffic-policy generation changes;
- dispatcher admission or binding incarnation changes;
- plugin activation generation changes;
- operator criterion selection changes.

Late evidence from an invalidated generation is discarded. Readiness polling
does not trigger owner activation, re-admission, token acquisition, or
dispatch.

## Bundle Readiness

Bundle registration may provide a criterion for the plugin-owned fact that the
complete required inventory published successfully.

That condition cannot override owner criteria:

- a valid bundle does not imply any owner binding is ready;
- one missing required contribution makes bundle publication unavailable;
- an optional missing contribution remains visible without becoming success;
- bundle removal invalidates all dependent owner evidence;
- same-plugin re-registration creates a new generation and invalidates prior
  results.

Bundle readiness participates only when selected. Advisory selection is
recommended unless bundle availability is part of the deployment serving
contract.

Selecting `openclaw.host-bindings-ready` adds the bundle's referenced criteria
to the same canonical result as advisory detail. It does not promote those
criteria to required. An operator or Hosting Profile may independently promote
a referenced selector through the ordinary RFC 0018 selection rules. The
aggregate's own semantics use bundle `required` to identify its required
members. Every required member must first have current authoritative owner
activation evidence for the active owner and bundle generations, independent
of its declared selectors; missing or stale owner evidence fails the aggregate.
The aggregate also fails when a referenced criterion on one of those
contributions reports `False` or `Unknown`. Plugin-owned observations may add
detail but cannot substitute for owner activation evidence. The detailed
criterion remains advisory unless separately promoted. This use of `required`
changes the aggregate result, not the RFC 0018 requirement classification of
the detail.

## Credential Slot Readiness

Credential-slot readiness observes prepared contract compatibility:

- slot and resolver versions match;
- slot id, placement, header, and exact origins match;
- selected resolver exists for the current bundle generation;
- trusted identity binding is configured when the owner requires it.

Readiness must not invoke the resolver, acquire a token, inspect a credential
value, or make a provider request. Runtime expiry, revocation, or backing-store
failure remains a request-time credential failure unless the resolver
independently maintains a bounded observational snapshot.

## Traffic Policy Readiness

Traffic-policy readiness observes:

- one valid current policy snapshot;
- selected route profiles and dispatcher references resolve;
- the policy generation matches the owner binding;
- no configured route conflict prevents activation.

Readiness does not evaluate arbitrary request URLs. Per-request allow or deny
decisions remain request-time policy results.

## Hosted Dispatcher Readiness

Hosted-dispatch readiness observes:

- the selected typed dispatcher contribution resolves;
- an admitted session exists when the selected route requires hosted dispatch;
- binding, interface, carrier, owner, bundle, and policy generations match;
- the carrier handshake completed within its bound;
- the current incarnation accepts new operations.

An in-flight disconnect invalidates the carrier observation. Locally
synthesized operation certainty remains an owner result and is not converted
into readiness success.

Saturation and overload remain bounded operation failures. A sustained state
that means the binding cannot accept new work changes the owner snapshot; a
single overloaded operation does not.

## Plugin Readiness Providers

An activated external host plugin may use the RFC 0018 provider API only for a
fact the plugin owns and can observe safely.

Providers must:

- read an asynchronously maintained local snapshot;
- remain observational and idempotent;
- perform no blocking synchronous I/O;
- honor cancellation;
- avoid credential acquisition, owner activation, policy mutation, dispatch,
  model calls, Channel calls, and remote side effects;
- return bounded redacted reasons and messages.

A provider descriptor is not bundle availability evidence. A bundle manifest
is not provider activation evidence. Both must be validated through their own
lifecycles.

## Operator And Profile Selection

Operators may select host-integration criteria directly through RFC 0018
required or advisory criterion lists.

An RFC 0023 operator profile may add the same criterion ids while extending one
standard profile. It may strengthen inherited readiness but cannot weaken the
standard baseline.

Fleet, Scout, Lobster, or another control plane may author the normal
configuration that selects a criterion or profile. That control plane does not
evaluate the criterion and cannot report the cell ready on OpenClaw's behalf.

## Status And Doctor

Readiness answers whether the current activation can accept work under selected
requirements.

Status and Doctor may additionally expose:

- desired and resolved contribution identity;
- bundle, owner, policy, and carrier generations;
- provenance and last transition;
- degraded detail that does not block readiness;
- migration and rollback authority;
- structured repair guidance.

They consume the same owner snapshots but may not synthesize a different
readiness result. Descriptor enumeration must not invoke providers or
materialize credentials.

## Stable Reasons

Owners define bounded stable reasons. Initial cross-contract reasons should
include equivalents of:

- `HostIntegrationBundleUnavailable`
- `HostIntegrationContributionMissing`
- `HostIntegrationContributionIncompatible`
- `HostIntegrationActivationPending`
- `HostIntegrationGenerationStale`
- `HostIntegrationPolicyUnavailable`
- `HostIntegrationDispatcherUnavailable`
- `HostIntegrationDispatcherNotAdmitted`

Messages are redacted operator guidance, not machine identity. Raw exceptions,
credentials, request bodies, tenant content, and private resolver state are
never projected.

## Conformance Checklist

An implementation conforms when it proves:

- bundle `required` does not make a criterion required;
- manifest criteria do not register providers or report success;
- unknown owner/kind pairs and malformed selector ids fail validation;
- syntactically valid unknown selectors remain unresolved and follow the
  required-versus-optional aggregate behavior above;
- owner success cannot be supplied by the external host;
- `True`, `False`, and `Unknown` mapping follows current owner evidence;
- absent and stale evidence never becomes `True`;
- operator and profile selection use RFC 0018 aggregation;
- generation and incarnation changes invalidate cached results immediately;
- readiness polling performs no token acquisition, provider request, dispatch,
  activation, reload, or mutation;
- transient operation overload does not flap readiness;
- sustained inability to accept work changes owner evidence;
- bundle success cannot override owner failure;
- plugin providers remain bounded, observational, and redacted;
- `/ready`, `/readyz`, health, status, and CLI agree on one canonical result;
- multi-cell installations produce independent evidence and results.

## Conformance Fixtures

The v1 fixture set includes:

- valid bundle with an activated current owner binding;
- bundle published while owner activation is pending;
- missing and incompatible selected contribution;
- unknown declared readiness criterion;
- bundle, owner, policy, plugin, and carrier generation rollover;
- stale late evidence after every rollover;
- hosted dispatcher not admitted, admitted, replaced, and disconnected;
- credential resolver compatibility without credential acquisition;
- request-time credential failure without readiness probing;
- policy snapshot ready while one request is denied;
- transient overload versus sustained inability to accept work;
- external plugin attempt to report owner success;
- advisory and required selection of the same owner criterion;
- operator profile composition without weakening its standard parent;
- redacted authenticated and unauthenticated projections;
- two cells with the same plugin and independent readiness.
