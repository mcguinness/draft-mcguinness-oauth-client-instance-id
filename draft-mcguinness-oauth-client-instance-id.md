---
title: "Client Instance Identification for Attestation-Based Client Authentication"
abbrev: "Client Instance Identification"
category: std
docname: draft-mcguinness-oauth-client-instance-id-latest
submissiontype: IETF
stand_alone: yes
ipr: trust200902
area: "Security"
workgroup: "Web Authorization Protocol"
keyword:
 - OAuth
 - client attestation
 - instance identity
venue:
  group: "Web Authorization Protocol"
  type: "Working Group"
  mail: "oauth@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/oauth/"
  github: "mcguinness/draft-mcguinness-oauth-client-instance-id"
  latest: "https://mcguinness.github.io/draft-mcguinness-oauth-client-instance-id/draft-mcguinness-oauth-client-instance-id.html"
author:
 - fullname: Karl McGuinness
   organization: Independent
   email: public@karlmcguinness.com
normative:
  ATTEST: I-D.ietf-oauth-attestation-based-client-auth
  RFC6749:
  RFC6750:
  RFC7519:
  RFC7662:
  RFC8693:
  RFC8705:
  RFC8725:
  RFC9449:
informative:
  ACTOR-PROFILE: I-D.mcguinness-oauth-actor-profile
  ATTESTER-ENDORSEMENT:
    title: "OAuth 2.0 Client Attester Endorsement"
    target: https://mcguinness.github.io/draft-mcguinness-oauth-client-attesters/draft-mcguinness-oauth-client-attesters.html
    author:
      - fullname: Karl McGuinness
    date: 2026-09-14
  AAUTH: I-D.hardt-oauth-aauth-protocol
  RFC2104:
  CIMD: I-D.ietf-oauth-client-id-metadata-document
  RFC7591:
  RFC7636:
  RFC8252:
  SPIFFE-OAUTH: I-D.ietf-oauth-spiffe-client-auth
  RFC8417:
  RFC8935:
  RFC9068:
--- abstract

This specification defines an optional claims profile of OAuth 2.0
Attestation-Based Client Authentication. Selecting it requires an
attester-qualified client instance identifier that remains stable
across verified key changes, and adds continuity and privacy rules for
that identifier. Conveying instance context in tokens and introspection
responses remains optional within the profile. Authentication and proof
methods follow the base specification.

--- middle

# Introduction

Attestation-Based Client Authentication {{ATTEST}} answers one
question: is this an authorized Client Instance in possession of this
key? This profile adds a second: is this the same Client Instance the
Receiver previously encountered? After a key change, a new attestation
alone cannot distinguish a continuing installation from a new instance,
and treating them as the same can merge unrelated audit histories and
status decisions.

Assigning each instance its own `client_id` with a shared `software_id`
{{RFC7591, Section 2}} is an alternative. This profile instead targets
deployments that share one Logical Client, one metadata URL when using
{{CIMD}}, and authorization server (AS) policy keyed by that client;
`software_id` correlates registrations but defines no shared grants or
policy across separate client identities.

The profile adds two claims:

* `client_instance_id`: identifies a particular client installation or
  runtime in its Client Attestation. The attester assigns it, retains
  it across verified key changes, and scopes it to a Receiver by default.
* `client_instance`: carries a mapped reference to that instance in a
  token or introspection response, so a resource server can correlate
  requests with the instance validated at issuance without receiving
  the attestation.

ATTEST alone is sufficient when correlation need only last for the
current key or can remain internal to one system.

This profile establishes instance identity and its continuity. It does
not define what authority, if any, follows from that identity. Instance
evidence grants no authority; authorization profiles can use validated
instance identity or Instance Context as a policy input, subject to the
prohibitions in {{processing}}.

Five roles implement this profile: Client Attesters, clients,
Receivers, token issuers conveying context, and Context Consumers.
{{conformance}} states what each implements and where.

## Identity and Scope

| Identity | Purpose |
|---|---|
| Logical Client (`client_id`) | Identifies the OAuth client |
| Authorization principal | Identifies the subject or delegated actor |
| Client Instance | Identifies one particular installation or running copy of the client software |

The deployment chooses the instance granularity:

* **Installation:** one installation, such as a harness on a managed
  laptop, retaining its identity across process restarts.
* **Execution:** one process, container, or scheduling unit such as a
  Kubernetes Pod, retaining its identity for that unit's lifetime.

Enrollment records that choice and binds the instance to its verified
keys. Several instances can share one `client_id`.

This profile is for administratively configured deployments such as
workloads and managed desktop or mobile applications. It is not a
general-purpose device or wallet identifier and defines no enrollment,
key-rotation, or status-distribution protocol. ATTEST supplies the
authentication and proof methods, including direct resource-server
presentation ({{ATTEST, Section 7}}). When selected under
{{configuration}}, the profile applies whether the Client Attestation
is the client authentication method or an additional security signal
({{ATTEST, Section 7.6}}); it does not replace the deployment's
required client authentication.

# Conventions and Definitions

{::boilerplate bcp14-tagged-bcp14}

The terms Client Attestation, Client Attester, Client Instance, and
Client Instance Key are used as defined in {{ATTEST}}.

Logical Client:
: The OAuth client {{RFC6749}} identified by `client_id`. Several Client
  Instances can authenticate as the same Logical Client.

Attester Issuer:
: The value of `iss` in the Client Attestation, identifying the
  Client Attester.

Instance Identifier:
: The `client_instance_id` value, assigned by a Client Attester to one
  Client Instance at the configured granularity and Receiver scope
  ({{identifier-scope}}).

Source Instance Identity:
: The pair `(iss, client_instance_id)` established by a validated
  Client Attestation, pairing the Attester Issuer with the Client
  Instance's Instance Identifier. This profile keeps it stable across
  verified key changes; it is the input from which Instance Context is
  mapped.

Instance Context:
: The `client_instance` object in a token or introspection response.
  It grants no authority and does not prove current possession.

Receiver:
: A party that validates a Client Attestation under this profile, such
  as an AS or resource server. A Receiver that issues tokens carrying
  Instance Context also acts as a token issuer.

Context Consumer:
: A party that consumes Instance Context from a token or introspection
  response. It need not receive the Client Attestation.

Consumer Scope:
: The Context Consumer, or explicitly configured set of Context
  Consumers, to which one mapping of Instance Context is scoped.

Instance Context Authority:
: The token issuer identified by `iss` in Instance Context, which
  assigned the current `id`: the enclosing token issuer unless the
  context was preserved from an upstream token. The Client Attester
  remains the authority for the Source Instance Identity.

Instance Context Identifier:
: The pair `(iss, id)` in Instance Context: an Instance Context
  Authority's pairwise representation of one Source Instance Identity
  for a Consumer Scope. Like a pairwise subject identifier, it is that
  authority's own correlator for the instance, and `id` need not equal
  `client_instance_id`.

Enrollment:
: An attester-maintained record binding one instance, at the configured
  granularity, to its verified keys and assigned identifiers. It is
  separate from a user account, device registration, or Logical Client.

Identifier values in this profile are opaque. Implementations MUST
compare `iss`, `client_instance_id`, `client_instance.iss`, and
`client_instance.id` as exact, case-sensitive strings without URI
normalization, and MUST NOT derive permissions, granularity, or key
locations by parsing them.

# Profile Selection and Trust {#configuration}

The client and Receiver administratively configure:

* the Logical Client, authentication method, and attester trust policy;
* the instance granularity, continuity evidence, and freshness limits;
* the intended Receiver and any explicitly shared Receiver scope
  ({{identifier-scope}}).

A Context Consumer that requires context configures that requirement.
Claims MUST NOT select this profile or change authentication methods;
selection is part of the client-specific trust agreement, and no
discovery or metadata parameter is added. The error in {{errors}}
reports rejection, not profile discovery.

## Attester Trust

The Receiver MUST bind each approved Attester Issuer to its validation
keys and authorized Logical Clients, using configured associations or,
at an AS, client endorsements accepted under {{ATTESTER-ENDORSEMENT}}.
That profile governs acceptance at AS endpoints only; a resource server
validating attestations directly uses configured associations. A
credential's `iss`, proof of possession, or client-published metadata
({{RFC7591}}, {{CIMD}}) alone does not establish attester authority.
Key resolution follows {{ATTEST, Section 10.8}}. Local trust
withdrawal MUST take effect on subsequent authentication.

## Conformance {#conformance}

Conformance is role-specific:

| Role | Implements | Where |
|---|---|---|
| Client Attester | Claim, identifier, continuity, and enrollment requirements | {{claims}}, {{lifetime}} |
| Client | Scoped attestation use and the selected ATTEST proof method | {{identifier-scope}} |
| Receiver | Trust, validation, and grant-continuity rules | {{configuration}}, {{processing}} |
| Token issuer conveying context | Mapping, preservation, and any binding that attribution requires | {{instance-context}} |
| Context Consumer | Context validation and applicable proof checks | {{context-claims}}, {{context-errors}} |

An implementation serving several roles satisfies each. Conveying
Instance Context ({{instance-context}}) is optional and independent of
the remainder of the profile.

# Client Attestation Claims {#claims}

This profile uses the additional claims permitted by
{{ATTEST, Section 4}}; all ATTEST requirements apply, including
`typ=oauth-client-attestation+jwt`, `sub=client_id`, required `exp`
and `cnf`, and optional `iat`.

`iss`:
: REQUIRED. Exactly matches an approved Attester Issuer
  ({{configuration}}).

`client_instance_id`:
: REQUIRED. Nonempty JSON string whose UTF-8 encoding is no longer
  than 256 octets after JSON string decoding, identifying the instance
  within the attester's namespace. A URI form carries no URI semantics.
  Receivers MUST reject longer values. Assignment, scoping, and
  generation follow {{attester-requirements}}.

## Receiver Scope {#identifier-scope}

The attester MUST assign distinct identifiers per Receiver unless an
administrative agreement explicitly authorizes a shared identifier
within a named set of Receivers; a shared client or trust domain does
not by itself authorize sharing. The client MUST request and use the
attestation for that configured scope.

The client MUST also use distinct Client Instance Keys across scopes.
{{ATTEST, Section 11.1}} recommends this across authorization and
resource servers; this profile requires it across the scopes a
deployment chooses to separate, because a shared key links
attestations regardless of their identifiers. A deployment that
intends correlation across Receivers, such as an enterprise workload,
configures one scope spanning them, and one identifier and key then
suffice. Binding-key separation between Context Consumers is addressed
in {{privacy}}.

The Receiver scope is an enrollment or issuance input, not an OAuth
parameter or attestation audience, and a Receiver cannot verify it
from the identifier alone.

## Example

Example decoded attestation payload, including the optional `iat`:

~~~ json
{
  "iss": "https://attester.example/tenant/acme",
  "sub": "https://platform.example/oauth-client",
  "client_instance_id": "i-7f3d9a2e6c8145b0a923d47e18f602cd",
  "iat": 1789128000,
  "exp": 1789128300,
  "cnf": {
    "jwk": {
      "kty": "EC",
      "crv": "P-256",
      "x": "VcKVNBZ4IaBAYW3jxM4w3TJFVA7myeUGQyGt-g_yvpQ",
      "y": "f-E-hYE3TAWKwhVv9pej9NABs9SX9XsNO80x57jFTyU"
    }
  }
}
~~~

# Request Processing {#processing}

The Receiver MUST:

1. Validate the attestation and proof under the configured ATTEST
   method.
2. Validate the claims in {{claims}} and attester authority under
   {{configuration}}.
3. Associate the Source Instance Identity with the Logical Client and
   validated Client Instance Key, then apply local instance acceptance
   policy ({{errors}}).

The Receiver MUST NOT:

* substitute an identifier for proof of possession or infer it from a
  key thumbprint, certificate serial number, or JWT `jti`;
* infer continuity from equal identifiers or keys across attester
  issuers; or
* set an access token's `sub`, add `act`, or extend an actor chain
  solely from instance evidence.

Migration between attester issuers requires a procedure establishing
trust in both authorities and continuity evidence; none is defined
here. Key selection follows ATTEST; when a separate token-binding key
is used, context identifies the instance associated with the Client
Instance Key.

## Grant Continuity {#grant-continuity}

{{ATTEST, Section 10.3}} binds a refresh token to the Client Instance,
by default through the Client Instance Key. This profile does not
introduce instance-bound grants; it gives that binding an identity
that survives verified key changes and detects a copied key presented
with a different identity. For a grant established using a Client
Attestation validated under this profile, the AS MUST record the
Source Instance Identity when issuing a refresh token and, on refresh,
MUST enforce two independent invariants:

* the Source Instance Identity in the current validated attestation
  MUST match the recorded one, absent an explicitly authorized
  migration establishing continuity under {{continuity}}; and
* the proof and key binding MUST satisfy ATTEST and any applicable
  rebinding profile under {{ATTEST, Section 13}}.

Instance continuity does not imply key-binding continuity, nor the
reverse. This profile defines no migration procedure. A conflict with
the recorded identity MUST produce `invalid_grant` {{RFC6749}} without
disclosing the expected identity.

If authorization-time policy bound a code or other artifact to an
instance, the AS MUST enforce that binding at redemption, whether the
attestation is the client authentication method or an additional
security signal ({{ATTEST, Section 7.6}}). {{ATTEST, Section 10.4}}
recommends establishing such bindings where attestation is the client
authentication method.

## Attestation Errors {#errors}

Missing or invalid required instance claims and rejection by instance
policy MUST produce `invalid_client_attestation`, deliberately reusing
ATTEST's validation error so that responses do not disclose whether an
instance is known, suspended, or retired; Receivers SHOULD also avoid
distinguishable response timing. Unknown instances are rejected only
when local policy requires prior enrollment. A failed profile check
MUST NOT fall back to authentication without the required evidence.
Grant-binding errors follow {{grant-continuity}}; other errors follow
ATTEST.

# Attester Requirements {#lifetime}

Whether an instance identifier means the same thing over time depends
entirely on the attester. This section covers how it generates
identifiers, when it may retain them, how suspension works, and what
it retains.

## Identifier Generation {#attester-requirements}

For each enrollment and Receiver scope ({{identifier-scope}}), the
attester MUST assign identifiers that are:

* distinct, opaque, and never reassigned, including after retirement;
* generated from at least 128 bits of cryptographically secure
  randomness or keyed pseudorandom-function output;
* unpredictable to parties other than the attester; and
* free of runtime hostnames, user identifiers, and embedded instance
  attributes.

A URI can name the attester's namespace; its instance-specific portion
remains opaque. Keyed derivation, such as HMAC {{RFC2104}}, MUST use a
secret with at least 128 bits of entropy and unambiguously encode the
Logical Client, Receiver scope, and a unique enrollment component; a
platform-stable input alone would reproduce retired identifiers after
reinstall. Changes to derivation inputs or secrets MUST preserve
assigned values within a continuing enrollment, for which storing
those values is sufficient.

## Continuity and Lifecycle {#continuity}

Continuity is authenticated evidence sufficient for the attester to
determine that a claimant represents the same enrolled Client Instance
at the configured granularity. An attester-recorded chain of verified
key custody within one enrollment is the primary mechanism; platform
or hardware-rooted identity evidence can supplement it or, where the
deployment's evidence policy permits, supply it. Before retaining an
identifier, the attester MUST verify and record:

1. an active enrollment binding the instance, Logical Client, Receiver
   scope, granularity, and previously verified keys;
2. fresh possession of the current key and authenticated evidence
   binding it to that enrollment, including an authorized custody
   transition when the key changes; and
3. the configured continuity checks, their freshness, and observed
   lifecycle events or evidence of independent claimants.

The deployment specifies its evidence, freshness limits, and
lifecycle boundaries. For example, a container restart can leave its
Kubernetes Pod intact, but a new Pod is a new scheduling unit, and an
installation or scheduling-unit identifier does not identify its
individual processes. The attester MUST apply the following outcomes:

| Event | Required outcome |
|---|---|
| Renewal, verified key change, process restart at Installation granularity, or in-place update | Retain identifiers when continuity is verified |
| Reinstall, independent clone, replacement or restart of the selected execution unit, or granularity change | New enrollment |
| Restore or snapshot rollback | Retain only with fresh evidence that the claimant succeeds the prior holder; copied keys and data alone are insufficient |
| Suspend/resume | Apply continuity checks at the next issuance using available authenticated evidence |
| Continuity cannot be established | Require new enrollment; a continuing original can retain its own enrollment |
| Detected fork of one enrollment | Retire its identifiers, stop issuance, and enroll claimants separately |

The attester MUST NOT knowingly retain identifiers for independent
instances. Concurrent processes within one installation are not by
themselves a fork, and the rule covers detected forks, not events the
platform cannot observe ({{assurance}}). An identifier, expired
attestation, or former public key alone does not establish continuity.

## Suspension and Status {#suspension}

The attester MUST stop issuance for suspended or retired enrollments.
A Receiver suspending an instance SHOULD revoke its grants or report
their tokens inactive through introspection {{RFC7662}}. Short
attestation lifetimes narrow the window in which a suspended instance
remains acceptable. When an AS revokes a grant for instance
suspension, retirement, or attester trust withdrawal, it MUST
invalidate all access and refresh tokens associated with that grant
and prevent further refresh issuance. Without a status channel,
existing attestations remain acceptable until expiration plus clock
skew, issued tokens remain valid for their own lifetimes, and local
revocation does not notify resource servers validating tokens
offline. Security Event Tokens {{RFC8417}} delivered under
{{RFC8935}} can support a separate status integration, which this
profile does not define.

## State and Retention {#state}

Attesters MUST retain enrollment and verification records while
issuing or renewing attestations, and status while supporting
resumption; verification summaries suffice, and deleting continuity
records requires new enrollment. A validating Receiver need not
maintain an instance allowlist, but local suspension, revocation, and
mapped context require the corresponding status, token associations,
and mappings ({{mapping-stability}}). Audit retention is local policy.

# Conveying Instance Context {#instance-context}

This section is optional. It defines how a token issuer represents a
validated instance to downstream consumers, and what a consumer may
conclude from that representation.

## Format and Mapping {#context-claims}

An issuer MAY include `client_instance` in a token or introspection
response {{RFC7662}}. Its JSON object has two REQUIRED members, which
together form an Instance Context Identifier:

| Member | Type | Meaning |
|---|---|---|
| `iss` | Nonempty string | Instance Context Authority that assigned `id` |
| `id` | Nonempty string, at most 256 octets of UTF-8 after JSON string decoding | That authority's representation of the instance |

For direct issuance from a validated Client Attestation, context MUST
identify the authenticated presenting instance; on refresh, that
identity is subject to {{grant-continuity}}. Token exchange follows
{{context-exchange}} rather than inheriting this association.

Context MUST refer to validated instance participation; issuers MUST
NOT copy unvalidated client-supplied context. The issuer MUST map the
Source Instance Identity or, when remapping under {{context-exchange}},
the upstream Instance Context Identifier (together, the mapping input)
to its own namespace. Each mapping MUST:

* keep distinct instances separate unless continuity is established;
* generate `id` under {{attester-requirements}}, within the same
  length bound as `client_instance_id` ({{claims}}), and retain it
  across attestation renewal and verified key changes; and
* scope `id` to a Consumer Scope, allowing sharing only within an
  explicitly configured set.

For derived mappings, the mapping input supplies the enrollment-specific
component and the consumer supplies the scope; issuers MUST separate
this derivation from attester identifiers, for example by a distinct
key or purpose label. {{wire-examples}} shows mapped context in an
access token and an introspection response.

## Mapping Stability {#mapping-stability}

An issuer MUST NOT represent one mapping input and Consumer Scope by
more than one `id`. Because attester identifiers are never reassigned,
a new enrollment presents a new input and receives a new mapping. An
issuer MUST retain a mapping while any token or grant it issued for
that input remains valid, including clock skew. An issuer that no
longer holds or can reproduce a mapping MUST omit `client_instance`
for that input and scope rather than assign a replacement; a consumer
requiring context then rejects under {{context-errors}}. Derived
mappings need no per-instance records while the derivation secret and
inputs remain available, as for attesters under
{{attester-requirements}}; random mappings need records for as long as
the issuer intends to include context. Random generation satisfies
non-reassignment probabilistically; derivation depends on never
reusing enrollment inputs.

## Presenter Attribution {#context-binding}

Instance Context grants no authority and by itself describes only the
instance that participated in obtaining the token. When a Context
Consumer uses context to attribute the current token presentation to
that instance, the applicable consuming profile MUST require, and the
consumer MUST validate, a mechanism associating the token presenter
with the instance. For tokens issued directly from a validated Client
Attestation, sender constraint using DPoP {{RFC9449}}, mutual TLS
{{RFC8705}}, or another mechanism defined by the consuming profile,
with a key the issuer associated with the authenticated instance at
issuance, is that mechanism; this applies equally when context is
conveyed only through introspection. A token without sender constraint
supports no presenter attribution, and a consumer requiring
attribution rejects it under {{context-errors}}. The HTTP `Bearer`
scheme does not indicate an unbound token; certificate-bound tokens
{{RFC8705}} use it as well.

All validation requirements of the token-binding mechanism apply
regardless of whether context is used for attribution, including
rejection of a bound token presented without its proof
({{RFC9449, Section 7.2}}). The binding authenticates the presenter;
it does not establish that the presenter is an instance named in
context derived from an upstream token. Any presenter or key change
during exchange requires authorization under the consuming exchange
profile, and unlinkability between Context Consumers additionally
requires distinct binding keys ({{privacy}}).

## Preservation and Authorization {#context-exchange}

An exchange issuing Instance Context MUST use a consuming profile that
defines:

* whether context identifies the authenticated presenting instance or
  an instance represented by validated input-token context;
* how its association with the subject, actor, or presenter is
  validated;
* when input context is remapped or preserved; and
* refresh binding and context continuity, if the exchange issues
  refresh tokens.

Issuers SHOULD remap upstream context into their own namespace, which
keeps each consumer's view pairwise. An issuer MAY instead preserve a
validated upstream Instance Context Identifier when configured trust
and the upstream Consumer Scope authorize its disclosure to the
downstream consumer; otherwise it MUST remap or omit the context.

An issuer MUST NOT assert an upstream Instance Context Authority
unless it has authenticated both that authority's assignment of the
context and the context's association with the instance represented
by the input token. Validating the input token authenticates only its
issuer's assertions, so by default an issuer MUST limit preservation
to context whose `iss` equals the authenticated input-token issuer and
MUST remap or omit context that an intermediary itself preserved. A
consuming profile that specifies authenticated provenance and its
enforcement can permit deeper preservation; the object carries no
forwarding history, and shallow preservation is also a privacy
default.

Context identifies one instance, not a chain, and MUST NOT be treated
as a separate token or delegated actor. Remapping changes the Instance
Context Identifier but, like preservation, leaves the Source Instance
Identity it represents unchanged; an issuer MUST NOT treat upstream
context as identifying a different presenting instance, and the
upstream identifier does not establish that the issuer validated the
original Client Attestation.

## Context Consumer Processing

Before using context, the Context Consumer MUST:

1. Validate the enclosing token or authenticated introspection
   response.
2. Validate the context members, rejecting `id` values over the length
   bound in {{context-claims}} and ignoring unrecognized members.
3. Accept its authority only when it is the token issuer or an
   upstream token issuer explicitly trusted for that issuer and
   consumer.
4. Reject invalid context and, when context is required, reject the
   request if context is missing or invalid ({{context-errors}}).

A Context Consumer MUST establish the context's association with the
subject, actor, or presenter from the applicable consuming profile and
the validated token or trusted introspection configuration before
using that association in policy or audit. The `client_instance`
object alone, including whether its `iss` matches the token issuer,
does not establish it. If the association is not established, the
consumer MUST treat context only as evidence of instance participation
and MUST NOT attribute the current request to that instance; a
consumer whose configured requirement includes attribution MUST reject
under {{context-errors}}. When trusted configuration establishes that
the token issuer conveys context only from direct Client Attestation
validation, this document is the consuming profile: context identifies
the authenticated presenting instance ({{context-claims}}), and
validating the token's sender constraint ({{context-binding}})
establishes the association.

For introspection, trusted endpoint configuration identifies the
expected token issuer; a response-level `iss`, if present, MUST match
it, and neither the endpoint URL nor `client_instance.iss` selects the
issuer. Multi-issuer introspection requires a consuming profile that
authenticates the represented issuer. An authority identifier does not
authorize fetching keys from that location, and extensions MUST NOT
change the meaning of `iss` or `id`.

## Context Errors {#context-errors}

Rejection for missing or invalid required context, or for required
attribution that cannot be established, MUST use `invalid_token` at a
resource server ({{RFC6750, Section 3.1}}) or `invalid_request` for a
rejected subject or actor token in an exchange
({{RFC8693, Section 2.2.2}}). Other consuming profiles define their
own error mapping. Direct Client Attestation failures follow
{{errors}}.

# Relationship to Other Identity Systems

This section places the profile alongside mechanisms that identify
clients or workloads but not individual instances. It adds no
requirements.

## Client ID Metadata Documents {#cimd}

With {{CIMD}}, the metadata URL is the Logical Client's `client_id`,
and many installations can use it. Fetching public metadata does not
prove that a caller is an authorized instance:

| Mechanism | Contribution |
|---|---|
| CIMD | Discovers Logical Client metadata and its authentication method |
| ATTEST | Authenticates an attester-approved instance and possession of its key |
| This profile | Retains instance identity across verified key changes and conveys optional downstream context |

Use one CIMD URL for the Logical Client and keep instance identifiers
in attestations, not separate metadata documents: the attestation's
`sub` remains that URL and `client_instance_id` distinguishes its
installations ({{cimd-example}}). CIMD removes per-AS registration of
client metadata but not this profile's trust agreement
({{configuration}}). {{ATTESTER-ENDORSEMENT}} lets a client endorse
attesters through `client_attesters` metadata, subject to AS policy;
it establishes the attester-to-client association but does not select
this profile or its continuity and privacy policy.

## Workload and Agent Credentials

A shared workload identity does not identify an individual instance;
the attester needs instance-level evidence under {{continuity}}.
Direct SPIFFE Verifiable Identity Document (SVID) authentication and
client mappings follow {{SPIFFE-OAUTH}}. An AAuth Agent Provider
{{AAUTH}} or workload attester can instead issue a Client Attestation
under this profile when it holds the required enrollment evidence.
Native credentials with different `sub` or `typ` semantics require a
separate carrier profile. Instance Context is sourced only from a
validated Client Attestation ({{context-claims}}), so a deployment
authenticating with native credentials does not convey it under this
profile. {{deployment-examples}} illustrates these boundaries.

# Security Considerations

The security considerations of {{ATTEST}} and {{RFC8725}} apply.

## Attester Compromise and Assurance {#assurance}

A compromised attester can impersonate instances within its approved
client associations. Receivers limit those associations and configure
acceptable evidence assurance; attesters MUST NOT claim stronger
assurance than their evidence supports, and self-reported,
platform-verified, and hardware-rooted evidence are not
interchangeable. Copied keys and enrollment data can be
indistinguishable from the original without independent platform
evidence; {{continuity}} governs detected forks, not guaranteed clone
detection. Instance identity does not prove software integrity beyond
the evaluated evidence.

## Forwarding and Privacy {#privacy}

* **Proof binding:** the Client Attestation has no audience; its
  proof-of-possession JWT identifies the Receiver, and combined DPoP
  binds the HTTP request. Forwarding resistance depends on ATTEST
  proof validation, freshness, and key possession, not identifier
  scope.
* **Correlation:** {{identifier-scope}} requires separate identifiers
  and keys across Receiver scopes, strengthening
  {{ATTEST, Section 11.1}}. Receivers MUST NOT assume identifiers
  across scopes are comparable; explicitly shared scopes permit
  correlation.
* **Token-binding keys:** scoping Instance Context to each consumer
  limits correlation through its identifier but not through the
  binding key. In DPoP combined mode ({{ATTEST, Section 5.2}}) the
  Client Instance Key is the DPoP key, so tokens for different
  consumers can carry different `id` values but the same `cnf.jkt`.
  Where unlinkability between Context Consumers is required, the
  client MUST use distinct token-binding keys across those scopes,
  for example DPoP without combined mode or a distinct mutual-TLS
  certificate per scope. Other claims and application data can
  correlate requests despite scoped identifiers. Identifier scoping does
  not permit changing a refresh token's bound key
  ({{grant-continuity}}).
* **Attester visibility:** supplying Receiver scope reveals it to the
  attester; deployments requiring ATTEST's audience-hiding property
  should omit this profile.
* **Disclosure:** status non-disclosure follows {{errors}}, and
  retention follows {{state}} and {{mapping-stability}}. Error
  responses SHOULD NOT reveal unrelated instance identities.
  Identifier unpredictability limits guessing and enumeration; it is
  not authentication.

# IANA Considerations

## JSON Web Token Claims

This document requests the following registrations in the "JSON Web
Token Claims" registry established by {{RFC7519}}. The Change Controller
for both entries is IETF.

| Claim Name | Claim Description | Specification Document(s) |
|---|---|---|
| `client_instance_id` | Issuer-scoped client instance identifier | {{claims}} of this document |
| `client_instance` | Validated client instance context | {{instance-context}} of this document |

## OAuth Token Introspection Response

This document requests the following registration in the "OAuth Token
Introspection Response" registry established by {{RFC7662}}:

* Parameter Name: `client_instance`
* Parameter Description: Validated client instance context
* Change Controller: IETF
* Specification Document(s): {{instance-context}} of this document

These registrations define claims and a response parameter, not subject
or actor profile values.

--- back

# Wire Examples {#wire-examples}
{:numbered="false"}

These examples are informative. The access-token and introspection
examples use the managed-device flow in {{managed-device-example}}: the
user is the authorization subject, and the installation is additional
context. The AS maps the attester's identifier to a value scoped to
`https://api.example`.

## Access Token Payload
{:numbered="false"}

Example decoded JWT access-token payload under {{RFC9068}}, with
`typ=at+jwt` in its protected header. The `cnf.jkt` value identifies
the public key in {{claims}}; the token's subject and context remain
separate.

~~~ json
{
  "iss": "https://as.example",
  "sub": "user-17",
  "aud": "https://api.example",
  "client_id": "https://platform.example/oauth-client",
  "iat": 1789128000,
  "exp": 1789128300,
  "jti": "at-95a76b823",
  "scope": "documents.read",
  "cnf": {
    "jkt": "Ak20Cf62SpTybasujYXbaI-Ms655MyvOZCtnnf8y1QU"
  },
  "client_instance": {
    "iss": "https://as.example",
    "id": "m-f61783ea4cb24d098851d34960a274be"
  }
}
~~~

## Introspection Response
{:numbered="false"}

For an opaque access token, an authenticated introspection response
conveys the same context. The resource has configured this endpoint as
authoritative for `https://as.example`. The response-level `iss` names
the token issuer; the nested `iss` names the Instance Context
Authority. They coincide in this mapped example.

~~~ http-message
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: no-store

{
  "active": true,
  "iss": "https://as.example",
  "sub": "user-17",
  "aud": "https://api.example",
  "client_id": "https://platform.example/oauth-client",
  "scope": "documents.read",
  "exp": 1789128300,
  "cnf": {
    "jkt": "Ak20Cf62SpTybasujYXbaI-Ms655MyvOZCtnnf8y1QU"
  },
  "client_instance": {
    "iss": "https://as.example",
    "id": "m-f61783ea4cb24d098851d34960a274be"
  }
}
~~~

## Governed Actor and Instance Context
{:numbered="false"}

An exchange profile can authorize instance B to continue work begun by
instance A for the same governed agent. In this example, that profile
selects the authenticated presenting instance for output context and
authorizes `agent-42` to act for `user-17` under {{ACTOR-PROFILE}}.
The AS validates B's attestation and proof, replaces A's input context
with B's resource-scoped mapping, and binds the output token under the
exchange profile. The relevant output claims are:

~~~ json
{
  "sub": "user-17",
  "act": {
    "iss": "https://as.example",
    "sub": "agent-42"
  },
  "client_instance": {
    "iss": "https://as.example",
    "id": "m-2b58e0d760954a5a9ce64f3e718d02ac"
  }
}
~~~

The governed actor remains `agent-42`; the mapped identifier identifies
B. Neither B's authentication nor continuity of the agent identity alone
authorizes this exchange. A profile preserving A's context instead would
have to define its upstream association explicitly.

## Attestation Rejection
{:numbered="false"}

Example token-endpoint response when a required instance claim is
missing or instance policy rejects the request ({{errors}}). It does
not disclose whether the instance is unknown, suspended, or retired.

~~~ http-message
HTTP/1.1 400 Bad Request
Content-Type: application/json
Cache-Control: no-store

{
  "error": "invalid_client_attestation"
}
~~~

# Deployment Examples {#deployment-examples}
{:numbered="false"}

These informative sketches share three steps. `C1` is the Logical Client;
`I1` and `K1` are an instance identifier and key scoped to the AS.

1. The attester verifies enrollment evidence and possession of `K1`, then
   issues the attestation in {{claims}}: `sub=C1`,
   `client_instance_id=I1`, and `cnf.jwk` containing public key `K1`.
2. The client presents its authorized grant, attestation, and combined
   DPoP proof using `K1` to the AS's token endpoint.
3. The AS validates them and issues a DPoP-bound token with mapped
   context `M1`, scoped to the resource. The resource validates the token,
   proof, and context without receiving the enrollment evidence.

## CIMD Client {#cimd-example}
{:numbered="false"}

Let `C1` be `https://platform.example/oauth-client`. Its CIMD declares
`attest_jwt_client_auth_dpop`; {{ATTESTER-ENDORSEMENT}} provides a
metadata example. The AS validates the CIMD and uses configured attester
trust or an accepted `client_attesters` endorsement. Instance-profile
selection remains separately configured. Installations share `C1`, with
distinct instance identifiers and keys; key renewal does not change the
CIMD. User-authorized access follows {{managed-device-example}}.

## AAuth Agent Provider {#aauth-example}
{:numbered="false"}

The Agent Provider verifies managed installation evidence at step 1 and
issues a separate Client Attestation. Native `aa-agent+jwt` credentials
and HTTP Message Signatures do not replace the OAuth attestation or
proof. Step 2 uses a pre-authorized client credentials grant; AAuth
metadata alone does not establish attester trust.

## SPIFFE Workload {#spiffe-example}
{:numbered="false"}

At step 1 the workload uses an X.509-SVID from the Workload API for
mutual TLS. The attester validates its trust domain and runtime evidence
binding this container to `K1`; a shared SPIFFE ID cannot distinguish
replicas. Step 2 uses client credentials with the resulting attestation.
At container granularity, restart requires new enrollment; SVID renewal
within a continuing container does not.

## Managed Device {#managed-device-example}
{:numbered="false"}

At step 1 a management component verifies the installed harness and its
platform-protected `K1`, using app-attestation evidence where available.
Device enrollment or key storage alone does not identify the installation.
Before step 2, the harness obtains a code through an external browser
{{RFC8252}} with PKCE `S256` {{RFC7636}}, then redeems it directly with
the code verifier, redirect URI, `C1`, attestation, and DPoP proof. The
browser receives neither attestation nor proof. Process restarts can
retain the installation identity; reinstall requires new enrollment.

# Document History {#history}
{:numbered="false"}

*RFC EDITOR: Remove this section before publication.*

* Initial draft.
