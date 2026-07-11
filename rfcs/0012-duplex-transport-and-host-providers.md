---
title: Duplex Transport and Host Providers
authors:
  - Gio Lodi
created: 2026-07-10
last_updated: 2026-07-10
status: draft
issue:
rfc_pr: https://github.com/giodl73-repo/rfcs/pull/2
---

# Proposal: Duplex Transport and Host Providers

## Summary

Extend OpenClaw's Gateway transport model with a sibling `host-provider` peer
that can receive bounded, capability-negotiated runtime-to-host invocations and
return typed results. Reuse the proven correlation, timeout, connection-binding,
and result machinery behind `node.invoke`, while giving host providers distinct
identity, permissions, generation fencing, namespaces, and conformance. This
allows Docker supervisors, Kubernetes operators, managed platforms, and OCC
runtime cells to provide services such as egress, secrets, durable publication,
or telemetry without inventing private runtime protocols.

## Motivation

The public `OpenClawTransport` contract and Gateway already support the normal
client direction:

```text
client/host  -- request --> OpenClaw
client/host  <-- event  --- OpenClaw
```

Serious hosting platforms also need the reverse application direction:

```text
OpenClaw -- bounded request --> host service
OpenClaw <-- typed result   --- host service
```

Without an upstream facility, hosts place unrelated services on private
sidecars, callbacks, or multiplexed pipes. The transport then begins defining
copies of Gateway approvals, Channels, AgentHarness, lifecycle, and product
semantics. Every new capability couples runtime and host releases and weakens
OpenClaw's support and conformance model.

OpenClaw already contains a close precedent. A paired node advertises commands
and capabilities; Gateway sends `node.invoke.request`; the node returns
`node.invoke.result`; pending calls are tied to a connection, timed out, and
failed on disconnect. Duplex Transport should reuse that peer-invocation
foundation rather than add another callback bus.

Hosts are not nodes, however. Node identity, pairing, mobile wake, local-machine
commands, and device policy are the wrong trust model for infrastructure. A
separate host-provider peer makes the reusable mechanics explicit without
turning a hosting platform into a paired user device.

This also aligns with OCC. OCC may admit and provision provider peers, but
runtime-to-host calls remain in the runtime/data plane rather than entering the
control plane.

## Goals

- Add a least-privilege host-provider peer distinct from operators and nodes.
- Support Gateway-to-provider unary request/result invocation.
- Reuse shared peer correlation, timeout, connection, and late-result behavior.
- Negotiate provider IDs, versions, methods, and optional capabilities.
- Bind provider sessions and invocations to an active deployment/runtime
  generation.
- Separate provider permissions from operator scopes and node pairing.
- Require proof-of-possession admission for remotely useful provider authority.
- Define provider namespaces, schemas, limits, failure behavior, and audit data.
- Give plugins a namespaced provider registration path without allowing them to
  claim core namespaces.
- Provide conformance tests usable by TypeScript, Rust, gRPC, WebSocket, Docker,
  Kubernetes, managed hosting, and OCC implementations.
- Provide a migration path from private host pipes without upstreaming their
  wire formats.

## Non-Goals

- A replacement for Gateway client requests and events.
- A new approval or node-pairing protocol.
- Modeling host providers as nodes.
- Carrying Teams-specific semantics; messaging remains a Channel concern.
- Redefining AgentHarness events or terminal behavior.
- Putting OCC in the runtime hot path.
- A generic arbitrary callback bus.
- Provider-specific backend implementation.
- Streaming, resumability, or remote cancellation in the first implementation.
- A single `hosted-openclaw` envelope containing every hosting concern.

## Proposal

### Shared peer invocation substrate

Extract or share the general machinery currently demonstrated by
`NodeRegistry.invoke`:

- peer session registration and connection identity;
- declared and effective capability/method ceilings;
- random request correlation IDs;
- request deadlines and pending-call cleanup;
- connection-change and disconnect failure;
- bounded request/result envelopes;
- harmless late-result handling;
- transport liveness and slow-consumer behavior.

Keep existing `node.invoke.*` wire names and behavior compatible. Add a sibling
host-provider surface over the shared implementation.

```text
Gateway peer invocation
      |                         |
      v                         v
node.invoke                host.invoke
node identity/pairing      provider identity/admission
node command policy        provider permissions
device commands            host service methods
```

### Host-provider role

Add `host-provider` to the closed Gateway role set. A host-provider:

- has no operator scopes;
- cannot call operator or node-only methods;
- does not enter node/device pairing;
- advertises a bounded provider surface;
- receives only invocations authorized for that surface;
- returns results through host-provider-only methods;
- is removed or superseded when its hosted generation becomes stale.

An operator connection remains separate. A hosting platform may maintain both:

```text
operator connection
  approvals, pairing, status, lifecycle administration

host-provider connection
  egress, secrets, publication, telemetry, plugin host services
```

Combining those authorities is discouraged and should not be required by the
protocol.

### Provider declaration

Connect parameters gain an additive provider declaration, conceptually:

```json
{
  "role": "host-provider",
  "client": {
    "id": "host-provider",
    "mode": "service",
    "version": "1.0.0",
    "platform": "linux",
    "instanceId": "provider-instance-7f2"
  },
  "providers": [
    {
      "id": "egress",
      "version": 1,
      "methods": ["fetchMetadata"]
    }
  ],
  "hostBinding": {
    "deploymentId": "deployment-123",
    "runtimeId": "runtime-456",
    "generation": "generation-9"
  }
}
```

Names are illustrative. Provider declarations are untrusted input. Gateway
computes the effective surface as:

```text
declared by peer
  intersect credential grants
  intersect configured host requirements/allowlist
  intersect protocol-version support
```

Like node command ceilings, an active connection may be narrowed but never
expanded beyond its declaration and credential grants.

### Admission and identity

Host-provider authority uses a short-lived, capability-bound credential tied
to:

- issuer and Gateway/deployment audience;
- provider peer identity or public-key thumbprint;
- allowed provider IDs, versions, and methods;
- tenant/runtime binding when applicable;
- active host/runtime generation;
- issued-at, expiry, and credential ID.

The RFC specifies validation semantics, not a mandatory JWT format.

Reuse the Gateway challenge/nonce pattern for proof of possession. The peer
signs a canonical payload containing its identity, role, declared provider
surface digest, host binding/generation, credential identifier or digest,
server nonce, and timestamp.

Shared Gateway passwords, trusted-proxy user headers, operator device tokens,
and node pairing are insufficient provider credentials because they do not bind
this authority and lifecycle.

Deployment paths may include:

- a one-time local bootstrap credential for a colocated Docker supervisor;
- a managed credential provisioned with Gateway startup by Lobster or OCC;
- explicit administrator enrollment for a remote third-party provider.

Interactive device-style pairing is not the default for infrastructure peers.

### Generation fencing

Every provider session and pending invocation carries both Gateway connection
identity and hosted generation.

1. A host/controller provisions expected generation `G`.
2. Gateway admits only a credential for `G`.
3. A newer valid generation atomically supersedes the previous provider peer.
4. Calls on the old connection fail with a stable stale-peer/route-changed
   result.
5. Late old-generation results are ignored and audited.
6. Credential or expected-generation rotation closes stale sessions.

Local deployments may use a process/startup generation. Managed deployments
may supply a lease or revision. The portable contract does not require any
particular host storage or lease system.

### Provider registry and permissions

Core providers use reserved namespaces such as:

```text
host.egress.v1
host.secrets.v1
host.publication.v1
host.telemetry.v1
```

Plugin providers use:

```text
plugin.<plugin-id>.<service>.v1
```

Each provider descriptor declares:

- provider ID and version;
- methods and request/result schemas;
- permission identifiers;
- required and optional availability;
- timeout and payload bounds;
- idempotency/replay behavior;
- redaction and audit metadata.

Plugins cannot claim core namespaces or register methods for another plugin.
Unknown provider IDs, methods, versions, or permissions default deny.

Provider permissions are distinct from operator scopes, for example:

```text
host.egress.fetch
host.secrets.resolve
host.publication.publish
host.telemetry.emit
```

### Provider selection and overload

For each provider ID and hosted generation, Gateway selects at most one active
route unless that provider descriptor explicitly defines a core-owned routing
mode. V1 uses deterministic single-route selection; providers do not load
balance themselves by racing responses. A newly admitted route supersedes the
old route atomically, and pending calls fail with `ProviderRouteChanged` rather
than being replayed implicitly.

Each descriptor defines maximum in-flight calls, request/result byte limits,
and a bounded queue policy. When the active route is absent, saturated, stale,
or slow, Gateway returns stable `ProviderUnavailable`, `ProviderOverloaded`,
`ProviderStale`, or `ProviderTimedOut` results. It does not allow unbounded
memory growth or let one provider starve Gateway control traffic. Retry remains
the caller's decision and is permitted only for descriptor-declared idempotent
methods using the same idempotency key.

### Invocation protocol

Gateway emits a host invocation event equivalent to:

```json
{
  "id": "request-uuid",
  "provider": "host.secrets.v1",
  "method": "resolve",
  "params": {},
  "deadlineMs": 30000,
  "idempotencyKey": "...",
  "subject": {
    "agentId": "agent-1",
    "sessionKey": "..."
  }
}
```

The provider returns through a host-provider-only result method:

```json
{
  "id": "request-uuid",
  "ok": true,
  "payload": {}
}
```

Request subject context is derived from authenticated OpenClaw runtime state.
The provider cannot choose tenant/user routing by echoing display fields.

The first version follows `node.invoke` lifecycle semantics:

- timeout resolves the Gateway waiter with a stable timeout result;
- disconnect fails pending requests;
- connection/generation change fails dispatch;
- late results are ignored and audited;
- retries occur only where the provider descriptor defines idempotency.

Timeout does not claim to stop remote work. Explicit remote cancellation is a
future protocol extension.

### SDK shape

The existing client transport remains compatible. A provider-capable client
adds handler registration conceptually:

```ts
interface OpenClawHostProviderTransport extends OpenClawTransport {
  registerProvider(provider: HostProvider): Disposable;
}
```

The concrete API should align with current SDK registration and event-loop
patterns. Non-TypeScript implementations consume the same published Gateway
schemas and conformance fixtures.

### Streaming extension

The unary foundation intentionally does not solve large HTTP bodies,
AgentHarness streams, or publication streams. A later extension must define:

- stream open/accept/reject;
- ordering and sequence behavior;
- credit/backpressure and maximum buffered data;
- cancellation and terminal acknowledgement;
- half-close semantics;
- disconnect/reconnect and non-resumable versus resumable behavior.

AgentHarness remains the owner of execution payload semantics even if it later
uses the transport's stream facility.

### Conformance

Reusable tests should prove:

- role and method default-deny behavior;
- credential audience, expiry, proof of possession, and provider grants;
- declaration/grant/config intersection;
- generation supersession and stale-result rejection;
- request/result correlation and payload validation;
- deadline, disconnect, overload, and late-result behavior;
- namespace isolation for plugins;
- redaction and audit events;
- compatibility with a non-TypeScript provider implementation.
- deterministic route supersession and no implicit replay;
- in-flight, payload, and queue bounds under a slow or disconnected provider;
- duplicate idempotency-key behavior for methods declared idempotent;
- Gateway control-plane responsiveness while provider traffic is saturated.

### ProxyPipe migration and deletion gate

Private protocols migrate by semantic frame family, not by tunneling their
existing envelopes through `host.invoke`. For each ProxyPipe family the host
must record its OpenClaw owner and disposition:

- host service calls map to a named provider and typed method;
- user/channel messages remain Channels;
- approvals remain existing Gateway approval APIs;
- remote execution streams remain AgentHarness;
- lifecycle and continuity results remain their owning lifecycle/state
  contracts; and
- product-only operations remain host or plugin features.

A frame family may be deleted after the replacement provider/API passes shared
conformance, generation rollover, disconnect, timeout, and mixed-version tests
against the minimum supported OpenClaw release. The host must not keep a
permanent dual path. Any temporary fallback has an owner, telemetry proving
usage, an expiry release, and a removal change.

### Implementation sequence

1. Add the host-provider role, identity/admission model, provider registry, and
   one unary request/result path by sharing peer-invocation machinery. Preserve
   `node.invoke` wire behavior.
2. Add one bounded portable provider and a non-TypeScript conformance adapter.
3. Propose streaming separately after a concrete provider requires it.

Approvals migrate through existing Gateway APIs and are not implementation
prerequisites for Duplex Transport.

## Rationale

### Why not extend a private reverse pipe?

A private pipe can remain a physical implementation, but its payloads should
implement OpenClaw-owned provider contracts. Otherwise each host invents
different semantics and release compatibility.

### Why not use `node.invoke` directly?

Its invocation mechanics fit, but its public trust model does not. Treating
infrastructure as a paired device imports node wake, command approval, device
revocation, and local-machine policy into host services.

### Why a new role rather than operator scopes?

Operator scopes authorize a client to control OpenClaw. Provider permissions
authorize OpenClaw to call a service. Keeping them separate limits compromise
and makes audit intent explicit.

### Why unary first?

Unary invocation is already proven by `node.invoke` and can be bounded and
conformed independently. Designing streaming without a selected consumer would
inflate the first change and risk an under-specified protocol.

### Why provider descriptors rather than arbitrary handlers?

Descriptors give OpenClaw stable schemas, permissions, bounds, versioning, and
release tests. Arbitrary callbacks recreate the unbounded private pipe problem.

## Unresolved questions

- What should the final role, client ID, event, and method names be?
- Should host-provider credentials be issued by Gateway, loaded from managed
  config, or support both local and external issuers?
- What minimum host binding belongs in core for single-tenant deployments?
- Which first unary provider best proves portability without requiring
  streaming or duplicating an existing plugin/provider API?
- Should provider descriptors be core-only initially or exposed to plugins in
  the first implementation?
- How should a Gateway advertise required provider availability to Hosting
  Profiles and readiness conditions?
- Which audit fields are stable public contract versus implementation detail?
- Should explicit cancellation precede streaming, or be designed with the
  stream lifecycle?
