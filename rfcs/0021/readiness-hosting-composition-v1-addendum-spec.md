# State CAPE Readiness and Hosting Composition v1 Addendum Specification

This optional addendum defines how RFC 0021 State CAPE evidence composes with
canonical Readiness, Standard Hosting Profiles, and Hosted Integration owner
evidence. It does not change the State CAPE levels or the Portable publication
provider contract.

Status: draft addendum, tied to RFC 0021 and dependent on RFC 0018. Hosting
Profile and Hosted Integration composition applies only when the corresponding
RFC is selected by the deployment.

## Scope

This addendum defines:

- canonical continuity readiness criteria and condition types;
- restored-startup readiness gating;
- steady-state recovery-point posture;
- optional Hosting Profile composition;
- optional Hosted Integration owner-evidence composition;
- generation invalidation, Status, and Doctor behavior; and
- conformance requirements for the composed result.

This addendum does not:

- define another readiness evaluator or condition schema;
- make a Hosting Profile mandatory;
- place Portable publication providers in a host integration bundle;
- let a host, profile, or plugin author continuity-owned truth;
- turn readiness into proof of immutable publication or `safeToDestroy`; or
- permit live backup, restore, credential, or network work during readiness
  evaluation.

## Dependencies

This addendum uses:

- [RFC 0018: Readiness Conditions and Providers](https://github.com/openclaw/rfcs/pull/33)
  and its Readiness v1 sidecar for condition identity, bounded evaluation,
  aggregation, caching, and projection;
- [RFC 0023: Standard Hosting Profiles](https://github.com/openclaw/rfcs/pull/37)
  and its Hosting Profile v1 sidecar when a profile is selected; and
- [RFC 0020: Hosted Owner Bindings and Managed Dispatch](https://github.com/giodl73-repo/rfcs/pull/2)
  and its Host Integration Readiness v1 sidecar when continuity dependencies
  use hosted owner bindings.

State CAPE remains valid without RFC 0020 or RFC 0023. RFC 0018 is required
only for the canonical readiness projection defined by this addendum.

## Core invariant

Continuity publishes immutable owner evidence. Readiness observes that evidence.
A profile classifies reusable criteria. Hosted Integration may supply evidence
for other semantic owners.

```text
continuity operation commits owner snapshot
  -> core-owned criterion adapter reads the snapshot
  -> RFC 0018 emits True, False, or Unknown
  -> operator or profile selects required/advisory posture
  -> every readiness projection consumes the same canonical result
```

Readiness evaluation must not capture state, publish or retrieve artifacts,
acquire credentials, mutate lifecycle authority, wake compute, reconcile cron,
or invoke a provider.

## Continuity owner snapshot

The criterion adapter reads one immutable bounded snapshot for the current
continuity configuration and runtime generation. The snapshot contains only
redacted evidence needed to derive conditions:

```ts
type ContinuityReadinessSnapshot = {
  specificationVersion: "state-cape/v1";
  selectedLevel: "conventional" | "archived" | "portable" | "elastic";
  effectiveLevel: "conventional" | "archived" | "portable" | "elastic";
  configGeneration: string;
  runtimeId: string;
  runtimeGeneration?: string;
  restore?: {
    state: "not-selected" | "pending" | "ready" | "failed" | "quarantined";
    sourceCheckpointId?: string;
    reason: string;
  };
  recoveryPoint?: {
    state: "current" | "stale" | "missing" | "unknown";
    checkpointId?: string;
    acceptedAgeMs?: number;
    targetRpoMs?: number;
    reason: string;
  };
  blockers: Array<{
    owner: string;
    reason: string;
  }>;
};
```

Exact implementation fields may differ, but the evidence must remain bounded,
redacted, attributable to one config/runtime generation, and sufficient to
derive every condition below without side effects.

Late or stale snapshots cannot update the active result. A missing snapshot or
generation mismatch is `Unknown`, never prior `True`.

## Criterion catalog

Continuity exposes these core-owned selectable criteria:

| Selector ID | Condition type | True when |
| --- | --- | --- |
| `openclaw.continuity-archived` | `ContinuityArchived` | Every State CAPE Archived requirement validates for the active configuration. |
| `openclaw.continuity-portable` | `ContinuityPortable` | Archived is true and every Portable authority, dependency-closure, publication, and restored-admission requirement validates. |
| `openclaw.continuity-elastic` | `ContinuityElastic` | Portable is true and every Elastic wake, deadline, retained-ingress, and hibernation requirement validates. |
| `openclaw.continuity-recovery-point-current` | `ContinuityRecoveryPointCurrent` | The latest accepted recovery point exists and its age is within the configured RPO target. |

The level criteria are monotonic. `ContinuityPortable=True` requires
`ContinuityArchived=True`; `ContinuityElastic=True` requires both lower
conditions to be true.

Elastic lifecycle authority and activation are defined by the
[Elastic Host Lifecycle v1 Specification](elastic-host-lifecycle-v1-spec.md).
Readiness projects its results but does not invoke `PrepareHibernate`,
`EnsureRuntimeReady`, revoke sleep, provision compute, or deliver retained
work.

Stable non-true reasons include:

| Condition | Stable reasons |
| --- | --- |
| `ContinuityArchived` | `ContinuityStateInventoryIncomplete`, `ContinuityCaptureUnavailable`, `ContinuityPublicationUnavailable`, `ContinuityRestoreUnsupported`, `ContinuityArchivedUnknown` |
| `ContinuityPortable` | `ContinuityGenerationAuthorityUnavailable`, `ContinuityProviderBindingInvalid`, `ContinuityDependencyClosureIncomplete`, `ContinuityRestoreAuthorityUnavailable`, `ContinuityPortableUnknown` |
| `ContinuityElastic` | `ContinuityWakeAuthorityUnavailable`, `ContinuityWakeSourceUnsupported`, `ContinuityDeadlineUnavailable`, `ContinuitySleepAuthorityUnavailable`, `ContinuityElasticUnknown` |
| `ContinuityRecoveryPointCurrent` | `ContinuityRecoveryPointMissing`, `ContinuityRecoveryPointStale`, `ContinuityRecoveryPointAgeUnknown` |

Messages are bounded redacted operator guidance. They must not include artifact
locations, provider credentials, tenant content, raw exception text, or
protected manifest metadata.

## Restored-startup gate

`ContinuityRestoreComplete` is a core continuity condition automatically
required when startup carries a restore intent. It is not operator-selectable
and cannot be made advisory by configuration or a Hosting Profile.

It is:

- `True` only after artifact integrity, compatibility, dependency closure,
  materialization, migration, identity reconstruction, scheduler
  reconciliation, and required owner activation complete for the current
  runtime generation;
- `False` after an authoritative terminal restore failure or quarantine; and
- `Unknown` while restore is pending or current evidence is unavailable.

Stable non-true reasons include:

```text
ContinuityRestorePending
ContinuityRestoreValidationFailed
ContinuityRestoreDependencyUnavailable
ContinuityRestoreQuarantined
ContinuityRestoreStatusUnknown
```

Gateway admission and every attachment or user-facing sidecar that reads or
mutates restored surfaces remain closed while this condition is non-true. A
successful HTTP process probe, plugin activation, or host assertion cannot
override it.

When startup has no restore intent, the condition is omitted rather than
reported as a successful restore.

## Recovery-point posture

`ContinuityRecoveryPointCurrent` reports steady-state protection posture. It
does not claim that the recovery point includes mutations after its accepted
capture, and it is not a `/synced` signal.

Its age uses the lifecycle authority's durable observed-acceptance time, not a
provider-authored timestamp. The criterion becomes `Unknown` when age,
acceptance identity, or current policy cannot be established.

Operators may keep this criterion advisory or make it required. Making it
required can deliberately remove a running instance from service when its RPO
is exceeded; the profile or operator owns that availability tradeoff.

## Hosting Profile composition

State CAPE does not add a standard RFC 0023 profile in v1. A deployment may use
an operator profile to extend one standard runtime posture:

```json5
{
  hosting: {
    profile: "acme/managed-elastic",
    profiles: {
      "acme/managed-elastic": {
        extends: "container",
        requiredCriteria: [
          "openclaw.continuity-archived",
          "openclaw.continuity-portable",
          "openclaw.continuity-elastic",
          "openclaw.continuity-recovery-point-current",
          "plugin.msteams.wake-capable",
        ],
      },
    },
  },
}
```

The profile:

- selects and classifies criteria but does not produce their status;
- may strengthen but never weaken the standard profile or restored-startup
  gate;
- cannot turn missing, stale, or unknown evidence into `True`;
- cannot substitute profile, artifact, or incarnation identity for continuity
  runtime generation; and
- must fail validation when a required selector is unknown.

A future standard managed profile requires its own accepted profile revision
and packaged conformance. This addendum does not reserve or silently activate
one.

## Hosted Integration composition

Portable publication remains an ordinary external continuity provider selected
through `continuity.publicationProvider`. It is not a Host Integration Bundle
contribution, and host-bundle generation is not publication or restore
authority.

Hosted Integration may optionally supply other dependency-closure capabilities,
including credential resolvers, Channel endpoints, retained-ingress routes, or
lifecycle adapters, when their semantic owners define versioned contracts.

For each hosted dependency:

1. the semantic owner prepares and activates its binding;
2. the owner publishes one immutable activation snapshot;
3. the RFC 0020 criterion adapter maps that snapshot to RFC 0018;
4. the operator or Hosting Profile classifies the criterion; and
5. continuity consumes only the owner result required by dependency closure.

The bundle, dispatcher, external host, and profile cannot author continuity,
Channel, credential, identity, or lifecycle success on the semantic owner's
behalf.

Host Integration generation metadata is diagnostic. Stale bundle, owner,
policy, or binding generation makes the affected owner criterion `Unknown`; it
does not mutate continuity runtime generation or provider compatibility
generation.

## Evaluation and invalidation

Criterion evaluation:

- reads only the current immutable owner snapshot;
- follows RFC 0018 deadlines, cancellation, coalescing, and cache bounds;
- performs no request-time provider, storage, credential, Channel, or host
  network call;
- returns `Unknown` on timeout, missing evidence, malformed evidence, or
  generation mismatch; and
- never reuses a prior `True` after invalidation.

Affected cache entries invalidate immediately when:

- continuity config or selected CAPE level changes;
- runtime generation changes or authority is lost;
- publication provider ownership, version, or compatibility generation
  changes;
- restore state, quarantine, or dependency closure changes;
- accepted recovery-point identity or RPO policy changes;
- an enabled wake source or semantic deadline changes; or
- an owner snapshot required by the selected profile changes generation.

## Status and Doctor

Readiness answers whether the selected runtime posture can currently serve.
Status and Doctor retain richer diagnostics.

Status includes:

- selected and effective CAPE level;
- condition status, requirement, reason, and evidence generation;
- current restore phase and source checkpoint identity;
- latest accepted recovery-point age and RPO target;
- provider plugin, ID, contract version, and compatibility generation;
- unresolved dependency-owner criteria;
- earliest semantic wake deadline;
- accepted wake-registration, sleep-authorization, and active wake-request
  identities;
- selected Hosting Profile and selection source; and
- relevant host owner/bundle generations as diagnostic metadata.

Doctor reports:

- unknown selectors or profile composition;
- missing or stale continuity snapshots;
- provider ownership, version, or generation mismatch;
- restore gate and dependency-closure blockers;
- stale or missing recovery points;
- unsupported enabled wake sources;
- stale Hosted Integration owner evidence; and
- a selected CAPE level not satisfied by the effective result.

Neither surface exposes credentials, protected artifact metadata, tenant
content, or raw provider errors.

## Conformance

Conformance proves:

- condition IDs and stable reasons exactly match this addendum;
- higher-level `True` cannot coexist with a non-true inherited level;
- restore intent automatically requires `ContinuityRestoreComplete`;
- admission and attached sidecars remain closed while restore is non-true;
- missing, timed-out, malformed, or stale evidence becomes `Unknown`;
- provider or runtime generation changes invalidate prior `True`;
- recovery-point age uses independently recorded lifecycle acceptance time;
- operator profiles select criteria without changing owner truth;
- unknown required profile selectors fail validation;
- Hosted Integration owner snapshots cannot author continuity success;
- stale hosted owner generations become `Unknown`; and
- every HTTP, health, Status, and CLI projection consumes one canonical result.

Conformance must include negative fixtures for stale generation, provider
version mismatch, missing dependency closure, expired RPO, unsupported wake
source, restore quarantine, and a sidecar attempting to report ready before
restore completion.
