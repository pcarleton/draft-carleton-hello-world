# Decision log

Design decisions for draft-carleton-jif, with the alternatives considered.
Most recent first within each state. Open items are at the bottom.

## Decided

### D10 — Retirement: state the bound, defer the signal (2026-07-28)
Retirement is cessation of signing; the draft states the residual-access
bound (remaining assertion lifetime + access-token lifetime) normatively and
keeps epoch/event signaling in Open Issues as a gap shared with WIMSE and
SPIFFE (neither has per-credential revocation either). Broader lifecycle
management (e.g. succession planning for resources owned by a retired agent)
is explicitly out of scope.
Alternatives: specify an issuer- or property-scoped epoch mechanism
(premature, no ecosystem precedent); event push a la SSE/CAEP (unstandardized
everywhere).

### D9 — Terminology: standard AS/RS split (2026-07-28)
"Authorization Server" and "Resource Server" are used in their standard
OAuth senses. Alternatives: composite "Resource Server" meaning the combined
AS+resource role a SaaS product presents (rejected: forces readers to track
an RS containing its own AS, and misreads in both OAuth and WIMSE review
communities); a coined composite term like "SaaS Service" (rejected: not a
real word, and combining the roles does nobody any favors).

### D8 — Multi-issuer scoping is normative (2026-07-28)
Keys are resolved and cached per allowlist entry and never merged across
issuers; sub, jti, and property-to-permission mappings are interpreted only
within the presenting iss; policy/audit keys are (iss, sub) or the complete
URI-form identifier. Alternative: leave tuple semantics implicit (rejected:
an AS indexing agents by sub alone lets tenancy B's agent inherit tenancy
A's mapping).

### D7 — Agent Identifier: URI form RECOMMENDED, bare string permitted (2026-07-28)
The identifier MAY (and is RECOMMENDED to) be a URI-form workload identifier
with an opaque path; a bare opaque string remains permitted. Role split: the
AS validates the URI authority against the allowlisted issuer's tenancy once
at issuance; RSes exact-match the complete identifier as opaque and never
derive trust from its components. URI form costs no opacity and carries the
tenancy inside sub for relying parties that evaluate only subject/audience.
Alternatives: bare-string only (conflicted with the normative WIMSE-ID
citation, which requires absolute URIs); URI-required (excludes simple
platforms for no gain).

### D6 — aud carries both issuer identifier and token endpoint URL (2026-07-28)
The assertion's aud SHOULD list both values; an AS MUST accept either.
rfc7523bis adds the issuer identifier as an audience option for
authorization grants (issuer-only is mandated only for the separate
client-authentication slot); carrying both keeps one assertion valid across
deployed and future processing.
Alternatives: token-endpoint-only (the value the ecosystem is moving away
from); issuer-only (breaks deployed ASes that match on the endpoint);
SHOULD-issuer with endpoint fallback (more prose, same effect, worse
compatibility story than dual-listing).

### D5 — IdP projection may be just-in-time (2026-07-27)
Projection into an Enterprise IdP (e.g. SCIM) must not be required ahead of
time; performing it just in time, including synchronously during first token
issuance, is acceptable. Supersedes the earlier stricter "MUST NOT gate
agent creation or token issuance".

### D4 — Client identity deliberately unspecified (2026-07-27)
The profile attaches no semantics to client_id. Simplest deployment is
unauthenticated; the door stays open for the platform to authenticate as its
own OAuth client (e.g. CIMD + private_key_jwt) with the grant issued by an
Enterprise IdP. Alternatives: public client with client_id = agent
identifier (earlier position, reversed: forecloses the platform-as-client
composition); mandatory client authentication (too heavy a floor).
See notes/oauth-slots.md.

### D3 — RFC 7523 authorization grant, not token exchange or client credentials (2026-07)
The agent presents a platform-signed JWT in the assertion parameter
(grant slot, RFC 7523 s2.1, third-party issuer per RFC 7521 s3).
Alternatives: RFC 8693 token exchange (requires a subject_token_type no
deployed AS ships; more moving parts at the endpoint); client credentials
with a 7523 client assertion (makes the agent the OAuth client, forcing
sub == client_id, which conflates the subject-of-the-grant with the
client role). Chosen for the adoption goal: jwt-bearer is the one grant
type every mainstream AS already implements.

### D2 — docname and repo: draft-carleton-jif (2026-07-27)
Short, speakable, matches how people will refer to it. Alternative:
draft-carleton-aims-agent-platform (descriptive but unpronounceable in
conversation).

### D1 — Mechanism name: just-in-time identity federation (JIF) (2026-07-27)
Criteria: pronounceable acronym; shape-legible to workload identity
federation without being called WIF; modest namespace claim; names the
mechanism (no registration step) rather than the audience.
Alternatives: WIFA "workload identity federation for agents" (pronounceable
and shape-clear, but contains WIF verbatim and concedes the agents-are-just-
workloads framing); AWIF (worse mouth-feel, buries the legible part); AIF
"agent identity federation" (unpronounceable, overclaims a broad term);
SCUBA "(S...) Credentials Under Backend Authorization" (memorable, zero
shape-legibility; reserved as a candidate name for a future standalone
protocol document).

## Open

### O1 — Proof of possession mechanism
Bearer grant is the adoption floor (with jti replay rejection and short
lifetimes under consideration as mandatory). Candidates for the hardened
mode: attestation-based client authentication (platform as attester,
instance-signed PoP JWT); DPoP (binds the issued access token, but requires
RS-side changes, cutting against the token-endpoint-only adoption goal);
assertion cnf + key-matched DPoP proof at the token endpoint (binds the
assertion itself to an instance key). See notes/oauth-slots.md.

Refinement (2026-07-28): client authentication also closes assertion theft,
IF the profile adds a binding rule -- "assertions from allowlisted issuer I
are accepted only from its bound, authenticated client C" (RFC 7523 does not
link the grant to client authentication by itself). Given that rule, the
choice between CIMD+private_key_jwt and attestation-based client auth is
topology, not capability: pkjwt authenticates the Platform with one client
key (right when the platform core fronts all token requests; platform-wide
blast radius on key theft), attestation pushes a key into each instance
(right when instances call the AS directly; per-instance blast radius, and
the AS can require attestation sub == grant sub). DPoP remains the
orthogonal token-binding layer (protects stolen access tokens, costs
RS-side changes).

Sharper cut (2026-07-28, paulc): against a whole-request thief (inside TLS
termination) every shape degrades to replay-within-freshness-window -- the
pkjwt/PoP artifacts travel next to the assertion, so signing buys nothing
there; short exp + jti replay caching are the real defense and belong in the
floor. Client authentication only matters when the assertion can exist APART
from the caller's key material: issuer != caller (IdP-issued grants),
at-rest leaks (logs/queues), or the AS wanting a caller identity for
quota/kill-switch. pkjwt vs attest is then just which boundary is being
defended: issuer != caller (platform key) vs platform != instance
(per-instance keys). Caveat (paulc): attest's per-instance blast radius
applies only to edge keys -- the attester key is itself a platform-level
durable key (~= the issuer key; possibly the same key), so at the root
pkjwt and attest carry identical concentration risk.

### O2 — Property claim naming and registration
Unregistered generic names (roles, groups) crossing an inter-organizational
boundary will not survive review. Options under consideration:
(a) reuse existing IANA-registered claims (name from OIDC Core; roles,
groups, entitlements from RFC 9068/SCIM semantics) and register only the
genuinely new ones (namespace, ctx);
(b) one registered container claim (e.g. agent_properties) holding the
vocabulary, avoiding all top-level collisions;
(c) collision-resistant (URI-prefixed) interim names;
(d) present the options in the -00 text and let reviewers weigh in.

### O3 — "Relationship to WIMSE and SPIFFE" section text
Outline agreed (slot argument; identifier compatibility; identity-plane
backing changes nothing at the RS boundary). Drafting waits on O1 and O2
since both feed the section's content.
