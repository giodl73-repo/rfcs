# Provider Request Traffic Policy v1 Specification

This document is the implementer-facing provider-request traffic-policy
specification for RFC 0020. It defines how owner-prepared request authority is
intersected with host routing and destination constraints before guarded local
or hosted dispatch.

Status: draft addendum, tied to RFC 0020.

## Scope

Traffic policy v1 defines:

- policy registration and immutable snapshots;
- request fact matching;
- deny and allow rules;
- route profiles and dispatcher selection;
- exact-origin, private-network, timeout, proxy, and pin intersection;
- deterministic conflict behavior;
- generation and diagnostic requirements;
- conformance fixtures.

It does not define:

- provider request semantics;
- tenant, mailbox, user, model, or Channel selection;
- credential acquisition;
- carrier framing;
- semantic retries or response interpretation;
- a general-purpose policy language.

## Version

```text
provider-request-traffic-policy/v1
```

Unknown major versions fail closed.

## Core Invariant

Traffic policy can only narrow the authority already prepared by the semantic
owner.

```text
owner destination, timeout, private-network, and dispatcher constraints
  intersect every matching allow rule
  intersect selected route profile
  -> allow with narrower effective authority
     or deny
```

Policy must never rewrite a logical destination, choose semantic identity, add
credentials, or silently fall back to a weaker route.

## Registration

A policy registration has:

```json
{
  "version": "provider-request-traffic-policy/v1",
  "id": "example/managed-egress",
  "required": true,
  "provenance": {
    "source": "plugins.entries.example-host.config",
    "revision": "sha256:..."
  },
  "routeProfiles": [],
  "rules": []
}
```

| Field | Required | Semantics |
| --- | --- | --- |
| `version` | Yes | Exact policy contract version. |
| `id` | Yes | Stable namespaced policy id. |
| `generation` | Yes in the published snapshot | Opaque registry-issued current policy generation. |
| `required` | Yes | Whether no matching rule denies instead of returning no policy decision. |
| `provenance.source` | Yes | Bounded configuration or plugin source. |
| `provenance.revision` | Yes | Bounded immutable revision or digest. |
| `routeProfiles` | Yes | Non-empty route profile set. |
| `rules` | Yes | Non-empty deterministic rule set. |

The v1 policy owner scope is one Gateway cell's provider-request policy
registry. Only one authoritative policy snapshot may be active in that scope.
A competing owner or different policy id fails closed. A bundle may inventory
multiple policy implementations, but owner configuration selects at most one
to publish. Disposal may clear only the snapshot it published.

The publisher does not choose `generation`. The registry issues a fresh opaque
generation for every accepted publication, including a semantically identical
reload, and rejects or ignores any caller-supplied generation value. Publication
input therefore omits this field; the published immutable snapshot and every
derived decision include it. Reusing a prior generation is not valid.

## Request Facts

The owner supplies immutable facts:

```json
{
  "provider": "m365mail-graph",
  "capability": "channel",
  "transport": "request-response",
  "endpointClass": "microsoft-graph-mail",
  "url": "https://graph.microsoft.com/v1.0/users/agent/sendMail",
  "allowPrivateNetwork": false,
  "timeoutMs": 30000
}
```

Facts may include a prepared dispatcher policy from local configuration.

Initial proven capability and transport values are:

- capabilities: `llm`, `channel`;
- transports: `stream`, `request-response`.

New values require an owner path with equivalent guarded-fetch, credential, and
response enforcement.

Web-search owner contributions do not use this policy contract in v1. A future
web-search capability value requires equivalent guarded one-hop enforcement
before admission.

## Match Object

A rule may match:

| Field | Semantics |
| --- | --- |
| `providers` | Exact case-normalized provider ids. |
| `capabilities` | Exact owner capability classes. |
| `transports` | Exact transport classes. |
| `endpointClasses` | Exact owner-defined endpoint classes. |
| `origins` | Exact normalized HTTP(S) origins. |

An omitted field matches all values for that dimension. Empty sets are invalid.
Duplicate entries are invalid.

## Rule Outcomes

### Deny

```json
{
  "action": "deny",
  "reason": "DestinationBlocked"
}
```

The reason is a bounded machine-readable code. It must not contain secrets or
raw request data.

### Allow

```json
{
  "action": "allow",
  "routeProfileId": "example/managed",
  "allowedOrigins": [
    "https://graph.microsoft.com"
  ],
  "allowPrivateNetwork": false,
  "maximumTimeoutMs": 30000
}
```

An allow outcome selects one route profile and further constrains exact
origins, private-network posture, and timeout.

## Route Profiles

```json
{
  "id": "example/managed",
  "dispatcherPolicy": {
    "mode": "explicit-proxy",
    "proxyUrl": "https://proxy.example.test"
  },
  "dispatchBindingId": "example/reverse-provider"
}
```

Route profile ids are unique.

Dispatcher policy modes are:

- `direct`;
- `environment-proxy`;
- `explicit-proxy`; and
- `managed-proxy`.

Profiles may carry bounded connect, proxy TLS, and pinned-hostname constraints.
They must not disable TLS verification. Explicit proxy URLs must use HTTP or
HTTPS. `managed-proxy` requires a `dispatchBindingId` and means that admitted
connection owner selects a physical proxy within the profile's declared
origin, address, TLS, timeout, and private-network constraints. A
`dispatchBindingId` selects one typed hosted dispatcher contribution; absence
means the route remains local and cannot use `managed-proxy`.

## Validation

Before publication:

1. Validate version, ids, provenance, and required posture, then assign a fresh
   registry generation.
2. Normalize exact origins.
3. Reject duplicate route and rule ids.
4. Reject missing route profiles.
5. Reject allow rules that reference unknown profiles.
6. Reject empty allowed-origin sets.
7. Reject invalid or non-positive timeout bounds.
8. Reject proxy policies that disable TLS verification or use unsupported
   schemes.
9. Freeze the complete snapshot.

## Evaluation

Evaluation is deterministic:

1. Compute the exact origin from the owner-prepared logical URL.
2. Select all matching rules.
3. If no rule matches:
   - return no decision when the policy is advisory;
   - deny with `required-policy-no-match` when it is required.
4. If any matching rule denies, deny.
5. Require the origin to appear in every matching allow rule.
6. Require all matching allow rules to select one route profile.
7. Require that route profile to exist.
8. Intersect owner-prepared dispatcher policy with the selected profile. When
   the owner omitted that optional policy, treat it as neutral rather than as
   an implicit direct-route requirement; the owner's URL, network guard, and
   other prepared restrictions still apply.
9. Allow private networking only when the owner permits it and every matching
   allow rule permits it.
10. Select the smallest defined timeout across owner and matching rules.
11. Return one immutable decision tied to the policy generation.

Rule order does not grant precedence. Deny wins, and multiple allow rules
intersect.

## Dispatcher Policy Intersection

Intersection must preserve or narrow prepared security:

- selected pinned addresses intersect with prepared pinned addresses;
- incompatible pinned hostnames deny;
- conflicting TLS or connect fields deny;
- an explicit proxy must match an already prepared explicit proxy;
- an environment proxy must remain an environment proxy when one was prepared;
- a managed proxy must remain bound to the selected hosted dispatcher and the
  profile's connection constraints;
- private proxy permission is true only when both sides allow it;
- host policy cannot replace prepared direct-route pinning with a proxy;
- host policy cannot replace a prepared environment or explicit proxy with
  direct execution;
- route conflict denies with a stable category.

The first two proxy-matching rules apply only when the owner prepared a
dispatcher policy. Omission is the neutral element: the selected route profile
may choose direct, environment, explicit, or managed proxy execution, but may
not weaken any other owner-prepared network restriction.

The result must describe the actual connection owner. When a proxy or hosted
dispatcher resolves and connects, the network guard must record proxy or
connection-owner enforcement rather than claiming caller-resolved direct
execution.

## Decision

An allow decision contains:

```json
{
  "action": "allow",
  "policyId": "example/managed-egress",
  "policyGeneration": "policy-generation-42",
  "routeProfileId": "example/managed",
  "dispatchBindingId": "example/reverse-provider",
  "allowPrivateNetwork": false,
  "timeoutMs": 30000,
  "dispatcherPolicy": {
    "mode": "explicit-proxy",
    "proxyUrl": "https://proxy.example.test"
  }
}
```

A deny decision contains policy id, generation, and one bounded reason.

Implementations should use stable reasons equivalent to:

- `required-policy-no-match`
- `destination-outside-policy`
- `conflicting-route-profiles`
- `route-profile-unavailable`
- `configured-route-conflict`
- owner-supplied deny reason

## Redirects

Every redirect returns to the owner and guard loop.

The redirected URL is evaluated as a new set of facts. Continuing the operation
requires:

- another allow decision;
- the same policy generation;
- the same route profile and dispatch binding;
- no wider private-network or timeout authority;
- an equivalent or narrower dispatcher policy;
- owner approval of redirect semantics.

Hosted dispatch never follows redirects internally.

## Generations And Reload

Policy generation is part of every decision and hosted operation fence.

- New policy publication supersedes prior decisions before new side effects.
- Every reverse-dispatch frame carries the policy generation selected at
  operation open.
- In-flight work follows owner certainty and cancellation rules.
- Stale results do not become authoritative merely because the logical URL is
  unchanged.
- Missing or invalid required policy never falls back to direct execution.

## Diagnostics

Status and Doctor may expose:

- policy id and generation;
- readiness;
- bounded provenance;
- matched route profile;
- dispatch binding id;
- stable deny or conflict reason.

They must not expose credentials, raw authorization headers, unbounded URLs,
proxy credentials, private TLS material, or owner request bodies.

## Example Published Policy Snapshot

```json
{
  "version": "provider-request-traffic-policy/v1",
  "id": "example/managed-egress",
  "generation": "policy-generation-42",
  "required": true,
  "provenance": {
    "source": "plugins.entries.example-host.config",
    "revision": "sha256:0123456789abcdef"
  },
  "routeProfiles": [
    {
      "id": "example/managed",
      "dispatcherPolicy": {
        "mode": "explicit-proxy",
        "proxyUrl": "https://proxy.example.test"
      },
      "dispatchBindingId": "example/reverse-provider"
    }
  ],
  "rules": [
    {
      "id": "m365mail-graph",
      "match": {
        "providers": ["m365mail-graph"],
        "capabilities": ["channel"],
        "transports": ["request-response"],
        "endpointClasses": ["microsoft-graph-mail"]
      },
      "outcome": {
        "action": "allow",
        "routeProfileId": "example/managed",
        "allowedOrigins": [
          "https://graph.microsoft.com"
        ],
        "allowPrivateNetwork": false,
        "maximumTimeoutMs": 30000
      }
    }
  ]
}
```

## Compatibility And Evolution

- New required fields require a new policy contract version.
- New capability or transport values require equivalent enforcement proof.
- Unknown action or dispatcher policy modes fail closed.
- New optional provenance or diagnostic metadata may be ignored.
- Policy syntax must not evolve into semantic request transformation or tenant
  routing.

## Policy Publisher Checklist

A policy publisher is compatible when it:

- emits one immutable versioned snapshot;
- uses exact origins and stable ids;
- supplies non-secret provenance;
- keeps rules deterministic and bounded;
- uses deny precedence and allow intersection;
- names route profiles explicitly;
- never writes credentials or semantic request mutations.

## OpenClaw Owner Checklist

An owner is compatible when it:

- supplies trusted immutable facts;
- applies policy after semantic preparation and before credential materialization
  or physical dispatch;
- rejects route and generation drift;
- re-evaluates redirects;
- preserves the owner's stricter destination, timeout, private-network, proxy,
  and pinning constraints;
- never silently falls back when required policy is unavailable.

## Conformance Fixtures

The v1 suite should include:

- advisory and required no-match behavior;
- deny precedence;
- exact-origin allow and denial;
- conflicting route profiles;
- missing route profile;
- private-network intersection;
- a rule that requests broader private-network or timeout authority is narrowed
  to the owner's grant;
- timeout minimum selection;
- direct, environment-proxy, and explicit-proxy intersection;
- proxy URL and TLS conflicts;
- pinned hostname and address intersection;
- hosted and local route selection;
- redirect route drift;
- policy generation supersession;
- stale policy generation on hosted dispatch;
- redacted diagnostics.
