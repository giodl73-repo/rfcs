# Credential Slot v1 Specification

This document is the implementer-facing credential-slot specification for RFC
0020. It builds on the host integration bundle core specification and defines
the narrow credential contract shared by local guarded dispatch and hosted
physical dispatch.

Status: draft addendum, tied to RFC 0020.

## Scope

Credential slot v1 defines:

- exact header-placement declarations;
- exact-origin allowlists;
- versioned resolver compatibility;
- preparation and readiness metadata;
- request-time acquisition and expiry checks;
- header conflict and duplicate-reference behavior;
- redirect and local/hosted parity requirements;
- redaction and conformance rules.

Credential slot v1 does not define:

- arbitrary header mutation;
- query-string or body credentials;
- computed request signing;
- multiple-header credential recipes;
- generic SecretRef expansion in a dispatcher;
- identity selection by a payload;
- credential persistence or reuse.

## Versions

```text
credential-slot/v1
credential-slot-resolver/v1
```

Definitions and resolvers must use exact supported versions. Version mismatch
fails before any credential value is acquired.

## Core Invariant

The semantic owner fixes the credential placement and destination authority.
The resolver supplies only a value for that exact slot.

```text
owner declaration
  intersect selected resolver metadata
  intersect current logical origin
  intersect trusted identity and generation
  -> resolve one ephemeral value
  -> place it in one declared header
```

No resolver may choose a different header, origin, owner, tenant, mailbox, or
request.

## Slot Definition

```json
{
  "version": "credential-slot/v1",
  "slotId": "m365mail/graph-token",
  "placement": "header",
  "headerName": "authorization",
  "allowedOrigins": [
    "https://graph.microsoft.com"
  ],
  "required": true,
  "resolverId": "example/graph-token-resolver"
}
```

| Field | Type | Required | Semantics |
| --- | --- | --- | --- |
| `version` | string | Yes | Must be `credential-slot/v1`. |
| `slotId` | string | Yes | Stable namespaced owner-defined slot id. |
| `placement` | string | Yes | Must be `header` in v1. |
| `headerName` | string | Yes | One valid HTTP header name, normalized to lowercase. |
| `allowedOrigins` | string array | Yes | Non-empty set of exact HTTPS origins. |
| `required` | boolean | Yes | Whether absence makes preparation unavailable. |
| `resolverId` | string | Yes | Exact selected resolver implementation id. |

Origins contain the `https` scheme, host, and optional port only. Cleartext
HTTP, paths, queries, fragments, userinfo, wildcards, and suffix matching are
not allowed. V1 has no insecure-transport exception for header credentials.

## Resolver Declaration

A resolver supplies metadata that must exactly match the slot:

```json
{
  "version": "credential-slot-resolver/v1",
  "resolverId": "example/graph-token-resolver",
  "slotId": "m365mail/graph-token",
  "placement": "header",
  "headerName": "authorization",
  "allowedOrigins": [
    "https://graph.microsoft.com"
  ]
}
```

At runtime the resolver implements:

```text
resolve(slotId, origin, signal?) -> Promise<{ value, expiresAtMs? } | null>
```

The resolver receives the normalized exact origin, not an arbitrary URL or
request body.

When credential subject or tenant is authoritative, the resolver selection or
credential validation must bind that subject to the owner's trusted identity.
Origin and audience checks alone are not a mailbox, tenant, or user binding.

The resolver must apply backing-store revocation before returning a value. A
revoked credential is returned as unavailable, not as a value with a separate
caller-interpreted revocation flag.

## Preparation

Preparation validates all definitions and resolver metadata before requests can
use them:

1. Reject unsupported versions and placements.
2. Normalize ids, header names, and exact origins.
3. Reject empty origin sets.
4. Reject duplicate slot ids and duplicate resolver ids.
5. Reject two slots that claim the same output header for any shared origin.
6. Require the resolver for each `required: true` definition to exist. Preserve
   a `required: false` definition with unavailable metadata when its resolver
   is absent.
7. For every present resolver, require its slot id, placement, header name, and
   origin set to match the definition exactly.
8. Freeze readiness metadata, including resolver availability.

Preparation must not invoke a resolver and must not materialize credential
values.

## Prepared Binding Metadata

Preparation exposes non-secret metadata that an owner activation snapshot may
reference:

```json
{
  "slotId": "m365mail/graph-token",
  "resolverId": "example/graph-token-resolver",
  "version": "credential-slot/v1",
  "resolverVersion": "credential-slot-resolver/v1",
  "placement": "header",
  "headerName": "authorization",
  "allowedOrigins": [
    "https://graph.microsoft.com"
  ],
  "required": true,
  "availability": "ready"
}
```

`availability` is `ready` or `unavailable`. An unavailable optional slot does
not fail owner preparation, but a request that explicitly references it fails
with `credential-unavailable` before any physical dispatch.

Readiness must never invoke the resolver or expose values, tokens, expiry
claims, credential subjects, private key material, or resolver internals.
Canonical condition mapping is defined by the host integration readiness
sidecar.

## Request-Time Application

Application receives:

```text
slotRefs
logical URL
request init
abort signal
current time
```

The implementation must perform this order:

1. Normalize and validate the logical URL.
2. Validate every slot reference before invoking any resolver.
3. Reject duplicate references.
4. Reject unknown slots.
5. Reject references whose exact origin is not allowed.
6. Reject an existing request header that conflicts with a selected slot.
7. Resolve selected slots with the normalized origin and abort signal.
8. Reject `null`, empty, malformed, or expired values.
9. Place each value in its one declared header.
10. Return a new request init without mutating the prepared owner request.

Invalid references must fail before the first credential acquisition.

Credential values must be validated without reflecting the value in thrown
errors. Header parser or runtime exceptions must be translated into bounded
slot failures that do not include the credential.

## Redirect Behavior

Credentials are not automatically carried across redirect origins.

Every redirect returns to the owner and guarded-fetch loop. Before another hop:

- the owner validates redirect semantics;
- traffic policy is re-evaluated;
- the new exact origin is checked;
- credential slots are re-applied only if the owner still declares them and the
  new origin is allowed.

Headers produced by credential slots on the prior hop must be removed before
slot reapplication on every redirect, including same-origin redirects, so
expiry and revocation are checked again. Cross-origin redirects additionally
remove all other sensitive headers before any new credential value is
acquired.

## Local And Hosted Parity

Local and hosted execution consume the same prepared slot references and
readiness metadata.

- Local guarded dispatch may invoke the selected resolver and place the header.
- Hosted dispatch may send slot references to an admitted host-owned resolver.

The hosted path must enforce the same slot id, resolver contract version,
header, origin, identity, expiry, revocation, and generation checks. It may not
accept a generic map of secret names or arbitrary header mutations.

## Failure Categories

Implementations should expose stable categories equivalent to:

- `invalid-definition`
- `duplicate-slot`
- `duplicate-resolver`
- `ambiguous-header`
- `missing-resolver`
- `incompatible-resolver`
- `unknown-slot`
- `duplicate-reference`
- `origin-denied`
- `header-conflict`
- `credential-unavailable`
- `credential-expired`
- `stale-generation`

Failure output may include a slot id and category. It must not include the
credential value, raw authorization header, private resolver state, or
credential-bearing URL.

## Security Requirements

- Credential values are ephemeral request data.
- Resolver execution is asynchronous and must not block the caller's event
  loop, even when a value is cached.
- Resolvers must not log values.
- Status, Doctor, readiness, traces, captures, and audit records contain only
  bounded non-secret metadata.
- A hosted dispatcher may observe the final header because it performs the
  physical exchange, but that does not grant authority to mint, retain, reuse,
  or broaden the credential.
- Cancellation does not guarantee a resolver or remote host erased work that
  already started.
- Generation changes invalidate future acquisition and dispatch authority.

## Compatibility And Evolution

- New placements require a new credential-slot contract version.
- New resolver capabilities require an exact resolver contract version.
- Optional non-secret readiness metadata may be added compatibly.
- A resolver must not silently accept a broader origin set or different header
  under the same version.
- Computed signing, multi-header recipes, and body credentials remain separate
  owner contracts until proven.

## Resolver Publisher Checklist

A resolver is compatible when it:

- implements `credential-slot-resolver/v1`;
- matches one exact slot definition;
- accepts only normalized exact origins;
- binds authoritative identity when required;
- returns a value and optional expiry only;
- honors cancellation and revocation as far as the backing system permits;
- emits no credential material in diagnostics.

## OpenClaw Owner Checklist

An owner is compatible when it:

- declares stable slot ids;
- fixes placement, header, and allowed origins;
- selects a resolver by typed id and exact version;
- validates all references before acquisition;
- applies credentials after semantic request preparation;
- re-evaluates credentials on redirects;
- owns response and replay decisions.

## Conformance Fixtures

The v1 suite should include:

- one valid exact-origin header slot;
- local and hosted application parity;
- missing and incompatible resolver;
- duplicate slot, resolver, reference, and header ownership;
- unknown slot;
- public and denied origins;
- explicit-port origin behavior;
- redirect to same and different origins;
- pre-existing header conflict;
- unavailable, empty, malformed, expired, and revoked values;
- abort before and during resolution;
- credential value redaction from every error and diagnostic surface;
- resolver subject or tenant mismatch against owner trusted identity;
- stale owner and bundle generation rejection.
