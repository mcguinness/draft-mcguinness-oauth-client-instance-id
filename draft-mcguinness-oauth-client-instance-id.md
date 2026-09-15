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
Attestation-Based Client Authentication. It adds an issuer-qualified
client instance identifier, continuity and privacy rules, and optional
instance context in tokens and introspection responses. The profile
supports correlation across attestations and verified key changes.
Authentication and proof methods follow the base specification; access
tokens carrying instance context are sender-constrained.

--- middle

# Introduction

Attestation-Based Client Authentication {{ATTEST}} authenticates a
Client Instance through an attestation and proof of key possession.
After a key change, a new attestation alone cannot distinguish a
continuing installation from a new instance. Treating them as the same
can merge unrelated audit histories and status decisions.

Assigning each instance its own `client_id`, with a shared `software_id`
as contemplated by {{RFC7591, Section 2}}, is an alternative, but this
profile targets deployments that share one Logical Client, one metadata
URL when using {{CIMD}}, and authorization server (AS) policy and resource
authorization keyed by that client.
Those deployments need instance identity to survive verified key changes
within the shared client model, because `software_id` correlates software
registrations without defining shared grants or authorization policy for
separate client identities.

This optional profile adds two claims:

* `client_instance_id`: identifies a particular client installation or
  runtime in its Client Attestation. The attester assigns it, retains
  it across verified key changes, and scopes it to a Receiver by default.
* `client_instance`: carries a reference to that instance in a token or
  introspection response, so a resource server can correlate requests
  with the instance validated when the token was issued.

For example, an AS validates a harness's attestation and proof, then
includes a mapped instance identifier in
its access token. A resource server can use that identifier for audit
without receiving the attestation. The token's proof-of-possession
mechanism authenticates its current presenter.

The attester verifies continuity under {{lifetime}}. ATTEST alone is
sufficient when correlation need only last for the current key or can
remain internal to one system.

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

This profile is for administratively configured deployments, including
workloads and managed desktop or mobile applications. It is not a
general-purpose device or wallet identifier. It adds sender constraint
for access tokens carrying Instance Context ({{context-binding}}) but
defines no enrollment, key-rotation, or status-distribution protocol.

ATTEST supplies the authentication and proof methods. Direct
resource-server presentation follows {{ATTEST, Section 1.1}},
{{ATTEST, Section 4}}, {{ATTEST, Section 5.1}}, and
{{ATTEST, Section 7}}.

When selected under {{configuration}}, this profile applies whether the
Client Attestation is the client authentication method or an additional
security signal under {{ATTEST, Section 7.6}}. Additional attestation
does not replace the deployment's required client authentication.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

The terms Client Attestation, Client Attester, Client Instance,
and Client Instance Key are used as defined in {{ATTEST}}.

Logical Client:
: The OAuth client {{RFC6749}} identified by `client_id`. Several Client
  Instances can authenticate as the same Logical Client.

Instance Identifier:
: An opaque identifier assigned by a Client Attester to one
  Client Instance at the configured granularity. The instance identity
  is the pair `(iss, client_instance_id)`.

Instance Context:
: The `client_instance` object in a token or introspection response.
  It identifies the instance associated with the token through a
  validated attestation and proof, or validated upstream context. It
  grants no authority and does not prove current possession.

Receiver:
: A party that validates a Client Attestation under this profile,
  such as an authorization server or resource server. A Receiver that
  issues tokens carrying Instance Context also acts as a token issuer.

Context Consumer:
: A party that consumes Instance Context from a token or introspection
  response. It need not receive the original Client Attestation.

Attester Issuer:
: The value of `iss` in the Client Attestation, identifying the
  Client Attester.

Instance Authority:
: The namespace authority identified by `iss` in a `client_instance`
  object. It is the token issuer that assigned the context, including
  an upstream token issuer when context is preserved.

Enrollment:
: An attester-maintained record binding one instance, at the configured
  granularity, to its verified keys and assigned identifiers. It is
  separate from a user account, device registration, or Logical Client.

# Profile Selection and Trust {#configuration}

The client and Receiver MUST administratively configure:

* the Logical Client, authentication method, and attester trust policy;
* the instance granularity, continuity evidence, and freshness limits;
* the intended Receiver and any explicitly shared correlation scope.

A Context Consumer requiring context MUST configure that requirement.
Claims MUST NOT select this profile or change authentication methods.

Profile selection is part of the client-specific trust and evidence
agreement; no discovery or client metadata parameter is added. The
shared error in {{errors}} reports rejection, not profile discovery.

## Attester Trust

The Receiver MUST bind each approved Attester Issuer to its validation
keys and authorized Logical Clients, using configured associations or,
at an AS, accepted client endorsements under {{ATTESTER-ENDORSEMENT}}.
For requests governed by that profile, both a current client endorsement
and AS policy approval are REQUIRED.

A resource server validating Client Attestations directly uses configured
attester-to-client and key associations. The endorsement profile defines
acceptance at AS endpoints only; publishing `client_attesters` does not
establish resource-server trust.

The Receiver MUST compare issuer and client identifiers as exact,
case-sensitive strings without URI normalization. A credential's `iss`,
proof of possession, or client-published metadata alone does not
establish attester authority. Metadata from {{RFC7591}} or {{CIMD}}
requires the applicable trust policy's approval.

Local trust withdrawal MUST take effect on subsequent authentication.
Endorsement updates follow {{ATTESTER-ENDORSEMENT}}; other trust
management follows ATTEST.

## Conformance

Conformance is role-specific:

* Client Attesters implement claim, identifier, and enrollment
  requirements.
* Clients implement scoped attestation use and the selected ATTEST
  proof method.
* Receivers implement trust, validation, and applicable
  grant-continuity rules.
* Token issuers conveying context implement mapping, binding, and
  preservation.
* Context Consumers implement context validation and applicable proof
  checks.

An implementation serving several roles satisfies each role's
requirements. Downstream context is optional.

# Client Attestation Claims {#claims}

This profile uses the additional claims allowed by {{ATTEST, Section 4}}.
All ATTEST requirements apply. This profile retains
`typ=oauth-client-attestation+jwt` and `sub=client_id`.
The claims `exp` and `cnf` remain required; `iat` remains optional.

## Profile Claim Requirements

`iss`:
: REQUIRED. String exactly matching an approved Attester Issuer in
  {{configuration}}.

`client_instance_id`:
: REQUIRED. Nonempty StringOrURI {{RFC7519}}, no longer than 256
  Unicode scalar values after JSON string decoding, identifying the
  instance within the attester's namespace. This is a character limit,
  not a limit on UTF-8 octets or JSON escape sequences.
  The instance identity is `(iss, client_instance_id)`.

Assignment, Receiver scoping, and generation follow
{{attester-requirements}}.

Receivers MUST treat identifiers as opaque, compare them as exact,
case-sensitive strings without URI normalization, accept conforming
values up to that limit, and reject longer values. They MUST NOT
derive permissions by parsing an identifier. Errors follow {{errors}}.

## Receiver Scope {#identifier-scope}

The attester MUST assign distinct identifiers per Receiver unless an
administrative agreement explicitly authorizes a shared identifier
within a named set of Receivers. The client MUST request and use the
attestation for that configured scope and use distinct Client Instance
Keys across scopes. A shared client or trust domain does not authorize
sharing identifiers.

The Receiver scope is an enrollment or issuance input, not an OAuth
parameter or an attestation audience. A Receiver cannot verify that
scope from the identifier alone; privacy limits are in {{privacy}}.

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

1. Validate the attestation and proof under the configured ATTEST method.
2. Validate the claims in {{claims}} and attester authority under
   {{configuration}}.
3. Associate `(iss, client_instance_id)` with the Logical Client and
   validated Client Instance Key, then apply instance policy.

The Receiver MUST NOT:

* substitute an identifier for proof of possession or infer it from a
  key thumbprint, certificate serial number, or JWT `jti`;
* infer continuity from equal identifiers or keys across attester
  issuers; or
* set an access token's `sub`, add `act`, or extend an actor chain solely
  from instance evidence.

Migration between attester issuers requires a procedure establishing
trust in both authorities and continuity evidence. Authorization
relationships belong to the consuming authorization profile.

Key selection follows ATTEST, with sender constraint required by
{{context-binding}}. When a separate token-binding key is permitted,
context identifies the instance associated with the Client Instance Key.
Key continuity does not authorize transfer of existing tokens or grants
to a replacement key; refresh-token rebinding follows
{{grant-continuity}}.

## Grant Continuity {#grant-continuity}

For a grant established using a Client Attestation validated under this
profile, the AS MUST:

* record `(iss, client_instance_id)` when issuing a refresh token, in
  addition to ATTEST's client and key bindings; and
* validate the current attestation on refresh and require its identity
  pair to match the recorded identity.

Possession of the original key alone does not permit a different
identity, and a matching identity alone does not permit a different
key. The refresh token remains bound to the Client Instance Key under
ATTEST; a verified key change that retains the instance identity under
{{continuity}} does not rebind it. Rebinding a refresh token to a new
Client Instance Key requires a separate profile under
{{ATTEST, Section 13}}.

A refresh request for such a grant MUST NOT introduce or change the
recorded instance identity without an explicitly authorized migration.
The migration MUST establish continuity under {{continuity}} and, when
the key changes, rebind the refresh token under such a profile; no
migration protocol is defined here. An otherwise valid attestation that
conflicts with the grant's instance binding MUST produce `invalid_grant`
under {{RFC6749}}, without disclosing the expected identity.

When an issuer derives Instance Context solely from a validated input
token, the consuming profile in {{context-exchange}} MUST define refresh
binding and context continuity before permitting refresh-token issuance.
The upstream `(iss, id)` does not establish validation of the original
Client Attestation by that issuer.

If authorization-time policy bound a code or other artifact to an
instance, the AS MUST enforce that binding at redemption. Attestation
does not replace the grant's authorization, redirect, PKCE, or replay
checks. ATTEST's protocol-artifact binding guidance applies independently.

## Errors {#errors}

Missing or invalid required instance claims and rejection by instance
policy MUST produce `invalid_client_attestation`, without disclosing
whether an instance is known, suspended, or retired. This deliberately
reuses ATTEST's validation error for policy rejection. Receivers SHOULD
avoid distinguishable response timing. Unknown instances are rejected
only when local policy requires prior enrollment at the Receiver.

A failed profile check MUST NOT trigger fallback without the required
evidence. Grant-binding errors follow {{grant-continuity}}; other
authentication and freshness errors follow ATTEST.

# Attester Requirements {#lifetime}

## Identifier Generation {#attester-requirements}

For each enrollment and Receiver scope ({{identifier-scope}}), the
attester MUST assign identifiers that are:

* distinct, opaque, and non-reassignable;
* generated using at least 128 bits of cryptographically secure
  randomness or keyed pseudorandom-function output;
* unpredictable to parties other than the attester; and
* free of runtime hostnames, user identifiers, and embedded instance
  attributes.

A URI can identify the attester's namespace; its instance-specific
portion remains opaque and unpredictable.

Keyed derivation, such as HMAC {{RFC2104}}, MUST use a secret with at
least 128 bits of entropy and unambiguously encode the Logical Client,
Receiver scope, and a unique enrollment component. A platform-stable
identifier alone would reproduce retired identifiers after reinstall.
Changes to derivation inputs or secrets MUST preserve assigned values
within a continuing enrollment; storing those values is sufficient.

## Continuity and Lifecycle {#continuity}

Continuity is an unbroken, attester-recorded chain of verified key
custody within one enrollment at the configured granularity. Before
retaining an identifier, the attester MUST verify and record:

1. An active enrollment binding the instance, Logical Client, Receiver
   scope, granularity, and previously verified keys.
2. Fresh possession of the current key and authenticated evidence
   binding it to that enrollment, including an authorized custody
   transition when the key changes.
3. The configured continuity checks, their freshness, and observed
   lifecycle events or evidence of independent claimants.

The deployment MUST specify its evidence, freshness limits, and
lifecycle boundaries. For example, a container restart can leave its
Kubernetes Pod intact, but a new Pod is a new scheduling unit. Receivers
MUST NOT treat an installation or scheduling-unit identifier as
identifying its individual processes.

The attester MUST apply the following outcomes. "New enrollment" means
new identifiers that are never reassigned, including after retirement.

| Event | Required outcome |
|---|---|
| Renewal, verified key change, installation process restart, or in-place update | Retain identifiers when continuity is verified |
| Reinstall, independent clone, replacement or restart of the selected execution unit, or granularity change | New enrollment |
| Restore or snapshot rollback | Retain only with fresh evidence that the claimant succeeds the prior holder; copied keys and data alone are insufficient |
| Suspend/resume | Apply continuity checks at the next issuance using available authenticated evidence |
| Continuity cannot be established | Require new enrollment; a continuing original can retain its own enrollment |
| Detected fork of one enrollment | Retire its identifiers, stop issuance, and enroll claimants separately |

The attester MUST NOT knowingly retain identifiers for independent
instances. Concurrent processes or requests within one installation are
not by themselves a fork. The rule covers detected forks, not events
that the platform cannot observe ({{assurance}}). An identifier, expired
attestation, or former public key alone does not establish continuity.
A replacement key requires a new attestation.

## Suspension {#suspension}

The attester MUST stop issuance for suspended or retired enrollments.
A Receiver suspending an instance SHOULD revoke its issued tokens or
report them inactive through introspection {{RFC7662}}. Attesters and
Receivers SHOULD agree on short attestation lifetimes where timely
status enforcement matters.

When an AS revokes a grant because of instance suspension, retirement,
or attester trust withdrawal, it MUST invalidate all access and refresh
tokens associated with that grant and prevent further refresh issuance.
Revoked tokens are inactive under {{RFC7662}}. This local enforcement
does not itself notify resource servers validating tokens offline.

Without a status channel, existing attestations can remain acceptable
until expiration plus clock skew, and issued tokens remain valid for
their own lifetimes. Security Event Tokens {{RFC8417}} and delivery
under {{RFC8935}} can support a separate integration; this profile
defines no status event, subject mapping, or revocation-delay
guarantee.

## State and Retention {#state}

Attesters MUST retain enrollment and verification records while issuing
or renewing attestations, and status while supporting resumption.
Verification summaries suffice; raw credentials need not be retained.
Deleting continuity records requires new enrollment. Attesters MUST NOT
allow retired credentials to recreate their old identifiers.

A validating Receiver need not maintain an instance allowlist. Local
suspension, revocation, and mapped context require the corresponding
status, token associations, and mappings.

In this section, "source identity" is the mapping input under
{{context-claims}}: the instance identity `(iss, client_instance_id)`
from a Client Attestation, or the upstream `(iss, id)` being remapped.
"Consumer scope" is the Context Consumer, or explicitly configured set
of Context Consumers, to which the mapping is scoped under
{{context-claims}}.

An issuer's obligation for a mapping ends only when the issuer retires
it. An issuer MAY retire a mapping when no still-valid token or
continuing grant, including allowed clock skew, requires its continuity
and local policy no longer permits issuance of Instance Context for that
source identity and consumer scope. Until then, the issuer MUST retain
or securely reproduce the same mapping, including across periods of
inactivity. Expiration of attestations, access tokens, or refresh tokens
alone does not retire a mapping.

If issuance later resumes for a retired source identity and consumer
scope, the issuer MUST restore or reproduce the previous mapped
identifier; it MUST NOT assign a replacement identifier. An issuer that
cannot do so MUST omit `client_instance` for that source identity and
consumer scope; a Context Consumer requiring context then rejects under
{{context-errors}}. Issuers using random mappings therefore need to
retain recoverable records if they intend to support resumption.

Random generation satisfies non-reassignment probabilistically without
an indefinite retired-identifier list; derivation depends on never
reusing enrollment inputs. After a mapping is retired and the attester's
continuity and status obligations end, this profile requires no further
retention. Audit retention is local policy.

# Conveying Instance Context {#instance-context}

## Format and Mapping {#context-claims}

An issuer MAY include `client_instance` in a token or introspection
response {{RFC7662}}. Its JSON object has two REQUIRED members:

| Member | Type | Meaning |
|---|---|---|
| `iss` | Nonempty string | Token issuer that assigned the context |
| `id` | Nonempty StringOrURI | Instance identifier in that issuer's namespace |

For direct issuance from a validated Client Attestation, context MUST
identify the authenticated presenting instance. On refresh, that identity
is subject to {{grant-continuity}}. Token exchange follows
{{context-exchange}} rather than implicitly inheriting this association.

The context MUST refer to validated instance participation; issuers
MUST NOT copy unvalidated client-supplied context. From a Client
Attestation, the issuer MUST map `(iss, client_instance_id)` to its own
namespace. Remapping validated upstream context under
{{context-exchange}} maps the upstream `(iss, id)` pair to the issuer's
namespace by the same rules; the upstream `id` alone is not the mapping
input. For each mapping, it MUST:

* keep distinct instances separate unless continuity is established;
* generate opaque, unpredictable, non-reassignable identifiers under
  {{attester-requirements}} and retain them across attestation renewal
  and verified key changes;
* scope identifiers to each Context Consumer, allowing sharing only
  within an explicitly configured set; and
* retain or reproduce the mapping under {{state}}.

For derived mappings, the source authority and identifier supply the
enrollment-specific input, and the consumer supplies the scope. Issuers
MUST separate this derivation from attester identifiers, for example by
using a distinct key or purpose label. {{wire-examples}} shows mapped
context in a JWT access token and an introspection response.

## Access Token Binding {#context-binding}

An AS MUST sender-constrain access tokens carrying `client_instance`,
including when context is conveyed only through introspection, using
Demonstrating Proof of Possession (DPoP) {{RFC9449}}, mutual TLS
{{RFC8705}}, or another mechanism defined by the consuming profile. A
resource server consuming such a token MUST validate its binding and
required proof, and reject an unconstrained token under
{{context-errors}}. Proof errors follow the selected binding
mechanism.

The binding authenticates the authorized token presenter. It does not
establish that the presenter is the instance named in context derived
from an upstream token, whether preserved or remapped. Key selection
follows ATTEST; any presenter or key change during exchange requires
authorization under the consuming exchange profile. Unlinkability
between Context Consumers additionally requires distinct binding keys
under {{privacy}}.

## Preservation and Authorization {#context-exchange}

An exchange issuing Instance Context MUST use a consuming profile that
defines:

* whether context identifies the authenticated presenting instance or
  an instance represented by validated input-token context;
* its association with the subject, actor, or presenter and how that
  association is validated; and
* when input context is replaced or preserved.

An issuer MUST NOT treat upstream context as identifying a different
presenting instance. The object identifies one instance, not a chain.

An issuer MAY preserve validated upstream `(iss, id)` instead of
mapping it when configured trust and correlation scope authorize its
disclosure to the downstream consumer. Otherwise it MUST map or omit
the context, subject to the consuming profile's requirements.

An issuer MUST limit preservation to input context whose `iss` equals
the authenticated input-token issuer. It MUST remap or omit context
already preserved by an intermediary unless a consuming profile specifies
authenticated provenance, a finite hop limit, and enforcement rules.
This permits one preservation hop by default; the object carries no
forwarding history. The limit prevents forwarding context whose
provenance the issuer has not authenticated; it does not bound the
number of exchanges through which an instance's participation is
conveyed, because each remap is a new mapping of validated input-token
context.

Context MUST NOT be treated as a separate token or delegated actor.
Remapping follows {{context-claims}} and, like preservation, leaves the
identified instance unchanged; presenter authentication follows
{{context-binding}}.

## Context Consumer Processing

Before using context, the Context Consumer MUST:

1. Validate the enclosing token or authenticated introspection response.
2. Validate the context members, comparing exact, case-sensitive strings
   without URI normalization; ignore unrecognized members.
3. Accept its authority only when it is the token issuer or an upstream
   token issuer explicitly trusted for that issuer and consumer.
4. Reject invalid context and, when context is required, reject the
   request if context is missing or invalid under {{context-errors}}.

A Context Consumer MUST establish the context's association with the
subject, actor, or presenter from the applicable consuming profile and
the validated token or trusted introspection configuration before using
that association in policy or audit. If the association is not
established, it MUST treat context only as evidence of instance
participation and MUST NOT attribute the current request to that
instance as its presenter. A Context Consumer whose configured
requirement ({{configuration}}) includes that attribution MUST reject
the request under {{context-errors}}. The `client_instance` object
alone, including whether its `iss` matches the token issuer, does not
establish this association.

When the consumer's trusted configuration establishes that the token
issuer conveys context only from direct Client Attestation validation,
this document is the applicable consuming profile: context identifies
the authenticated presenting instance under {{context-claims}}, and
validating the token's binding under {{context-binding}} establishes
that association.

For introspection, trusted endpoint configuration identifies the
expected token issuer. A response-level `iss`, if present, MUST match
it; neither the endpoint URL nor `client_instance.iss` selects the
issuer. Multi-issuer introspection requires a consuming profile that
authenticates the represented issuer.

An authority identifier does not authorize fetching keys from that
location. Consumers relying on granularity MUST configure it rather
than parse the identifier. Extensions MUST specify their processing
without changing the meaning of `iss` or `id`.

## Errors {#context-errors}

Rejection for missing or invalid required context, or for required
attribution that cannot be established, MUST use:

* `invalid_token` at a resource server ({{RFC6750, Section 3.1}}); or
* `invalid_request` for a rejected subject or actor token in an exchange
  ({{RFC8693, Section 2.2.2}}).

Other consuming profiles MUST define their error mapping. Direct Client
Attestation failures follow {{errors}}.

# Relationship to Other Identity Systems

## Client ID Metadata Documents {#cimd}

With {{CIMD}}, the metadata URL is the Logical Client's `client_id`.
Many installations can use that URL. Fetching its public metadata does
not prove that a caller is an authorized instance:

| Mechanism | Contribution |
|---|---|
| CIMD | Discovers Logical Client metadata and its authentication method |
| ATTEST | Authenticates an attester-approved instance and possession of its key |
| This profile | Retains instance identity across verified key changes and conveys optional downstream context |

Use one CIMD URL for the Logical Client and keep instance identifiers
in attestations, not separate metadata documents. The Client
Attestation's `sub` remains that exact URL; `client_instance_id`
distinguishes its installations or runtimes. {{cimd-example}} shows the
existing ATTEST metadata used for this composition.

CIMD can remove registration of client metadata at each AS; it does
not remove this profile's trust agreement ({{configuration}}).
{{ATTESTER-ENDORSEMENT}} supplies `client_attesters` metadata for clients
to endorse attesters, subject to AS acceptance policy. It can establish
the attester-to-client association without individually configured
attesters, but does not select this optional identification profile or
its continuity and privacy policy. Both profiles also support locally
registered client metadata.

## Workload and Agent Credentials

An integration MUST NOT assert that a shared workload identity identifies
an individual instance without additional authenticated evidence.
Direct SPIFFE Verifiable Identity Document (SVID) authentication and
client mappings follow {{SPIFFE-OAUTH}}. An AAuth Agent Provider
{{AAUTH}} or workload attester can instead issue a Client Attestation
using this profile when it has the required enrollment evidence.
Native credentials with different `sub` or `typ` semantics require a
separate carrier profile; they are not implicitly conformant.
{{deployment-examples}} illustrates these boundaries.

# Security Considerations

The security considerations of {{ATTEST}} and {{RFC8725}} apply.

## Attester Compromise and Assurance {#assurance}

A compromised attester can impersonate instances within its approved
client associations. Receivers MUST limit those associations and
configure acceptable evidence assurance; attesters MUST NOT claim
stronger assurance than their evidence supports. Self-reported,
platform-verified, and hardware-rooted evidence are not interchangeable.

Copied keys and enrollment data can be indistinguishable from the
original without independent platform evidence. {{continuity}} governs
detected forks, not guaranteed clone detection. Instance identity does
not prove software integrity beyond the evaluated evidence.

## Forwarding and Privacy {#privacy}

* **Proof binding:** the Client Attestation has no audience. Its
  proof-of-possession (PoP) JWT identifies the Receiver; combined DPoP
  binds the HTTP request. Forwarding resistance depends on ATTEST proof
  validation, freshness, and key possession, not identifier scope.
* **Correlation:** {{identifier-scope}} requires separate identifiers
  and keys across Receiver scopes, consistent with
  {{ATTEST, Section 11.1}}. Receivers MUST NOT assume identifiers across
  scopes are comparable. Explicitly shared scopes permit correlation.
* **Token-binding keys:** scoping Instance Context to each consumer
  limits correlation through its identifier; it does not guarantee
  unlinkability between consumers. An AS can issue tokens with different
  mapped identifiers but the same DPoP `cnf.jkt`, allowing resources to
  correlate them while that key is reused. In DPoP combined mode
  ({{ATTEST, Section 5.2}}), the Client Instance Key is the DPoP key,
  so every token bound to one attestation shares it. Scoping
  attestations to an AS does not by itself separate binding keys
  between resources served by that AS.
  Where unlinkability between Context Consumers is required, the client
  MUST use distinct token-binding keys across those consumer scopes, and
  the deployment MUST address other correlating token claims and
  application data. ATTEST permits DPoP without combined mode, with a
  DPoP key independent of the Client Instance Key; that mode, or a
  distinct mutual-TLS certificate per consumer scope, satisfies this
  requirement. This applies to DPoP keys, mutual-TLS certificate keys,
  and other binding mechanisms. Key selection follows {{processing}} and
  refresh-token binding follows {{grant-continuity}}; identifier scoping
  does not permit changing a refresh token's bound key.
* **Attester visibility:** supplying Receiver scope reveals it to the
  attester. Deployments requiring ATTEST's audience-hiding property
  should omit this profile. Other claims and application data can also
  correlate clients despite scoped identifiers.
* **Disclosure:** status non-disclosure follows {{errors}} and retention
  follows {{state}}. Error responses SHOULD avoid revealing unrelated
  instance identities. Logs MUST NOT contain raw credentials or private
  keys. Identifier unpredictability limits guessing and enumeration;
  it is not authentication.

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
the token issuer; the nested `iss` names the Instance Authority. They
coincide in this mapped example.

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

## Governed Actor and Runtime Context
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
the verifier, redirect URI, `C1`, attestation, and DPoP proof. The browser
receives neither attestation nor proof. Process restarts can retain the
installation identity; reinstall requires new enrollment.

# Document History {#history}
{:numbered="false"}

*RFC EDITOR: Remove this section before publication.*

* Initial draft.
