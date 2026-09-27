# Hosted Provider Dispatch v1 Specification

This document is the implementer-facing hosted-dispatch specification for RFC
0020. It defines a semantically empty one-hop provider-request dispatcher and
the optional bounded reverse carrier used when physical execution must occur in
an admitted host runtime.

Status: draft addendum, tied to RFC 0020.

## Scope

Hosted provider dispatch v1 defines:

- cell-local dispatcher admission;
- one immutable owner-prepared operation;
- one redirect-disabled physical exchange;
- request and response streaming;
- credit-based flow control;
- owner, bundle, policy-generation, and incarnation fencing;
- bounded frames, queues, bytes, traces, and deadlines;
- cancellation and dispatch certainty;
- terminal outcomes and failure categories;
- diagnostics and conformance.

It does not define:

- provider, Channel, mailbox, tenant, user, or model semantics;
- credential minting;
- traffic-policy authoring;
- semantic retries;
- a generic host method API;
- a carrier catalog;
- cross-cell or cross-tenant routing.

## Versions

The reverse carrier frame contract is:

```text
reverse-provider-dispatch/v1
```

The typed dispatcher contribution uses an exact owner contract version such as:

```text
provider-request-dispatcher/v1
```

Unsupported versions fail before admission or operation delivery.

## Core Invariant

The semantic owner authorizes and prepares the request. The dispatcher performs
one physical exchange without reinterpreting it.

```text
immutable owner request
  -> credential-slot references
  -> effective traffic-policy decision
  -> network guard profile
  -> one admitted dispatcher binding
  -> one redirect-disabled exchange
  -> bounded response and certainty
  -> owner response and replay decision
```

The dispatcher may narrow or fail. It may not widen or transform semantic
authority.

## Roles

- **Owner**: prepares method, logical URL, non-secret headers, body, credential
  slot references, response policy, replay policy, limits, and generation.
- **Gateway cell**: admits one dispatcher binding, validates frames, enforces
  generations and bounds, and routes operation frames.
- **Host dispatcher**: resolves physical routing, materializes declared slots,
  performs the one-hop exchange, and returns bounded response frames.

These roles may run in one process or separate processes. Their authority
boundaries remain the same.

## Admission

A remote dispatcher session is admitted for exactly one Gateway cell and one
typed binding.

Admission proves:

- trusted issuer and deployment audience;
- cell audience and binding runtime audience;
- binding id and contract version;
- owner and host bundle generations;
- allowed role and operations;
- issued-at, expiry, and credential id;
- proof of possession;
- current bundle and owner authority.

The Gateway rejects missing, expired, replayed, wrong-audience, wrong-binding,
wrong-generation, or revoked credentials.

One binding has at most one active carrier session. A new admitted incarnation
may replace the prior session, but it does not replay prior operations.

A carrier becomes active only after its successful protocol handshake is
written and acknowledged.

Admission handshake completion is bounded by a fixed platform deadline. On
expiry the Gateway releases the reservation and fails the attempt.

## Operation Open

An operation begins with:

```json
{
  "version": "reverse-provider-dispatch/v1",
  "type": "operation-open",
  "incarnationId": "carrier-incarnation",
  "operationId": "operation-id",
  "ownerGeneration": "owner-generation",
  "hostBundleGeneration": "bundle-generation",
  "policyGeneration": "policy-generation-42",
  "bindingId": "example/reverse-provider",
  "timeoutMs": 30000,
  "requestByteLimit": 10485760,
  "responseByteLimit": 50331648,
  "maxFrameBytes": 2097152,
  "maxChunkBytes": 1048576,
  "request": {
    "method": "POST",
    "url": "https://graph.microsoft.com/v1.0/users/agent/sendMail",
    "headers": {
      "content-type": "application/json"
    },
    "credentialSlotRefs": [
      "m365mail/graph-token"
    ],
    "routeProfile": "example/managed",
    "networkGuard": {
      "version": "network-guard/v1",
      "target": {
        "protocol": "https:",
        "origin": "https://graph.microsoft.com",
        "hostname": "graph.microsoft.com",
        "port": 443
      },
      "route": {
        "mode": "explicit-proxy",
        "resolution": "proxy",
        "tls": "required"
      },
      "addressPolicy": {
        "mode": "public-only",
        "trustedHostnames": [],
        "hostnameAllowlist": [],
        "allowedPrivateCidrs": [],
        "allowRfc2544BenchmarkRange": false,
        "allowIpv6UniqueLocalRange": false,
        "dnsRebinding": {
          "policy": "reject",
          "enforcement": "connection-owner-required"
        }
      }
    },
    "auditCorrelation": "bounded-correlation-id"
  }
}
```

The exact network guard object is defined by the guarded-fetch contract. It
describes the logical target and the connection owner's enforcement duties.

Operation fields are immutable after open.

`timeoutMs` is a positive relative operation budget in milliseconds, not a
wall-clock timestamp. The Gateway starts its authoritative timer when it
commits `operation-open` to the carrier. The host starts an equal-or-shorter
timer when it receives the frame; transport time therefore consumes the
Gateway's budget and never extends it. A terminal received after the Gateway
budget expires is rejected as late evidence.

`bindingId` must equal the traffic-policy decision's `dispatchBindingId`.
`policyGeneration` must equal the generation on that same decision.
`request.routeProfile` must equal the decision's `routeProfileId`, and
`timeoutMs` must be positive and no greater than the decision's `timeoutMs`.
The supplied network guard must equal or narrow the decision's effective
connection authority. Any mismatch fails before operation delivery.

## Common Frame Fields

Every frame contains:

| Field | Semantics |
| --- | --- |
| `version` | Exact reverse dispatch contract version. |
| `incarnationId` | Current carrier incarnation. |
| `operationId` | Unique operation within the current authority scope. |
| `ownerGeneration` | Authoritative owner configuration generation. |
| `hostBundleGeneration` | Admitted bundle generation. |
| `policyGeneration` | Effective traffic-policy generation. |
| `type` | Exact frame type. |

A frame with a stale incarnation, owner generation, bundle generation, policy
generation, unknown operation, unsupported version, or unknown field fails
before delivery to the operation consumer and is recorded in bounded audit
evidence.

## Frame Types

| Type | Direction | Semantics |
| --- | --- | --- |
| `operation-open` | Gateway to host | Opens one immutable operation and its limits. |
| `credit` | Either | Grants bounded bytes for request or response chunks. |
| `chunk` | Either | Carries canonical base64 payload bytes with a monotonic sequence. |
| `half-close` | Either | Declares no more chunks for one stream direction. |
| `dispatch-started` | Host to Gateway | Physical execution may have started. |
| `response-open` | Host to Gateway | Returns status, status text, and response headers. |
| `cancel` | Either | Requests cancellation with a bounded reason. |
| `terminal` | Host to Gateway | Settles the operation once with outcome and certainty. |

Unknown frame types fail closed.

Stream direction is fixed:

- for `stream: "request"`, the host sends `credit`; the Gateway sends `chunk`
  and `half-close`;
- for `stream: "response"`, the Gateway sends `credit`; the host sends `chunk`
  and `half-close`.

The opposite endpoint or an invalid stream value is a protocol violation.

In addition to the common fields, v1 permits only these type-specific fields:

| Type | Type-specific fields |
| --- | --- |
| `operation-open` | `bindingId`, `timeoutMs`, `requestByteLimit`, `responseByteLimit`, `maxFrameBytes`, `maxChunkBytes`, and `request` exactly as defined above. |
| `credit` | `stream` (`request` or `response`) and positive integer `bytes`. |
| `chunk` | `stream`, non-negative integer `sequence`, and canonical base64 string `data`. |
| `half-close` | `stream`. |
| `dispatch-started` | None. |
| `response-open` | final integer `status` from 200 through 599, bounded `statusText`, and bounded response `headers`. V1 does not carry informational responses or protocol upgrades. |
| `cancel` | bounded `reason`. |
| `terminal` | `outcome` (`completed`, `failed`, or `cancelled`), `certainty`, optional stable `failureCode`, and optional bounded `message`. |

Required fields may not be omitted. Fields listed only for another frame type
are unknown fields and fail closed. All integers are JSON integers within the
advertised platform bounds; header names and values use the same bounded
validated representation as `operation-open`.

## Flow Control

Request and response streams have independent byte credit.

- A sender may transmit chunk payload bytes only after receiving sufficient
  credit for that stream.
- Credit is consumed by decoded payload bytes.
- Credit grants are positive bounded integers.
- Chunk sequence starts at zero and increases by one.
- Chunks use canonical base64.
- One `half-close` ends each stream direction.
- A half-closed stream cannot receive later chunks.
- Queue, outstanding credit, frame, chunk, and aggregate byte limits are
  enforced independently.

Slow or absent consumers must not create unbounded memory use or block Gateway
control traffic or another binding's operations.

Implementations clamp frame and chunk limits to fixed platform ceilings
regardless of owner-declared values. A single JSON parse, base64 decode, or
chunk delivery must not create unbounded event-loop work.

## Response Semantics

The host returns `response-open` before response chunks.

The dispatcher:

- preserves status and bounded response headers;
- does not interpret provider success or failure;
- returns redirects to the owner;
- does not follow redirects;
- respects response byte and deadline limits;
- half-closes the response stream before terminal completion.

Bodyless responses are valid and still require coherent response-open,
half-close, and terminal sequencing.

## Credential Materialization

The operation carries credential slot references, not values.

The admitted host may materialize a value only when:

- the owner declared the slot;
- the selected resolver is compatible;
- the logical origin is allowed;
- the effective traffic policy selected this route;
- trusted identity, expiry, revocation, and generations remain valid.

Values are placed only in declared headers and are never returned on the
carrier, in traces, or in diagnostics.

## Network Safety

OpenClaw supplies the effective network guard profile after owner and
traffic-policy intersection.

The connection owner must enforce:

- required scheme and exact logical origin;
- public, private, loopback, link-local, and reserved address posture;
- approved private origins or CIDRs;
- DNS rebinding and address selection;
- proxy or route profile restrictions;
- timeout and connection bounds.

When the host or proxy performs DNS and connection, the profile records
connection-owner enforcement. It must not claim caller-resolved or pinned
direct execution.

The dispatcher may select physical DNS or an approved proxy inside the named
route profile. It may not change the logical URL.

## Network Guard Profile

`operation-open.request.networkGuard` uses the exact
`network-guard/v1` shape:

| Field | Values and semantics |
| --- | --- |
| `target.protocol` | `http:` or `https:`; must match the request URL. |
| `target.origin` | Exact logical origin. |
| `target.hostname` | Normalized logical hostname. |
| `target.port` | Effective port from 1 through 65535. |
| `route.mode` | `direct`, `environment-proxy`, `explicit-proxy`, or `managed-proxy`. |
| `route.resolution` | `pinned`, `proxy`, `connection-owner`, or `caller`. |
| `route.tls` | `required` for HTTPS and `cleartext` for HTTP. |
| `addressPolicy.mode` | `public-only`, `trusted-host`, or `allow-private-network`. |
| `trustedHostnames` | Exact trusted hostnames. |
| `hostnameAllowlist` | Exact additional allowed hostnames. |
| `allowedPrivateCidrs` | Explicit private CIDR grants. |
| `allowRfc2544BenchmarkRange` | Whether RFC 2544 benchmark addresses are granted. |
| `allowIpv6UniqueLocalRange` | Whether IPv6 unique-local addresses are granted. |
| `dnsRebinding.policy` | Must be `reject`. |
| `dnsRebinding.enforcement` | `local-pinned`, `connection-owner-required`, or `not-enforced`. |

The target tuple must match the one-hop URL exactly. Proxy route modes require
`resolution: "proxy"`. A hosted direct route uses `pinned` when enforceable
addresses are supplied or `connection-owner` when the admitted host performs
DNS and validates every result against the address policy immediately before
connecting. DNS-rebinding enforcement must match the resolution owner: pinned
local resolution uses `local-pinned`; proxy and connection-owner resolution
use `connection-owner-required`; caller resolution is rejected for hosted
dispatch. V1 never accepts `not-enforced`; physical connection must remain with
an owner that can enforce the declared address policy and DNS-rebinding posture.

`managed-proxy` is valid only when the traffic-policy decision selected the
same mode and hosted `dispatchBindingId`. It delegates physical proxy choice,
not policy authority: the host remains constrained by the exact logical
origin, address posture, TLS requirement, timeout, and private-network grants
in that decision. Other route modes must match their decision mode exactly.

Unknown network-guard fields, inconsistent target tuples, mismatched TLS
posture, unsupported address modes, and inconsistent resolution ownership fail
before operation delivery.

## Dispatch Certainty

Terminal certainty is one of:

| Certainty | Meaning |
| --- | --- |
| `not-started` | Physical dispatch did not begin. |
| `started-unconfirmed` | Dispatch may have started, but no response was confirmed. |
| `response-started` | Response metadata or body began, but completion was not confirmed. |
| `completed` | The physical response exchange completed. |

Certainty is transport evidence, not a semantic retry decision.

Side-effecting owners default to no automatic replay after any outcome other
than a proven `not-started`. Even `not-started` remains subject to owner policy.

## Terminal Outcomes

A terminal frame contains:

```json
{
  "version": "reverse-provider-dispatch/v1",
  "type": "terminal",
  "incarnationId": "carrier-incarnation",
  "operationId": "operation-id",
  "ownerGeneration": "owner-generation",
  "hostBundleGeneration": "bundle-generation",
  "policyGeneration": "policy-generation-42",
  "outcome": "failed",
  "certainty": "started-unconfirmed",
  "failureCode": "connection-lost"
}
```

Each operation settles exactly once.

Stable failure codes include:

- `denied`
- `unavailable`
- `overloaded`
- `stale-generation`
- `timeout`
- `cancelled`
- `not-started`
- `started-unconfirmed`
- `response-limit-exceeded`
- `protocol-violation`
- `connection-lost`

Failure messages are bounded and redacted.

When `failureCode` is `not-started` or `started-unconfirmed`, it must equal the
terminal certainty. Other failure codes describe the transport fault while
certainty independently records how far dispatch progressed. A completed
terminal has `certainty: "completed"` and no failure code.

## Cancellation

Cancellation is advisory.

- Cancellation before `operation-open` is delivered to the carrier may settle
  `not-started`. After delivery, only a host terminal may prove `not-started`.
- Cancellation after `dispatch-started` may settle
  `started-unconfirmed`, `response-started`, or `completed`.
- Disconnect does not prove provider rejection.
- Generation supersession rejects future frames but cannot undo a provider side
  effect.
- Reconnect never replays an in-flight operation.

When the carrier terminates before a host terminal frame, the Gateway locally
settles the operation as failed with `connection-lost`. It may synthesize
`not-started` only when `operation-open` was never delivered to the carrier.
Once delivery begins, absence of `dispatch-started` is not proof that physical
dispatch did not begin: certainty is at least `started-unconfirmed`, or
`response-started` after response metadata or bytes begin. A host terminal may
still explicitly prove `not-started`. The synthesized settlement is recorded
in bounded audit evidence.

## Validation

Receivers validate before delivering a frame:

1. JSON object shape and allowed fields.
2. Exact version and frame type.
3. Bounded non-empty common identifiers.
4. Current incarnation and owner, bundle, and policy generations.
5. Known operation and legal direction.
6. Positive limits and credit.
7. Canonical base64 and decoded byte counts.
8. Chunk sequence and stream state.
9. Frame, chunk, request, response, queue, trace, and deadline limits.
10. Legal response and terminal ordering.

Malformed or oversized frames fail without exposing payload bytes in
diagnostics.

## Operation State Machine

```text
operation-open
  -> request credit/chunks may stream
  -> dispatch-started before physical dispatch begins
  -> request chunks and request half-close may follow
  -> response-open may arrive before request half-close
  -> response credit/chunks and response half-close
  -> terminal
```

The host may begin physical dispatch and drain request chunks as they arrive;
it must not buffer the complete request before `dispatch-started` or request
`half-close`. Request and response streams may progress concurrently. A final
response may stop further request credit and trigger cancellation of the open
request stream. Terminal settlement occurs only after both stream directions
are closed or cancelled according to the reported certainty.

Cancellation or failure may move to terminal from any valid state. A terminal
operation ignores or rejects later frames according to local audit policy, but
never delivers them to the owner.

## Carrier Lifecycle

- One admitted session serves one binding in one cell.
- Session replacement creates a new incarnation.
- Authority revalidation occurs when bundle or owner state changes.
- Authority revalidation occurs when traffic policy changes.
- Revocation closes new operation admission and settles active operations
  according to certainty.
- Shutdown closes streams and releases bounded resources.
- A shared physical socket may multiplex internally only if protocol identity,
  credit, generations, and failure isolation remain cell-scoped.

V1 does not define a reusable carrier catalog.

## Diagnostics And Audit

Bounded evidence may include:

- cell-local binding id;
- operation id or redacted correlation;
- owner, bundle, and incarnation generations;
- policy generation;
- route profile;
- logical origin;
- request and response byte counts;
- frame and queue counts;
- certainty and failure code;
- timing and cancellation state.

Diagnostics must not include:

- credential values;
- authorization or cookie headers;
- request or response bodies;
- prompt contents or tool arguments;
- private keys;
- unbounded URLs, traces, or owner messages.

Rejected stale frames and locally synthesized connection-loss settlements must
include operation identity, the relevant generation or incarnation mismatch
category, certainty, and failure code in bounded audit evidence. Malformed or
unknown-operation frames include those fields only when validated bounded
values and a known operation can be established; otherwise they record only a
bounded parse or validation category and authenticated carrier identity.

## Example Trace

```text
operation-open
credit(request, 1024)
dispatch-started
chunk(request, sequence=0)
half-close(request)
response-open(status=202)
half-close(response)
terminal(outcome=completed, certainty=completed)
```

A redirect trace ends after returning the redirect response. The owner decides
whether a separately authorized next hop is allowed.

## Compatibility And Evolution

- Unknown major versions and frame types fail closed.
- New optional diagnostic metadata must not change frame semantics.
- New required fields require a new protocol version.
- New failure codes require an explicit old-client fallback rule.
- New semantic operations do not belong in this carrier; they require owner
  contracts.
- A second capability must prove value before shared carrier generalization.

## Host Dispatcher Checklist

A host dispatcher is compatible when it:

- authenticates for one cell and binding;
- activates only after a successful handshake;
- enforces current generations and incarnation;
- honors independent request and response credit;
- performs one redirect-disabled exchange;
- materializes only declared credential slots;
- enforces the supplied network guard;
- reports bounded certainty and one terminal outcome;
- does not replay on reconnect;
- redacts all credential and payload material.

## OpenClaw Owner Checklist

An owner is compatible when it:

- prepares immutable semantic bytes and trusted identity;
- selects a typed dispatcher and route profile;
- supplies exact slot references, limits, and network guard;
- validates redirect responses;
- interprets status and response bodies;
- owns replay decisions;
- rejects stale generations and route drift.

## Conformance Fixtures

The v1 suite should include:

- successful bodyless and streamed responses;
- request and response chunking;
- independent credit exhaustion;
- one binding's saturation without blocking Gateway control traffic or another
  binding;
- invalid sequence and post-half-close chunk;
- malformed and non-canonical base64;
- unknown and oversized frame fields;
- request, response, frame, chunk, queue, trace, and deadline limits;
- redirect returned without following;
- credential slot conflict and origin denial;
- public and explicitly granted private routes;
- stale owner, bundle, and incarnation frames;
- stale policy-generation frames;
- session replacement and authority revocation;
- cancellation before and after dispatch start;
- disconnect at each certainty phase;
- failure-code and certainty mismatch;
- no reconnect replay;
- terminal settlement exactly once;
- redacted diagnostics.
