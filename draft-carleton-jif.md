---
title: "Just-in-Time Identity Federation (JIF): An AIMS Profile for Agent Platforms"
abbrev: "JIF for Agent Platforms"
category: info

docname: draft-carleton-jif-latest
submissiontype: IETF
number:
date:
v: 3
area: "sec"
keyword:
 - agent identity
 - AIMS
 - workload identity
 - just-in-time identity federation
venue:
  group: WIMSE
  type: Working Group
  mail: wimse@ietf.org
  github: pcarleton/draft-carleton-jif

author:
 -
    fullname: Paul Carleton
    organization: Anthropic
    email: paulc@anthropic.com

normative:
  RFC7517:
  RFC7519:
  RFC7521:
  RFC7523:
  RFC8414:
  RFC8707:
  RFC9525:
  AIMS: I-D.klrc-aiagent-auth
  WIMSE-ID: I-D.ietf-wimse-identifier
  OIDC-DISCOVERY:
    title: OpenID Connect Discovery 1.0 incorporating errata set 2
    target: https://openid.net/specs/openid-connect-discovery-1_0.html
    date: 2023-12
    author:
      - ins: N. Sakimura
      - ins: J. Bradley
      - ins: M. Jones
      - ins: E. Jay

informative:
  RFC7591:
  RFC8693:
  WIMSE-ARCH: I-D.ietf-wimse-arch
  CIMD: I-D.ietf-oauth-client-id-metadata-document
  ATTEST-CLIENT-AUTH: I-D.ietf-oauth-attestation-based-client-auth
  TXN-TOKENS: I-D.ietf-oauth-transaction-tokens
  SCIM-AGENT: I-D.wzdk-scim-agent-resource
  IDJAG: I-D.ietf-oauth-identity-assertion-authz-grant
  MCP-WIF:
    title: "Workload Identity Federation (Model Context Protocol ext-auth extension, draft)"
    target: https://github.com/modelcontextprotocol/ext-auth/blob/main/specification/draft/workload-identity-federation.mdx
    author:
      - org: Model Context Protocol Community

--- abstract

This document profiles the Agent Identity Management System (AIMS)
framework for agent platforms that host many agent instances per customer.  Each agent is identified by an
opaque, non-reassignable agent identifier; the agent obtains access tokens by
presenting a JWT authorization grant (RFC 7523), signed by the platform's
per-tenancy issuer, in the assertion parameter at the resource server's
authorization server.  Trust is established once, by reference to the
issuer's published metadata and keys; agent creation requires no per-agent
step at the resource server; and authorization is expressed over
platform-asserted agent properties that resource servers map locally to
permissions.

--- to_be_removed_note_Note_to_Readers

This document is an early, exploratory individual draft, published to solicit
discussion of the deployment pattern it describes.  It is not a working group
document, does not describe a shipped or committed design, and does not
represent a position or roadmap of the author's employer.  Every aspect of it
is subject to change or withdrawal, including whether this profile should
exist as a separate document at all.  Most sections are placeholders.  Issues
and pull requests:
https://github.com/pcarleton/draft-carleton-jif.

--- middle

# Introduction

Agent platforms increasingly host many agent instances per customer, created
and retired at the cadence at which the customer organizes its work -- per
channel, repository, or pipeline.  An individual agent may persist for weeks
or months, but the person creating it is typically not the person authorized
to provision credentials at the resource servers it will access.  Per-agent
credentials are therefore not issued in practice, and deployments collapse to
a single credential shared across all agents of an installation -- the pattern
{{AIMS}}, Section 7, identifies as an antipattern -- at the cost of any
attribution of an individual agent's actions at the resource server.

Existing workload identity federation {{WIMSE-ARCH}} addresses an analogous
problem between an organization's workloads and infrastructure providers, but
its claim and policy semantics are defined per provider rather than portably,
and it has not been applied between agent platforms and SaaS resource
servers.  The Model Context Protocol's Workload Identity Federation extension
{{MCP-WIF}} defines the corresponding wire mechanics for MCP servers; this
profile is intended to be interoperable with it.

This document specifies a profile of {{AIMS}} for that deployment pattern.
The resulting mechanism is called just-in-time identity federation (JIF):
trust in an issuer is established once, by reference, and individual agents
are provisioned at the resource server just in time, on first presentation,
with no per-agent registration step.

## Relationship to AIMS

{{AIMS}} remains the base specification for anything not restated here.  This
profile constrains {{AIMS}} as follows:

| AIMS component | Here | Relationship |
|---|---|---|
| Identifiers (Sec. 6) | {{identity-model}} | Constrained: opaque, immutable, non-reassignable agent identifier per {{WIMSE-ID}}, carried as sub |
| Credentials (Sec. 7), Authentication (Sec. 9) | {{authorization-grant}} | Constrained: {{RFC7523}} JWT authorization grant (assertion), platform-signed; no per-agent credential; client identity deliberately unspecified |
| Credential Provisioning (Sec. 8) | {{instantiation}} | Constrained: platform-internal |
| Authorization (Sec. 10, case 10.4.2) | {{properties}} | Added: standard property claims; by-reference trust ({{trust}}) |
| Monitoring/Remediation (Sec. 11) | {{oi}} | TODO |
| Policy (Sec. 12), Compliance (Sec. 13) | -- | Inherited |

## Relationship to WIMSE and SPIFFE

TODO.  A reader arriving from WIMSE or SPIFFE will ask where the Workload
Identity Token and the SPIFFE ID are in this design.  Explain: why the
assertion is an {{RFC7523}} authorization grant rather than a WIMSE WIT;
whether the Agent Identifier can be, or deliberately is not, a {{WIMSE-ID}}
identifier; and what changes if the Platform's issuer is backed by a
SPIFFE/WIMSE-style workload identity plane rather than operated as a
standalone OAuth issuer.

## Scope

In scope: agents acting on their own behalf ({{AIMS}}, Section 10.4.2).  Out
of scope: on-behalf-of access (see {{IDJAG}}), legacy integration via static
credentials, runtime attestation, agent-to-agent protocols.

# Conventions and Terminology

{::boilerplate bcp14-tagged}

Agent Platform ("Platform"):
: The service that hosts Agents and operates the per-tenancy issuers that
  vouch for them.

Agent:
: A hosted workload with its own Agent Identifier, context, and
  configuration.

Agent Property:
: A Platform-asserted attribute carried as a claim in the authorization
  grant.

Resource Server (RS):
: A service holding customer resources, used here for the combined
  authorization-server-plus-resource role a SaaS product presents.

Enterprise IdP:
: The customer's identity provider (optional).

Customer Administrator:
: The human who performs one-time trust establishment.

# Deployment Model

TODO: diagram.  One-time trust establishment by the Customer Administrator
({{trust}}); agent creation by end users with no interaction with the
Resource Server ({{instantiation}}); per-request JWT authorization grant
({{authorization-grant}}).

# Agent Identity Model {#identity-model}

An Agent's identity has three units: the tenancy's issuer, which signs
assertions about the Agent; the Agent Identifier, opaque, unique within the
issuer, immutable, and never reassigned, carried as the assertion's sub
({{authorization-grant}}); and claims, carrying everything else
({{properties}}).  Renaming an Agent MUST NOT change its Agent Identifier.
Resource Servers MUST NOT parse or pattern-match the Agent Identifier for
authorization; single-agent policy is an exact match on it.

TODO: Agent Identifier as opaque string vs. URI form (cf. {{WIMSE-ID}}).

# Authorization Grant {#authorization-grant}

The Agent obtains access tokens from the Resource Server's authorization
server by presenting a JWT as an authorization grant per {{RFC7523}},
Section 2.1, issued by the Platform as a third party in the sense of
{{RFC7521}}, Section 5.2.  The token request carries
grant_type=urn:ietf:params:oauth:grant-type:jwt-bearer, the JWT in the
assertion parameter, and the target resource in the resource parameter
{{RFC8707}}.  This profile deliberately does not specify the OAuth client
identity or attach semantics to client_id.  In the simplest deployment the
token request is made without client authentication.  Deployments MAY layer
client authentication on top -- for example, the Platform authenticating as
an OAuth client in its own right with a Client ID Metadata Document {{CIMD}}
and a private_key_jwt client assertion -- a composition that becomes natural
when the authorization grant's issuer is the customer's Enterprise IdP
rather than the Platform (obtained, e.g., by token exchange {{RFC8693}} with
the IdP).  This document intentionally leaves that composition open rather
than fully specifying it ({{oi}}).

In the assertion, iss is the tenancy's issuer identifier, sub is the Agent
Identifier, and aud is the authorization server's token endpoint URL; exp,
iat, and a unique jti are REQUIRED.  The assertion is signed under a key in
the issuer's published JWK Set {{RFC7517}}.  The authorization server MUST
resolve the signing key by iss ({{trust}}), not via a client registration.
The assertion carries the Agent's Properties ({{properties}}), and the
authorization server MUST make them available to its Resource Server's
authorization decision.  Authorization servers supporting this profile MUST
include urn:ietf:params:oauth:grant-type:jwt-bearer in grant_types_supported
in their metadata {{RFC8414}}.

{{fig-token-request}} shows an example token request (with extra line
breaks for display purposes only):

~~~
POST /token HTTP/1.1
Host: as.saas.example
Content-Type: application/x-www-form-urlencoded

grant_type=urn%3Aietf%3Aparams%3Aoauth%3Agrant-type%3Ajwt-bearer
&assertion=eyJhbGciOiJFUzI1NiIsImtpZCI6IjIwMjYtMDctMTQi...
&resource=https%3A%2F%2Fapi.saas.example%2F
~~~
{: #fig-token-request title="Example token request"}

{{fig-assertion-claims}} shows the decoded claims of the assertion carried
in that request; namespace, groups, roles, and ctx are Agent Properties
({{properties}}):

~~~
{
  "iss": "https://agents.platform.example/acme",
  "sub": "agent_7f3d9a2e",
  "aud": "https://as.saas.example/token",
  "exp": 1785271980,
  "iat": 1785271680,
  "jti": "7d0f5a2b-93c8-4f0e-9c33-1b6a0e6d5f10",
  "namespace": "acme/support",
  "groups": ["support-eng"],
  "roles": ["responder"],
  "ctx": "channel:C0123456789"
}
~~~
{: #fig-assertion-claims title="Example assertion claims"}

No refresh tokens are issued; access tokens are short-lived and
audience-restricted {{RFC8707}}.  Assertion lifetimes SHOULD be as short as
system availability constraints allow.

TODO: proof-of-possession -- the assertion is a bearer grant; options are
sender-constrained access tokens, or attestation-based client authentication
{{ATTEST-CLIENT-AUTH}} (Platform as client attester, agent instance signs the
PoP JWT) if agent instances hold keys.

# Trust Establishment {#trust}

The Customer Administrator once records, at the Resource Server, an allowlist
entry binding one issuer to one tenancy: the issuer identifier, from which
the issuer's metadata and JWK Set location are discovered per
{{OIDC-DISCOVERY}} (retrieved over https {{RFC9525}}), and the initial
mapping from Properties to permissions ({{properties}}).  Establishment is by
reference; no keys or secrets are transferred, and key rotation is by JWK Set
update alone.  Platforms serving multiple customers MUST use a distinct
issuer per tenancy: the issuer is the trust boundary the allowlist expresses,
and a shared issuer would move tenancy enforcement into claim evaluation at
every Resource Server -- including those that can evaluate only subject and
audience ({{oi}}).  This deliberately tightens the corresponding SHOULD in
{{MCP-WIF}}.

# Agent Instantiation {#instantiation}

Creating an Agent is Platform-internal and MUST NOT require any
ahead-of-time interaction with the Resource Server, its authorization
server, an Enterprise IdP, or the Customer Administrator.  A Resource Server
first learns that an Agent exists when the Agent presents its first
authorization grant: it MUST accept a previously-unseen Agent Identifier
presented as sub under an allowlisted issuer and MUST authorize it via
{{properties}} rather than by identifier structure.  Agents are not
dynamically registered clients {{RFC7591}}.

Optionally, a Platform MAY project Agents into an Enterprise IdP (e.g., as
{{SCIM-AGENT}} resources) for inventory and lifecycle governance; such
projection MUST NOT gate agent creation or token issuance.  TODO: BYO-IdP
deployment model.

# Agent Properties and Authorization {#properties}

TODO.  Initial standard property claims: namespace, groups, roles, and an
optional ctx naming the collaboration context; additional attributes use
collision-resistant claim names per {{RFC7519}}, Section 4.3.  A Resource
Server keeps a local, administrator-controlled mapping from Property
predicates to permissions; possession of a Property is not itself
authorization.  Deny semantics do not travel.  TODO: worked example.

# Attribution

TODO.  Resource Servers log the Agent Identifier (sub), referenced
Properties, and jti; internal fan-out via {{TXN-TOKENS}} rather than
forwarding the access token.

# Open Issues {#oi}

- Resource Servers that authorize only on subject and audience (e.g., cloud
  IAM federation trust policies) and cannot evaluate Property predicates.
- Proof-of-possession: the authorization grant is bearer; see
  {{authorization-grant}}.
- Issuer placement: issuer operated by the Platform versus by the Enterprise
  IdP, with the Platform obtaining assertions by token exchange {{RFC8693}}
  with the IdP; client authentication is redundant in the former case and
  load-bearing in the latter; what, if anything, client_id means in each
  case.
- Staleness signaling in place of per-agent revocation (issuer- or
  Property-scoped epoch versus event push versus lifetime alone).
- Retirement: no signal currently informs a Resource Server that an Agent
  Identifier is permanently retired.
- Agent Identifier format: opaque string versus URI.

# Security Considerations

TODO: unseen agent identifiers under trusted issuers; issuer allowlist as the
trust boundary; tenant confusion at multi-issuer Resource Servers;
bearer-assertion theft and assertion lifetime; Platform as root of trust;
credential non-exposure to the model; automated trust establishment.

# IANA Considerations

This document has no IANA actions at this time; provisional claim names
({{properties}}) may be registered in a future revision.

--- back

# Acknowledgments
{:numbered="false"}

The author thanks Kevin Kelley, Aaron Parecki, Brian Campbell, Nick
Steele, Emily Lauber, and Maxwell Gerber for discussions that shaped this
document.  This profile builds directly on the Agent Identity Management
System framework {{AIMS}} and would not exist without it.  Further
acknowledgments will be added in a future revision.
