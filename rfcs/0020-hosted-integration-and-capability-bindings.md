---
title: Hosted Owner Bindings and Managed Dispatch
authors:
  - Gio Lodi
created: 2026-07-10
last_updated: 2026-07-13
status: draft
issue:
rfc_pr: https://github.com/giodl73-repo/rfcs/pull/2
---

# Proposal: Hosted Owner Bindings and Managed Dispatch

## Summary

Managed hosts should integrate with OpenClaw through the semantic owner that
already understands each operation.

V1 follows one decision tree:

```text
Can the host use an existing Gateway, Channel, plugin, proxy, or service-mesh
surface?
  -> yes: use that native surface
  -> no:
       Can OpenClaw still prepare and directly dispatch the owner's request?
         -> yes: use the owner contract with local guarded dispatch
         -> no:
              prepare the same owner request
              resolve only declared credential slots
              intersect host traffic policy
              execute through one semantically empty hosted dispatcher
```

The proposal therefore has three branches:

1. **Native attachments** reuse canonical Gateway, Channel, plugin-route, proxy,
   and lifecycle surfaces.
2. **Owner contracts** prepare exact requests, identity, credentials,
   response policy, and replay semantics.
3. **Hosted dispatch** replaces only physical network execution when the host
   must execute an already authorized request.

A request may compose all three branches. The decision tree selects the
physical execution path; owner preparation, traffic policy, and network safety
still apply when hosted dispatch is selected.

A namespaced host bundle may package implementations for these independently
owned contracts. Hosting Profiles, Status, and Doctor aggregate their evidence.
Neither the bundle nor a carrier becomes a new semantic owner.

This proposal does not define ClawBus, a generic host API, a carrier catalog,
or a bidirectional extension of `OpenClawTransport`.

## Thesis

The isolation boundary is not the product boundary.

Lobster's ProxyPipe accumulated provider adaptation, credential injection,
traffic policy, Channel delivery, approvals, lifecycle, product services, and
transport framing in one reverse connection. The physical multiplexing was not
the core problem. The problem was that the pipe became the private owner of
unrelated interfaces.

The falsifiable V1 thesis is:

> OpenClaw can remove ProxyPipe by promoting semantics to their existing
> owners, adding one bounded hosted-dispatch seam for the remaining physical
> network gap, and composing readiness without introducing a replacement bus.

The thesis is disproved if a migration requires:

- a generic `{ interface, operation, payload }` host method;
- provider, Channel, approval, lifecycle, or identity semantics in the
  dispatcher;
- a host-written success state that overrides owner readiness;
- arbitrary credential mutation rather than declared slots;
- automatic replay decisions below the semantic owner; or
- a multipurpose reverse carrier before a second capability proves its value.

## Why This RFC Exists

OpenClaw normally owns its process environment, credentials, network,
filesystem, Channels, and Gateway. A managed host may intentionally isolate
some of those resources:

- credentials remain outside the container;
- approved destinations are reachable only from the host network;
- enterprise routing or private-origin policy applies;
- inbound activities enter through host-controlled routing;
- operator approvals are presented in a host UI; and
- workload readiness depends on host-provided implementations.

Most of these needs already have a canonical OpenClaw surface. The missing
piece is not one universal host protocol. It is a small set of owner-local
binding seams plus one optional physical dispatcher.

## Design Principles

### Semantic ownership

The owner that understands an operation defines:

- configuration and trusted identity;
- exact request or event shape;
- bounds and allowed destinations;
- credential requirements;
- response interpretation;
- retry and replay safety;
- generation and activation;
- readiness evidence; and
- migration authority.

Lower layers may narrow authority but never widen or reinterpret it.

### Native surfaces first

Use existing product surfaces whenever they already express the operation:

- Gateway methods and events for approvals, pairing, administration, and
  lifecycle;
- Channel-owned HTTP or WebSocket routes for ingress;
- Channel adapters for Channel behavior;
- provider request policy and guarded fetch for ordinary egress;
- proxies and service meshes for transparent routing;
- SecretRef providers for secret lookup;
- AgentHarness for prepared execution attempts; and
- Runtime State Continuity for durable publication.

Hosted dispatch is not the default integration path.

### Local and hosted parity

An owner contract is independent of where physical work runs. The same
prepared request, policy, credential declaration, response policy, and
structured failure must apply to local and hosted bindings.

### No silent fallback

When governed or hosted execution is selected, missing, stale, incompatible,
or unavailable bindings fail explicitly. OpenClaw never silently falls back to
a less-governed direct path.

### Deletion is the success metric

Every adoption slice names the downstream code, frames, flags, or overlays it
makes removable. Registration without an adoption and deletion gate is not a
complete migration.

## Branch 1: Native Attachments

The first branch uses canonical product surfaces and adds only the metadata or
packaging needed for managed operation.

### Gateway clients

Hosts use `OpenClawTransport` and canonical Gateway methods/events when acting
as applications or operators.

`OpenClawTransport` remains:

```ts
type OpenClawTransport = {
  request<T>(
    method: string,
    params?: unknown,
    options?: GatewayRequestOptions,
  ): Promise<T>;
  events(filter?: (event: GatewayEvent) => boolean): AsyncIterable<GatewayEvent>;
  close?(): Promise<void> | void;
};
```

It is not extended with reverse callbacks, provider registration, Channel
ingress, lifecycle ownership, or AgentHarness streams.

Approval presentation, approval resolution, pairing, observation, and
administration use separate least-privilege Gateway identities even when a
host shares connection infrastructure.

### Channel and plugin routes

Inbound host delivery terminates at a Channel- or plugin-owned route. The owner
defines:

- route identity;
- Gateway, provider-direct, or owner-specific authentication;
- body and framing bounds;
- sender and tenant policy;
- idempotency source;
- acknowledgement and retry classification;
- reload behavior; and
- readiness.

The host forwards an owner payload. No generic `channel.ingress` envelope is
introduced.

Gateway-authenticated plugin routes are the default trusted-forwarder seam when
the host already participates in the Gateway deployment. A Channel may define
stronger provider-direct authentication where available.

### Traffic governance

Transparent egress governance stays in provider request policy, proxies,
firewalls, TLS policy, service meshes, and guarded fetch.

Host policy may:

- deny a destination;
- narrow origins or private-network grants;
- reduce timeout or limits;
- select an approved route profile; or
- attach redacted provenance.

It may not:

- weaken OpenClaw hard safety;
- inspect credential values;
- rewrite semantic request bodies;
- change provider response meaning; or
- create a remote per-request policy callback when compiled policy is enough.

## Branch 2: Owner Contracts

When Lobster currently owns request semantics, those semantics move into the
relevant OpenClaw owner.

There is no universal provider-adapter registry. Model providers, Channels,
web tools, and other owners expose their own request contracts through their
existing package or registry boundaries.

### Prepared owner request

An owner prepares:

```text
owner and operation version
trusted owner identity
method
exact logical URL
non-secret headers
bounded body bytes or stream
credential slot references
response policy
replay policy
timeout and byte limits
owner generation
```

The prepared request is immutable. Local and hosted dispatch consume the same
owner result.

The owner validates all semantic inputs before dispatch. Examples include:

- CAPI request defaults and endpoint selection;
- Substrate body and token audience;
- WebIQ query bounds and response filtering;
- Anthropic request and SSE response semantics;
- ACF Activity validation and Bot Framework origins; and
- Microsoft Graph mailbox identity, URL paths, exact `202 Accepted`, and
  no-replay mail semantics.

### Trusted identity

Routing data is not authority.

Trusted mailbox, tenant, user, provider, or agent identity comes from owner
configuration or another trusted owner binding. Inbound payloads may attest to
that identity but may not override it.

The dispatcher does not receive a generic identity assertion that lets it
select a mailbox, tenant, user, provider, or credential.

### Credential slots

V1 credential slots are exact, origin-bound header placements:

```text
slotId
placement: header
headerName
allowedOrigins
required
resolver version
```

The owner declares the slot and destination. A resolver materializes only the
credential value. The dispatcher may place that value only when:

- the owner declared the slot;
- owner config selected the resolver;
- placement and header name match;
- the current origin is explicitly allowed;
- audience, expiry, revocation, and generation checks pass; and
- traffic policy permits the request.

When credential subject or tenant is authoritative, the owner must bind the
selected resolver to its declared trusted identity. It must either validate a
verifiable credential subject/audience or select a slot that is statically
scoped to exactly that identity. Origin and audience checks alone are not an
identity binding.

The following are outside V1 until a real owner defines a narrow contract:

- arbitrary header mutation;
- query-string credentials;
- body credentials;
- computed request signing;
- multiple-header credential recipes; and
- generic SecretRef expansion inside the dispatcher.

### Response and replay policy

The owner, not the dispatcher, interprets provider status and decides whether
an operation can be retried.

The prepared response policy may define:

- accepted status or response classes;
- response byte and stream limits;
- owner-specific error classification;
- bounded provider retry advice;
- replayability; and
- dispatch-certainty requirements.

Side-effecting operations default to no automatic replay. A timeout,
disconnect, or server error does not prove the provider rejected the request.
For example, a Graph mail POST may have been accepted before its response was
lost.

## Branch 3: Hosted Physical Dispatch

Hosted dispatch is used only when:

- OpenClaw cannot directly reach the approved destination;
- a standard proxy or service mesh is insufficient; or
- a host-managed credential must be materialized at the physical execution
  boundary.

The dispatcher is semantically empty. It receives an already prepared and
authorized request and performs one physical exchange.

### Dispatcher contract

The dispatcher receives:

```text
operation ID
binding ID
owner generation
host bundle generation
deadline
method
logical URL
headers
credential slot references
body bytes or bounded stream
request and response limits
route profile
network guard profile
audit correlation
```

It may:

- resolve physical DNS in the host network;
- enforce the supplied network guard profile;
- select the authorized proxy or route profile;
- materialize declared credential slots;
- execute one redirect-disabled request;
- stream a bounded response;
- report dispatch certainty; and
- return redacted network evidence.

It may not:

- choose or rewrite the semantic destination;
- follow redirects automatically;
- change methods or bodies;
- add undeclared credentials;
- interpret provider statuses;
- retry semantic operations;
- select mailbox, tenant, user, Channel, or model identity; or
- retain credentials across requests or redirect hops.

### Network safety

OpenClaw owns the network policy. The dispatcher enforces it where physical DNS
resolution and connection occur.

The fixed-shape guard profile covers:

- required scheme and exact logical origin;
- public, private, loopback, link-local, and reserved address posture;
- approved private origins or CIDRs for named route profiles;
- DNS rebinding and address-selection requirements;
- proxy or service-mesh constraints; and
- timeout and connection limits.

Every redirect returns to the owner/guard loop for a new decision. Hosted
dispatch never follows redirects internally.

Hosted dispatch assumes that the admitted host runtime is trusted to enforce
the supplied guard. OpenClaw can verify policy inputs, generations, and
reported evidence, but it cannot prevent a compromised host that already
receives credentials from ignoring the destination or guard. Local dispatch
remains inside OpenClaw's enforcement boundary.

### Dispatch certainty

Transport outcome distinguishes at least:

```text
not-started
started-unconfirmed
response-started
completed
```

Cancellation is advisory. Reconnect does not replay in-flight work. Only the
semantic owner can use certainty, idempotency, and response policy to decide a
later retry.

### Carrier

The dispatcher may run in-process, through a local sidecar, through an explicit
proxy, or across a dedicated provider-dispatch reverse session.

V1 defines no carrier catalog. If a reverse session is required, it is
provider-dispatch-specific and supports:

- authenticated admission;
- bounded bidirectional streams;
- independent flow control;
- half-close and cancellation;
- one terminal result;
- connection-incarnation tracking;
- generation fencing; and
- no reconnect replay.

Existing Gateway or peer machinery may be reused internally. A host is not a
node, an operator, or a generic Gateway service.

A multipurpose carrier is deferred until a second capability proves that
shared framing and connection management remove more complexity than they add.

## Composition Without Centralized Semantics

### Host integration bundle

A host package may register one immutable, namespaced inventory of
implementations:

```jsonc
{
  "id": "lobster-host",
  "version": "1.0.0",
  "contracts": {
    "channelEndpoints": ["lobster/teams"],
    "trafficPolicies": ["lobster/enterprise-egress"],
    "credentialSlotResolvers": [
      "lobster/capi-token",
      "lobster/substrate-token",
      "lobster/acf-token",
      "lobster/graph-token"
    ],
    "providerRequestDispatchers": ["lobster/egress"]
  }
}
```

The exact manifest syntax is not normative.

The normative rules are:

1. the full bundle snapshot is validated before contributions become visible;
2. contributions register with their real semantic owners;
3. existing owner config selects typed implementation IDs;
4. each owner validates compatibility and authority;
5. owner activation remains independent;
6. missing, duplicate, disabled, or incompatible references fail explicitly;
7. configured governed bindings never silently fall back; and
8. the bundle supplies packaging and provenance, not semantics.

Future SecretRef, publication, or telemetry implementations may register in
the same package without sharing the provider-request interface.

### Binding selection

For each owner scope:

```text
bundle registers implementation
  -> owner config selects typed ID
  -> owner prepares immutable binding
  -> dependencies become ready
  -> owner activates one authoritative generation
  -> owner publishes readiness evidence
```

There is no cross-owner distributed activation transaction. Hosting Profiles
observe owner results; they do not construct or activate bindings.

### Generations

Remote operations carry:

- **owner generation**, identifying authoritative owner configuration;
- **host bundle generation**, identifying the admitted implementation set; and
- **binding or carrier incarnation**, identifying the current live route.

A newer owner generation supersedes the old binding before side effects. A new
host bundle generation requires re-admission. Same-generation reconnect may
replace an incarnation but never replay in-flight work.

Late or stale results are rejected and audited. A request that was already
`started-unconfirmed` when its generation was superseded remains an
owner-recorded indeterminate outcome; rejecting its late result does not undo
a provider side effect or make replay safe.

### Readiness, Status, and Doctor

This RFC reuses Hosting Profiles.

Each owner publishes trusted evidence such as:

```text
owner and contract version
desired and resolved implementation
owner generation
host bundle generation
binding incarnation
required or advisory posture
desired, effective, and observed state
configuration and policy provenance
migration authority
structured conditions and failure
last transition
```

Owner criteria are advisory by default. A named Hosting Profile promotes the
criteria required for a deployment.

Bundle health can explain a shared root cause but cannot override a failed
owner binding. Readiness remains fast, bounded, and non-mutating. Status and
Doctor provide richer redacted diagnostics and repairs.

## Security Model

### Least privilege

Gateway clients, Channel forwarders, credential resolvers, and provider
dispatchers use separate logical identities and permissions even if they share
host infrastructure.

A remote binding proves:

- issuer and deployment audience;
- implementation identity;
- allowed owner contract, version, and operations;
- runtime or tenant binding where applicable;
- owner and host generations;
- issued-at, expiry, and credential identifier; and
- proof of possession.

Unknown contracts, versions, operations, routes, and generations fail closed.

### Authority intersection

Effective authority is the intersection of:

```text
owner semantics and configuration
  intersect selected implementation declaration
  intersect credential grant
  intersect host traffic policy
  intersect network guard
  intersect current generations
```

No layer can widen an earlier grant.

### Credential exposure

A host that physically executes a request may observe the final credential
header. That data-plane exposure does not grant authority to mint, resolve,
reuse, log, or broaden credentials.

Credentials remain redacted from status, failures, captures, and audit output.

## Failure Model

Owners define domain failures. Bindings add portable availability failures:

```text
Unavailable
Overloaded
TimedOut
StaleGeneration
RouteChanged
InvalidResult
CredentialUnavailable
PolicyDenied
```

Each binding declares bounded in-flight work, queue behavior, request bytes,
response bytes, and stream limits. Saturation in one owner binding must not
block Gateway control traffic or another binding.

Timeout does not imply cancellation. Disconnect does not imply rejection.
Retries remain owner decisions.

## Migration Strategy

### Paired owner and adopter slices

Each migration is a pair:

1. **Owner slice** in OpenClaw:
   - promote or define the semantic owner contract;
   - preserve local behavior;
   - add an inactive hosted binding;
   - add local/hosted fixtures and structured failures; and
   - name the downstream deletion target.
2. **Adopter slice** in Lobster:
   - register the implementation or credential resolver;
   - bind trusted host identity and policy;
   - activate a managed canary;
   - prove rollback and redacted evidence; and
   - delete or expiry-gate the old path.

The owner slice does not activate host authority by itself. The adopter does
not reimplement owner semantics.

### Single-writer states

Side-effecting migrations use:

```text
old-authoritative
  -> shadow-new
  -> new-authoritative
  -> old-removed
```

Shadow paths may compare immutable preparation, policy decisions, or
non-side-effecting outcomes. They do not duplicate live side effects.

Rollback explicitly changes authority. It is never an implicit fallback after
an error.

### ProxyPipe mapping

| ProxyPipe family | Canonical destination |
| --- | --- |
| Provider adaptation | Relevant model, Channel, web, or plugin owner |
| Request credentials | Exact owner-declared credential slot resolver |
| Brokered physical egress | Provider-request dispatcher binding |
| Enterprise traffic policy | Provider request policy, proxy, or service mesh |
| Teams and mail ingress | Gateway-authenticated Channel/plugin route |
| Approvals and pairing | Gateway methods and events |
| Lifecycle | Gateway lifecycle and Hosting Profiles |
| AgentHarness traffic | AgentHarness |
| Publication | Runtime State Continuity |
| Product services | Owning plugin or Lobster API |

A temporary compatibility adapter may carry several contracts over one old
connection. Its private frame union never becomes the canonical interface.

### Deletion gate

Old code is removable only after:

- owner and adopter conformance pass;
- trusted identity and credential checks pass;
- generation rollover and stale-result checks pass;
- timeout, disconnect, overload, and certainty behavior pass;
- authoritative canary and rollback pass;
- minimum supported versions are declared;
- old-path telemetry shows no required traffic; and
- fallback expiry is reached.

## Implementation Evidence

The implementation series has exercised the owner-contract branch, native
attachments, credential slots, and the dispatcher boundary. The dedicated
reverse carrier and some adopter activations remain future proof points.

| Slice | Status and evidence | Architectural result |
| --- | --- | --- |
| Gateway and approval work | Existing canonical Gateway methods/events and native approval consumer behavior | No approval protocol or reverse callback API is needed |
| Channel endpoint work | Existing owner routes and Gateway authentication | No generic Channel ingress protocol is needed |
| CAPI and Substrate | Owner/adopter request preparation, exact token slots, policy intersection, and local/hosted dispatcher fixtures | Provider semantics stay out of the dispatcher |
| WebIQ | Completed owner/adopter query, default, filtering, and API-key-slot slices | Web tools use their own owner surface, not a universal adapter registry |
| Anthropic | Completed owner/adopter request and streamed-response slices with exact key/origin binding | Streaming semantics do not belong in the carrier |
| ACF | Completed owner and credential-resolver slices for Channel-owned Activity bytes | Tenant/user/Channel identity stays off the dispatch wire |
| Microsoft Graph mail | Owner complete; adopter activation remains next. Owner proof covers trusted mailbox selection, exact production origins, exact `202`, no replay, and Gateway-authenticated ingress | Side-effect certainty and mailbox authority stay with the owner |

These slices support the simplified strategy:

- native product surfaces handle most host integration;
- owner contracts are the reusable abstraction;
- credential slots can remain narrow;
- one semantically empty dispatcher contract covers the proven physical
  execution boundaries;
- hosted activation belongs in paired adoption work; and
- carrier mechanics can remain optional and subordinate.

The evidence does not yet claim production proof of the dedicated reverse
carrier or Graph hosted activation.

## Conformance

Every owner contract supplies fixtures that prove:

- schema and version validation;
- local and hosted preparation equivalence;
- trusted identity and mismatch failure;
- exact origins and credential placement;
- immutable request bytes or bounded streams;
- policy intersection and no weaker fallback;
- generation supersession;
- response classification and replay policy;
- redaction and audit output; and
- compatibility with shared language-neutral fixtures when a remote
  non-TypeScript implementation is claimed.

Hosted dispatcher conformance additionally proves:

- redirect-disabled one-hop execution;
- physical DNS and network-guard enforcement;
- public and explicitly granted private-route behavior;
- credential-slot conflicts and origin denial;
- overload and independent queue isolation;
- cancellation and dispatch certainty;
- stale generation and route-change rejection;
- bounded request and response streaming; and
- no reconnect replay.

Bundle and Hosting Profile conformance proves:

- one package can register several independently owned contracts;
- owner references resolve only compatible registrations;
- missing or disabled registrations fail explicitly;
- bundle health cannot override owner failure;
- required criteria keep the workload non-ready when false or unknown; and
- readiness, Status, and Doctor derive from consistent owner evidence.

Transport conformance alone is insufficient.

## Implementation Plan

The remaining work follows deletion value rather than framework layers:

1. finish paired owner/adopter migrations for Graph and remaining provider
   families;
2. activate each binding behind an explicit canary and rollback authority;
3. project stable owner evidence through Hosting Profiles, Status, and Doctor;
4. implement the provider-dispatch reverse session only for deployments where
   proxy or direct networking cannot satisfy the dispatcher contract;
5. delete each ProxyPipe family after its own expiry gate; and
6. consider shared carrier extraction only after another real capability
   demonstrates duplicated operational cost.

The implementation does not need to complete a generic carrier, universal
provider registry, or global host activation framework before deleting useful
ProxyPipe paths.

## Non-Goals

- Extending `OpenClawTransport` into a bidirectional host protocol.
- Defining ClawBus or a generic host service bus.
- Defining a universal provider-adapter registry.
- Defining one generic host method or callback API.
- Treating a host as a node, operator, Channel, or semantic owner.
- Copying Gateway, Channel, lifecycle, continuity, or AgentHarness schemas into
  a carrier envelope.
- Requiring hosted dispatch when direct networking, a proxy, or a service mesh
  is sufficient.
- Supporting arbitrary credential mutation in V1.
- Automatically replaying side-effecting requests after uncertain outcomes.
- Activating all owner bindings in one distributed transaction.
- Solving every ProxyPipe family before deleting any of them.

## Rationale

### Why owner contracts instead of provider adapters?

The implementation evidence spans model providers, web tools, Channels, and
mail. They share prepared request and binding invariants, but not one owner
registry or one semantic payload. "Owner contract" preserves that distinction.

### Why retain a host bundle?

One package, version, provenance chain, and installation path make a host's
offerings operable. Typed owner references and owner-local activation prevent
that packaging boundary from becoming a central protocol.

### Why retain Hosting Profiles?

Hosted deployments need one admission result across many owners. Hosting
Profiles already aggregate required and advisory criteria without taking
ownership of the underlying interfaces.

### Why not start with the reverse carrier?

Approvals use Gateway. Ingress uses owner routes. Policy often uses proxies or
service meshes. Many provider requests can dispatch locally. Only the residual
case where the host must physically execute an approved request needs the
reverse session.

### Where should ProxyPipe's convenience survive?

In:

- one installed host bundle;
- typed references from owner configuration;
- generated validation and conformance fixtures;
- one named Hosting Profile;
- one joined Status and Doctor view; and
- paired migration plans with explicit deletion gates.

It should not survive as one private semantic envelope.

## Unresolved Questions

- Which existing package manifests and owner registries should carry each
  remaining contribution?
- What is the smallest stable public network-guard profile for hosted physical
  dispatch?
- Which status fields are portable contract and which remain operator-only?
- Which provider families require streaming in V1 rather than bounded bytes?
- Which deployment topology first requires the dedicated reverse dispatcher
  instead of direct networking or a standard proxy?
- After Graph adoption, which ProxyPipe family yields the next largest deletion
  for the smallest owner contract?
