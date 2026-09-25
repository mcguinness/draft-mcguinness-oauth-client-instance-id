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
  RFC9421:
--- abstract

This specification defines an optional claims profile of OAuth 2.0
Attestation-Based Client Authentication. When selected, the profile
requires an attester-qualified client instance identifier that remains
stable across verified key changes, and adds continuity and privacy
rules for that identifier. Conveying instance context in tokens and
introspection responses remains optional within the profile.
Authentication and proof methods follow the base specification.

--- middle

# Introduction

Attestation-Based Client Authentication {{ATTEST}} answers one
question: is this an authorized Client Instance in possession of this
key? This profile adds a second: is this the same Client Instance the
Receiver previously encountered? A new attestation alone cannot answer
it: {{Section 10.6 of ATTEST}} requires one for every new key, and after
a key change a continuing installation is indistinguishable from a new
one.

The gap opens wherever one Logical Client has many running copies: an
application on each managed laptop, a container per replica, an agent
runtime per host. Every copy presents a valid attestation for the same
`client_id`, distinguishable only by its current key. Once a copy
rotates that key, a Receiver cannot tell whether it is the copy it
suspended or a new one, and audit history and status decisions tied to
the old key do not carry over.

The profile adds two claims:

* `client_instance_id` lets a Receiver follow each copy across key
  changes. It names one installation or runtime in its Client
  Attestation. The attester assigns it, retains it across verified key
  changes, and by default scopes it to one Receiver, which reveals that
  Receiver to the attester.
* `client_instance` lets a resource server that never sees the
  attestation correlate requests with the instance validated at
  issuance. It carries a mapped reference to that instance in a token
  or introspection response.

This profile establishes instance identity and its continuity, not
authority: instance evidence grants none. Authorization profiles can
use validated instance identity or Instance Context as a policy input,
subject to the prohibitions in {{processing}}.

ATTEST alone suffices when correlation need only last for the current
key or stay within one system.

Assigning each instance its own `client_id` with a shared `software_id`
({{Section 2 of RFC7591}}) is an alternative. This profile instead
targets deployments that share one Logical Client, one metadata URL
when using {{CIMD}}, and authorization server policy keyed by that
client. `software_id` correlates registrations but defines no
shared grants or policy across separate client identities.

{{conformance}} lists the five implementing roles and their
requirements.

## Identity and Scope

| Identity | Purpose |
|---|---|
| Logical Client (`client_id`) | Identifies the OAuth client |
| Authorization principal | Identifies the subject or delegated actor |
| Client Instance | Identifies one particular installation or running copy of the client software |

The deployment chooses the instance granularity:

* **Installation:** one installation, retaining its identity across
  process restarts.
* **Execution:** one unit the deployment names, such as a process,
  container, or Kubernetes Pod, retaining its identity for that
  unit's lifetime.

Enrollment records that choice.

This profile is for administratively configured deployments such as
workloads and managed desktop or mobile applications. It is not a
general-purpose device or wallet identifier and defines no enrollment,
key-rotation, or status-distribution protocol. ATTEST supplies the
authentication and proof methods, including direct resource-server
presentation ({{Section 7 of ATTEST}}). When selected under
{{configuration}}, the profile applies whether the Client Attestation
is the client authentication method or an additional security signal
({{Section 7.6 of ATTEST}}). It does not replace the deployment's
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
  Client Instance at the configured granularity and Receiver Scope
  ({{receiver-scope}}).

Source Instance Identity:
: The pair `(iss, client_instance_id)` from a validated Client
  Attestation: the Attester Issuer and the Instance Identifier. This
  profile keeps it stable across verified key changes, and Instance
  Context is mapped from it.

Instance Context:
: The `client_instance` object in a token or introspection response.
  It grants no authority and does not prove current possession.

Receiver:
: A party that validates a Client Attestation under this profile, such
  as an authorization server or resource server. A Receiver that issues
  tokens carrying Instance Context is also a token issuer.

Context Consumer:
: A party that consumes Instance Context from a token or introspection
  response. It need not receive the Client Attestation.

Receiver Scope:
: The Receiver, or explicitly configured set of Receivers, to which one
  assignment of `client_instance_id` is scoped ({{receiver-scope}}).

Consumer Scope:
: The Context Consumer, or explicitly configured set of Context
  Consumers, to which one mapping of Instance Context is scoped
  ({{format-and-mapping}}).

Instance Context Authority:
: The token issuer identified by `iss` in Instance Context. It assigned
  the current `id` and is the enclosing token issuer unless the context
  was preserved from an upstream token. The Client Attester remains the
  authority for the Source Instance Identity.

Instance Context Identifier:
: The pair `(iss, id)` in Instance Context. Like a pairwise subject
  identifier, it is the Instance Context Authority's own correlator for
  one Source Instance Identity within a Consumer Scope; its `id` need
  not equal `client_instance_id`.

enrollment:
: An attester-maintained record binding one instance, at the configured
  granularity, to its verified keys and assigned identifiers, separate
  from any user account, device registration, or Logical Client.

Identifier values in this profile are opaque. Implementations MUST
compare `iss`, `client_instance_id`, `client_instance.iss`, and
`client_instance.id` as exact, case-sensitive strings without URI
normalization, and MUST NOT derive permissions, granularity, or key
locations by parsing them.

# Profile Selection and Trust {#configuration}

The client and Receiver administratively configure:

* the Logical Client, authentication method, and attester trust policy;
* the instance granularity, continuity evidence, and freshness limits;
* the intended Receiver and any explicitly shared Receiver Scope
  ({{receiver-scope}}).

A Context Consumer that requires context configures that requirement,
including whether it extends to attribution, with the issuers it accepts
context from. Claims MUST NOT select this profile or change
authentication methods; selection is part of the client-specific trust
agreement, and this profile adds no discovery or metadata parameter.
The error in {{errors}} reports rejection, not profile discovery.

## Attester Trust {#attester-trust}

The Receiver MUST bind each approved Attester Issuer to its validation
keys and authorized Logical Clients through configured associations. An
authorization server can derive those associations from client
endorsements it accepts, for example under {{ATTESTER-ENDORSEMENT}}.
That document governs authorization server endpoints only; a resource
server validating attestations directly uses configured associations. A
credential's `iss`, proof of possession, or client-published metadata
({{RFC7591}}, {{CIMD}}) alone does not establish attester authority. Key
resolution follows {{Section 10.8 of ATTEST}}. Local trust withdrawal
MUST take effect on subsequent authentication.

## Conformance {#conformance}

Conformance is role-specific:

| Role | Implements | Where |
|---|---|---|
| Client Attester | Claim, identifier, continuity, and enrollment requirements | {{claims}}, {{attester-requirements}} |
| Client | Scoped attestation use, key separation, and the selected ATTEST proof method | {{receiver-scope}}, {{presenter-attribution}}, {{token-binding-keys}} |
| Receiver | Trust, validation, grant-continuity, and revocation rules | {{configuration}}, {{processing}}, {{suspension}} |
| Token issuer conveying context | Mapping, preservation, and any binding that attribution requires | {{instance-context}} |
| Context Consumer | Context validation and applicable proof checks | {{presenter-attribution}}, {{context-consumer}}, {{context-errors}} |

An implementation serving several roles satisfies each. Conveying
Instance Context ({{instance-context}}) is optional and independent of
the rest of the profile.

# Client Attestation Claims {#claims}

This profile uses the additional claims permitted by
{{Section 4 of ATTEST}}; all ATTEST requirements apply, including
`typ=oauth-client-attestation+jwt`, `sub=client_id`, required `exp`
and `cnf`, and optional `iat`.

`iss`:
: REQUIRED. Exactly matches an approved Attester Issuer
  ({{attester-trust}}).

`client_instance_id`:
: REQUIRED. Nonempty JSON string whose UTF-8 encoding is at most 256
  octets after JSON string decoding, identifying the instance within
  the attester's namespace. A URI form carries no URI semantics.
  Receivers MUST reject longer values. Assignment, scoping, and
  generation follow {{identifier-generation}}.

## Receiver Scope {#receiver-scope}

A Receiver Scope is an enrollment or issuance input, not an OAuth
parameter or attestation audience. A Receiver cannot verify it from the
identifier alone.

The attester MUST assign distinct identifiers per Receiver unless an
administrative agreement explicitly authorizes a shared identifier
within a named set of Receivers. A shared client or trust domain does
not by itself authorize sharing. The client MUST request and use the
attestation for that configured scope. A Receiver cannot detect an
attestation presented outside its scope; the resulting correlation
exposes the end user of that instance ({{Section 11.1 of ATTEST}}).

Scoping to a Receiver means the attester learns that Receiver. That
gives up a property {{ATTEST}} states in its abstract: the client proves
its authenticity without revealing its target audience to the
attester. This profile trades attester visibility for unlinkability
between Receivers. A deployment that needs the attester not to learn
individual Receivers, or that intends correlation across them as an
enterprise workload might, configures one Receiver Scope spanning
them. The attester then learns only that scope, those Receivers can
correlate the instance, and one identifier and key suffice.

The client MUST also use distinct Client Instance Keys across scopes.
{{Section 11.1 of ATTEST}} recommends this across authorization and
resource servers; this profile requires it across the scopes a
deployment chooses to separate, because a shared key links
attestations regardless of their identifiers. Token-binding key
separation between Context Consumers is addressed in {{token-binding-keys}}.

In combined mode, the Client Instance Key is also the DPoP key. In
normal mode, the DPoP key is independent of the attestation
({{Section 5.2 of ATTEST}}). A token issued in combined mode is bound to
the authorization server's scoped key. A separately scoped resource
server's attestation carries a different key. No single proof can
match both, so combined mode cannot also be used at that resource
server. Instead, a client authenticating in normal mode can present
the resource-server-scoped key as its DPoP key at issuance. That
allows combined mode at the resource server. The authorization server
then also sees that key. The two Receivers can therefore correlate
through its thumbprint, even though their identifiers and Client
Instance Keys differ.

## Example {#attestation-example}

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
   {{configuration}}, including that the key that verified the
   attestation is bound to the asserted `iss`. Key resolution
   under {{Section 10.8 of ATTEST}} selects a key from JOSE header
   parameters, not from `iss`, so a trust anchor covering several
   attesters does not by itself establish that association.
3. Associate the Source Instance Identity with the Logical Client and
   validated Client Instance Key, then apply the Receiver's local
   policy for that instance, such as suspension or required prior
   enrollment, reporting rejection under {{errors}}.

The Receiver MUST NOT:

* substitute an identifier for proof of possession or infer it from a
  key thumbprint, certificate serial number, or JWT `jti`;
* infer continuity from equal identifiers or keys across Attester
  Issuers; or
* set an access token's `sub`, add `act`, or extend an actor chain
  solely from instance evidence.

Migration between Attester Issuers requires a procedure that
establishes trust in both authorities and continuity evidence; this
profile defines none. Key selection follows ATTEST; when a separate
token-binding key is used, context identifies the instance associated
with the Client Instance Key.

## Grant Continuity {#grant-continuity}

{{Section 10.3 of ATTEST}} binds a refresh token to the Client Instance,
by default through the Client Instance Key. The refresh token stays
bound to that key; only a profile acting under {{Section 13 of ATTEST}}
can rebind it. This profile records an identity for the binding that
survives a verified key change, so later grants correlate with the same
instance. It adds no instance-bound grants. Under the default binding, a
refresh after a key change fails on the key alone. Under that binding,
the identity check below instead catches an attestation for the bound
key that names a different instance; it governs refresh across key
changes only under a profile that rebinds refresh tokens.

For a grant established using a Client Attestation validated under this
profile, the authorization server MUST record the Source Instance
Identity when issuing a refresh token. On refresh of such a grant, the
authorization server MUST require a validated attestation whether it is
the client authentication method or an additional security signal. This
extends {{Section 10.3 of ATTEST}}, which requires the attestation
mechanism when refreshing. The authorization server MUST also enforce
two independent invariants:

* the Source Instance Identity in the current validated attestation
  MUST match the recorded one; and
* the proof and key binding MUST satisfy ATTEST, or a profile that has
  redefined refresh-token binding under {{Section 13 of ATTEST}}.

Instance continuity and key-binding continuity are independent. A
conflict with the recorded identity MUST produce `invalid_grant`
{{RFC6749}} without disclosing the expected identity.

If authorization-time policy bound a code or other artifact to an
instance, the authorization server MUST enforce that binding at
redemption, whether the attestation is the client authentication method
or an additional security signal ({{Section 7.6 of ATTEST}}). Presenting
the attestation is optional in that second mode, so an authorization
server that binds artifacts to instances MUST require it at their
redemption; otherwise a conforming client cannot supply what the
authorization server must check. {{Section 10.4 of ATTEST}} recommends
establishing such bindings where attestation is the client
authentication method.

## Attestation Errors {#errors}

Missing or invalid required instance claims and rejection by the
Receiver's local policy for the instance ({{processing}}) MUST produce
`invalid_client_attestation`. {{Section 7.4 of ATTEST}} defines that
code for failures to verify the attestation or its proof. This profile
deliberately extends it to instance claims and policy, so responses do
not disclose whether an instance is unknown, suspended, or retired.
Receivers SHOULD also avoid distinguishable response timing. The cost is
that a client cannot tell a transient failure from a durable policy
decision, or whether retrying with a fresh attestation will help.
Responses use the format ATTEST specifies for the code
({{Section 5.2 of RFC6749}} at the authorization server,
{{Section 3 of RFC6750}} at a resource server). Unknown instances are
rejected only when local policy requires prior enrollment. A failed
profile check MUST NOT fall back to authentication without the required
evidence. Grant-binding errors follow {{grant-continuity}}; other errors
follow ATTEST.

# Attester Requirements {#attester-requirements}

An Instance Identifier's meaning over time depends on the attester:
how it generates identifiers ({{identifier-generation}}), when it may
retain them ({{continuity}}), how it handles suspension
({{suspension}}), and what records it keeps ({{state}}).

## Identifier Generation {#identifier-generation}

For each enrollment and Receiver Scope ({{receiver-scope}}), the
attester MUST assign identifiers that are:

* distinct, opaque, and never reassigned, including after retirement;
* generated from at least 128 bits of either cryptographically secure
  randomness or keyed pseudorandom-function output;
* unpredictable to parties other than the attester; and
* free of runtime hostnames, user identifiers, and embedded instance
  attributes.

A URI can name the attester's namespace; its instance-specific portion
remains opaque. Keyed derivation, such as HMAC {{RFC2104}}, MUST use a
secret with at least 128 bits of entropy and unambiguously encode the
Logical Client, Receiver Scope, and a unique enrollment component; a
platform-stable input alone would reproduce retired identifiers after
reinstall. If the attester changes its derivation inputs or secrets,
it MUST still produce the values already assigned within a continuing
enrollment. Storing those values suffices.

## Continuity and Lifecycle {#continuity}

Continuity is authenticated evidence that shows the attester that a
claimant is the same enrolled Client Instance at the configured
granularity. An attester-recorded chain of verified key custody within
one enrollment is the primary mechanism; platform or hardware-rooted
identity evidence can supplement that chain or, where the deployment's
evidence policy permits, replace it. Before retaining an identifier,
the attester MUST verify and record:

1. an active enrollment binding the instance, Logical Client, Receiver
   Scope, granularity, and previously verified keys;
2. fresh possession of the current key and authenticated evidence
   binding it to that enrollment, including an authorized custody
   transition when the key changes; and
3. the configured continuity checks, their freshness, and observed
   lifecycle events or evidence of independent claimants.

The deployment specifies its evidence, freshness limits, and lifecycle
boundaries. For example, when the unit is a Kubernetes Pod, a container
restart keeps the identity, but a new Pod is a new unit. An identifier
at Installation or Execution granularity does not identify individual
processes within that installation or unit. The attester MUST apply the
following outcomes:

| Event | Required outcome |
|---|---|
| Renewal, verified key change, process restart at Installation granularity, or in-place update | Retain identifiers when continuity is verified |
| Reinstall, independent clone, replacement or restart of the unit at Execution granularity, or granularity change | Require new enrollment |
| Restore or snapshot rollback | Retain only with fresh evidence that the claimant succeeds the prior holder; copied keys and data alone are insufficient |
| Resume after suspension ({{suspension}}) | Apply continuity checks at the next issuance using available authenticated evidence |
| Continuity cannot be established | Require new enrollment; a continuing original can retain its own enrollment |
| Detected fork of one enrollment | Retire the identifiers and enroll claimants separately, unless authenticated evidence shows which claimant continues the enrollment; that claimant keeps them |

The attester MUST NOT knowingly retain identifiers for independent
instances. Concurrent processes within one installation are not by
themselves a fork. This prohibition covers detected forks, not events
the platform cannot observe ({{assurance}}). An identifier, expired
attestation, or former public key alone does not establish continuity.

## Suspension and Status {#suspension}

The attester MUST stop issuance for suspended or retired enrollments.
A Receiver suspending an instance SHOULD revoke its grants or report
their tokens inactive through introspection {{RFC7662}}. Short
attestation lifetimes narrow the window in which a suspended instance
remains acceptable.

When an authorization server revokes a grant for instance suspension,
retirement, or attester trust withdrawal, it MUST invalidate all of that
grant's access and refresh tokens and prevent further refresh issuance.

Without a status channel, parties unaware of the change face three
gaps: existing attestations may remain acceptable until expiration
plus clock skew; issued tokens may still be accepted for their own
lifetimes; and local revocation does not notify resource servers
validating tokens offline. Security Event Tokens {{RFC8417}} delivered
under {{RFC8935}} can support a status integration outside this
profile.

## State and Retention {#state}

Attesters MUST retain enrollment and verification records while
issuing or renewing attestations, and status while supporting
resumption. Verification summaries suffice. Deleting continuity
records requires new enrollment. A validating Receiver need not keep
an instance allowlist, but local suspension needs instance status,
revocation needs token associations, and mapped context needs its
mappings ({{mapping-stability}}). Audit retention is local policy.

# Conveying Instance Context {#instance-context}

This section is optional. It defines how a token issuer represents a
validated instance to downstream consumers ({{format-and-mapping}})
and what a consumer may conclude from it ({{context-consumer}}).

## Format and Mapping {#format-and-mapping}

An issuer MAY include `client_instance` in a token or introspection
response {{RFC7662}}. The `client_instance` value is a JSON object
whose two REQUIRED members together form an Instance Context
Identifier:

| Member | Type | Meaning |
|---|---|---|
| `iss` | Nonempty string | Instance Context Authority that assigned `id` |
| `id` | Nonempty string, at most 256 octets of UTF-8 after JSON string decoding | That authority's representation of the instance |

For direct issuance from a validated Client Attestation, context MUST
identify the authenticated presenting instance; on refresh, that
identity is subject to {{grant-continuity}}. Token exchange follows
{{context-exchange}} instead.

Context MUST refer to validated instance participation; issuers MUST
NOT copy unvalidated client-supplied context. The issuer MUST map its
mapping input (the Source Instance Identity or, when remapping under
{{context-exchange}}, the upstream Instance Context Identifier) into
its own namespace. Each mapping MUST:

* keep distinct instances separate unless continuity is established;
* generate `id` under {{identifier-generation}}, within the same
  length bound as `client_instance_id` ({{claims}}), and retain it
  across attestation renewal and verified key changes; and
* scope `id` to a Consumer Scope, allowing sharing only within an
  explicitly configured set.

Each Consumer Scope is a configured set of audience values. The issuer
selects the mapping by the token's audience: its `aud` claim or, for an
opaque token, the audience recorded at issuance. If a token has no
audience, or its audiences span Consumer Scopes, no single `id` is
correct and the issuer MUST omit `client_instance`. An introspection
response conveys the `id` the token would carry. The issuer MUST omit
`client_instance` when the authenticated caller is outside the token's
Consumer Scope.

For derived mappings, the mapping input supplies the enrollment-specific
component and the consumer supplies the scope. Issuers MUST separate
this derivation from attester identifiers, for example by a distinct
key or purpose label. {{wire-examples}} shows mapped context in an
access token and an introspection response.

## Mapping Stability {#mapping-stability}

For one mapping input and Consumer Scope, an issuer:

* MUST NOT represent that input by more than one `id`;
* MUST retain the mapping while any token or grant it issued for that
  input remains valid, including clock skew; and
* MUST omit `client_instance` rather than assign a replacement once it
  no longer holds or can reproduce the mapping. A Context Consumer
  requiring context then rejects under {{context-errors}}.

An issuer that changes its derivation inputs or secrets MUST still
produce the identifiers already assigned for any input and scope for
which it continues to include context, as attesters must under
{{identifier-generation}}. Storing those values suffices. Otherwise,
rotating one secret would silently and permanently strip context from
every instance mapped under it.

Attester identifiers are never reassigned, so a new enrollment
presents a new mapping input and gets a new mapping.

Storage needs differ by mapping strategy:

| Strategy | Records needed | Non-reassignment rests on |
|---|---|---|
| Derived | None per instance, while the derivation secret and inputs remain available ({{identifier-generation}}) | Never reusing enrollment inputs |
| Random | Retained for as long as the issuer intends to include context | Probability |

## Presenter Attribution {#presenter-attribution}

Instance Context grants no authority. Alone, it describes only the
instance that participated in obtaining the token.

When a Context Consumer uses context to attribute the current token
presentation to that instance, the applicable consuming profile MUST
require a mechanism associating the token presenter with the instance. A
Context Consumer attributing a presentation this way MUST validate that
mechanism. For tokens issued directly from a validated Client
Attestation, the mechanism is sender constraint: DPoP {{RFC9449}},
mutual TLS {{RFC8705}}, or another the consuming profile defines, using
a key the issuer associated with the authenticated instance at issuance.

For presenter attribution, the client MUST use a constraining key
unique to the instance at the configured granularity. A key shared by
instances inside that boundary
establishes no attribution, because any of them can present the
token and satisfy the proof.

Where the configured requirement for a Consumer Scope includes
attribution, an issuer that cannot bind such a key MUST omit
`client_instance`. Context conveyed without such a key records only
participation ({{context-consumer}}), including when it is conveyed
only through introspection.

A token without sender constraint supports no presenter attribution,
and a Context Consumer requiring attribution rejects it
({{context-errors}}). The HTTP `Bearer` scheme does
not indicate an unbound token; certificate-bound tokens {{RFC8705}}
also use it.

All validation requirements of the token-binding mechanism apply
whether or not context is used for attribution, including rejection of
a bound token presented without its proof ({{Section 7.2 of RFC9449}}).
The binding authenticates the presenter; it does not establish that the
presenter is the instance named in context derived from an upstream
token. Any
presenter or key change during exchange requires authorization under
the consuming profile. Unlinkability between Context Consumers also
requires distinct token-binding keys ({{token-binding-keys}}).

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
and the upstream Consumer Scope authorize its disclosure downstream;
otherwise it MUST remap or omit the context.

An issuer MUST NOT name an upstream Instance Context Authority in
`iss` unless it has authenticated:

1. that the named authority assigned the context; and
2. that the context refers to the instance the input token represents.

Validating the input token authenticates only its issuer's assertions.
By default, therefore, an issuer MUST limit preservation to context
whose `iss` equals the authenticated input-token issuer, and MUST remap
or omit context that an intermediary itself preserved. A consuming
profile that specifies authenticated provenance and its enforcement can
permit deeper preservation; the object carries no forwarding history,
and shallow preservation is also a privacy default.

Context identifies one instance, not a chain, and MUST NOT be treated
as a separate token or delegated actor. Remapping changes the Instance
Context Identifier; like preservation, it leaves the represented
Source Instance Identity unchanged. An issuer MUST NOT treat upstream
context as identifying a different presenting instance. The upstream
identifier does not establish that the issuer validated the original
Client Attestation.

## Context Consumer Processing {#context-consumer}

Before using context, the Context Consumer MUST:

1. Validate the enclosing token or authenticated introspection
   response.
2. Validate the context members, treating `id` values over the length
   bound in {{format-and-mapping}} as invalid and ignoring unrecognized
   members.
3. Accept the Instance Context Authority only when it is the token
   issuer or an upstream token issuer explicitly trusted for that
   issuer and consumer.
4. Discard invalid context and, when context is required, reject the
   request if context is missing or was discarded ({{context-errors}}).

Before using the context's association with the subject, actor, or
presenter in policy or audit, a Context Consumer MUST establish that
association from the applicable consuming profile and the validated
token or trusted introspection configuration.

Where the issuer may have preserved context from an input token, the
association is established only if that profile also defines how the
context's provenance is authenticated, because the object does not
record how the issuer obtained it. The `client_instance` object alone,
including whether its `iss` matches the token issuer, establishes
neither the association nor the provenance.

When trusted configuration establishes that the token issuer conveys
context only from direct Client Attestation validation, this document
is the consuming profile. Context then identifies the authenticated
presenting instance ({{format-and-mapping}}), and validating the
token's sender constraint ({{presenter-attribution}}) establishes the
association. That configuration describes the issuer, not the token,
so it does not apply at an issuer that also preserves context from
input tokens. There, only a consuming profile carrying provenance can
distinguish directly validated context from preserved context.

If the association is not established, the Context Consumer MUST treat
context only as evidence of instance participation and MUST NOT
attribute the current request to that instance. A Context Consumer
whose configured requirement includes attribution MUST reject under
{{context-errors}}.

An issuer whose trust agreement with a Context Consumer states that it
conveys context only from direct Client Attestation validation MUST NOT
convey to that consumer Instance Context derived from anything other
than a Client Attestation it validated for the issuing request.

For introspection, trusted endpoint configuration identifies the
expected token issuer. A response-level `iss`, if present, MUST match
it; neither the endpoint URL nor `client_instance.iss` selects the
issuer. Multi-issuer introspection requires a consuming profile that
authenticates the represented issuer. The `iss` value does not
authorize fetching keys from that location, and extensions MUST NOT
change the meaning of `iss` or `id`.

## Context Errors {#context-errors}

Rejection for missing or invalid required context, or for required
attribution that cannot be established, MUST use `invalid_token` at a
resource server ({{Section 3.1 of RFC6750}}) or `invalid_request` for a
rejected subject or actor token in an exchange
({{Section 2.2.2 of RFC8693}}).

Retrying with a new access token ({{Section 3.1 of RFC6750}}) does not
help a client rejected for a missing sender constraint. A resource
server rejecting for that reason SHOULD include the challenge for the
binding mechanism it requires, such as the `DPoP` scheme in
{{Section 7.1 of RFC9449}}, so the client learns what to change.

Other consuming profiles define their own error mapping. Direct Client
Attestation failures follow {{errors}}.

# Relationship to Other Identity Systems

This section compares the profile with mechanisms that identify
clients or workloads but not individual instances; it adds no
requirements.

## Client ID Metadata Documents

With {{CIMD}}, the metadata URL is the Logical Client's `client_id`,
and many installations can use it. Fetching public metadata does not
prove a caller is an authorized instance:

| Mechanism | Contribution |
|---|---|
| CIMD | Discovers Logical Client metadata and its authentication method |
| ATTEST | Authenticates an attester-approved instance and possession of its key |
| This profile | Retains instance identity across verified key changes and conveys optional downstream context |

One CIMD URL serves the Logical Client, and Instance Identifiers belong
in attestations, not per-installation metadata. The attestation's `sub`
is that URL, and `client_instance_id` distinguishes installations
({{cimd-example}}). CIMD removes per-authorization-server registration
of client metadata but not this profile's trust agreement
({{configuration}}). {{ATTESTER-ENDORSEMENT}} lets a client endorse
attesters through `client_attesters` metadata, subject to authorization
server policy. That endorsement establishes the attester-to-client
association but does not select this profile or its continuity and
privacy policy.

## Workload and Agent Credentials

A shared workload identity does not identify an individual instance; the
attester needs instance-level evidence under {{continuity}}. Direct
SPIFFE Verifiable Identity Document (SVID) authentication and client
mappings follow {{SPIFFE-OAUTH}}. An AAuth Agent Provider {{AAUTH}} or
workload attester can instead issue a Client Attestation under this
profile when it holds the required enrollment evidence. Native
credentials with different `sub` or `typ` semantics require a separate
carrier profile. A deployment authenticating only with them conveys no
Instance Context under this profile, which maps context only from a
validated Client Attestation ({{format-and-mapping}}).
{{deployment-examples}} illustrates these boundaries.

# Security Considerations

The security considerations of {{ATTEST}} and {{RFC8725}} apply.

## Attester Compromise and Assurance {#assurance}

A compromised attester can impersonate instances within its approved
client associations. Receivers limit those associations and configure
acceptable evidence assurance. An attester that overstates its
evidence's assurance defeats that configuration. Self-reported,
platform-verified, and hardware-rooted evidence are not
interchangeable. Without independent platform evidence, copied keys and
enrollment data can be indistinguishable from the original;
{{continuity}} governs detected forks and does not guarantee clone
detection. Instance identity proves no software integrity beyond the
evaluated evidence.

## Forwarding {#forwarding}

Forwarding resistance comes from ATTEST proof validation, freshness,
and key possession, not from identifier scope. The Client Attestation
has no audience; its proof-of-possession JWT identifies the Receiver,
and combined DPoP binds the HTTP request.

# Privacy Considerations {#privacy}

The privacy considerations of {{Section 11 of ATTEST}} apply.

## Correlation Across Receivers {#correlation}

Separate identifiers and keys per Receiver Scope ({{receiver-scope}})
limit correlation, strengthening {{Section 11.1 of ATTEST}}. Receivers
MUST NOT assume identifiers across scopes are comparable; explicitly
shared scopes permit correlation.

## Token-Binding Keys {#token-binding-keys}

Per-consumer Instance Context Identifiers do not prevent correlation
through a shared binding key. In DPoP combined mode
({{Section 5.2 of ATTEST}}) the Client Instance Key is the DPoP key, so
tokens for different consumers can carry different `id` values but the
same `cnf.jkt`. Where unlinkability between Context Consumers is
required, the client MUST use distinct token-binding keys across those
scopes, for example DPoP without combined mode or a distinct mutual-TLS
certificate per scope. Other claims and application data can also
correlate requests. Identifier scoping does not permit changing a
refresh token's bound key ({{grant-continuity}}).

## Attester Visibility {#attester-visibility}

The attester learns any Receiver Scope the client supplies.
{{receiver-scope}} describes the trade-off and how a single Receiver
Scope spanning every Receiver the client uses limits it.

## Error Disclosure {#disclosure}

Error responses SHOULD NOT reveal unrelated instance identities.
Status non-disclosure follows {{errors}}; retention follows {{state}}
and {{mapping-stability}}. Unpredictable identifiers limit guessing and
enumeration but are not authentication.

# IANA Considerations

## JSON Web Token Claims

This document requests the following registrations in the "JSON Web
Token Claims" registry established by {{RFC7519}}.

| Claim Name | Claim Description | Change Controller | Specification Document(s) |
|---|---|---|---|
| `client_instance_id` | Attester-qualified client instance identifier | IETF | {{claims}} of this document |
| `client_instance` | Validated client instance context | IETF | {{instance-context}} of this document |

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

These examples are informative. The first two use the managed-device
flow ({{managed-device-example}}): the user is the authorization
subject, and the installation appears as Instance Context. The
authorization server maps the attester's identifier to a value scoped to
`https://api.example`.

## Access Token Payload
{:numbered="false"}

A decoded {{RFC9068}} access-token payload; its protected header has
`typ=at+jwt`. `cnf.jkt` identifies the public key in
{{attestation-example}}.

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
the token issuer, and the nested `iss` names the Instance Context
Authority; here they are the same authorization server.

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

A consuming profile can authorize instance `B` to continue work that
instance `A` began for the same governed agent. Here that profile
authorizes `agent-42` to act for `user-17` under {{ACTOR-PROFILE}} and
selects the authenticated presenting instance for output context. The
authorization server validates `B`'s attestation and proof, replaces
`A`'s input context with `B`'s resource-scoped mapping, and binds the
output token under the consuming profile. The relevant output claims
are:

~~~ json
{
  "sub": "user-17",
  "aud": "https://api.example",
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

The governed actor remains `agent-42`; the mapped identifier is `B`'s.
Neither `B`'s authentication nor continuity of the agent identity
alone authorizes this exchange. A profile that instead preserved `A`'s
context would have to define that upstream association explicitly.

## Attestation Rejection
{:numbered="false"}

A token-endpoint response when a required instance claim is missing
or local policy rejects the instance ({{errors}}). It does not reveal
whether the instance is unknown, suspended, or retired.

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

These informative sketches share three steps. `C1` is the Logical
Client; `I1` and `K1` are an Instance Identifier and key scoped to the
authorization server.

1. The attester verifies enrollment evidence and possession of `K1`,
   then issues the attestation in {{claims}}: `sub=C1`,
   `client_instance_id=I1`, and `cnf.jwk` holding public key `K1`.
2. The client presents its grant, attestation, and combined DPoP proof
   using `K1` at the authorization server token endpoint.
3. The authorization server validates them and issues a DPoP-bound token
   with mapped context `M1`, scoped to the resource. The resource
   validates the token, proof, and context without seeing the enrollment
   evidence.

## CIMD Client {#cimd-example}
{:numbered="false"}

Let `C1` be `https://platform.example/oauth-client`, whose CIMD declares
`attest_jwt_client_auth_dpop` ({{ATTESTER-ENDORSEMENT}} has a metadata
example). The authorization server validates the CIMD and trusts the
attester through configuration or an accepted `client_attesters`
endorsement. Key renewal does not change the CIMD. User-authorized
access follows {{managed-device-example}}.

## AAuth Agent Provider {#aauth-example}
{:numbered="false"}

At step 1 the Agent Provider verifies managed installation evidence and
issues a Client Attestation. Native `aa-agent+jwt` credentials and HTTP
Message Signatures {{RFC9421}} do not replace it or the proof. Step 2
uses a pre-authorized client credentials grant; AAuth metadata alone
does not establish attester trust.

## SPIFFE Workload {#spiffe-example}
{:numbered="false"}

At step 1 the workload uses an X.509-SVID from the Workload API for
mutual TLS. The attester validates its trust domain and runtime evidence
binding this container to `K1`; a shared SPIFFE ID cannot distinguish
replicas. Step 2 uses client credentials with the resulting attestation.
With the container as the Execution unit, restart requires new
enrollment; SVID renewal within a continuing container does not.

## Managed Device {#managed-device-example}
{:numbered="false"}

At step 1 a management component verifies the installed client and its
platform-protected `K1`, using app-attestation evidence where available.
Device enrollment or key storage alone does not identify the
installation.
Before step 2, the client obtains a code through an external browser
{{RFC8252}} with PKCE `S256` {{RFC7636}}, then redeems it with
the code verifier, redirect URI, `C1`, attestation, and DPoP proof. The
browser receives neither attestation nor proof. Process restarts can
retain the installation identity; reinstall requires new enrollment.

# Document History {#history}
{:numbered="false"}

*RFC EDITOR: Remove this section before publication.*

* Initial draft.
