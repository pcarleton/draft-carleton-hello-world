# The two JWT slots at the token endpoint

Working notes for how WAG's authorization grant relates to the
near-identical-looking client-authentication JWTs (private_key_jwt,
attestation-based client auth, SPIFFE client auth). Not part of the draft;
kept here so we can decide later what to incorporate.

## Anatomy of a token request

RFC 7521 defines two independent assertion slots in one token request:

```
POST /token
  grant_type=urn:ietf:params:oauth:grant-type:jwt-bearer
  assertion=<JWT A>                                  <- GRANT slot
  client_assertion_type=...:jwt-bearer
  client_assertion=<JWT B>                           <- CLIENT AUTH slot
  resource=https://api.example
```

- JWT A (grant, RFC 7523 s2.1): "issue an access token FOR the subject of
  this JWT." Answers: who is the token about, and on whose say-so.
- JWT B (client auth, RFC 7523 s2.2): "the party making this request is
  client X." Answers: who is calling the endpoint.

The slots are orthogonal. A request can carry either or both.

## Where WAG sits

WAG uses only the GRANT slot: the platform (or an IdP) signs JWT A with
sub = the Agent Identifier. The agent is the SUBJECT of the grant, not the
OAuth client, which is why the draft attaches no semantics to client_id
(decision D4). Confusion risk: oauth-spiffe-client-auth puts a
similar-looking platform-signed JWT at the same endpoint in the OTHER slot,
where it means client authentication. Same wire shape, different question
answered. The draft should eventually state this in one paragraph.

## Compositions (what each buys)

| # | Composition | Protects | Cost |
|---|---|---|---|
| 1 | Grant only (bearer) | Nothing beyond TLS; adoption floor | Assertion theft = replay within lifetime; mitigate with jti replay cache + short exp |
| 2 | Grant + CIMD/private_key_jwt (platform authenticates as client) | Endpoint knows the caller; natural when grant issuer is the Enterprise IdP and platform exchanged for it | Platform needs a client identity per RS ecosystem |
| 3 | Grant + attestation-based client auth (platform attests instance key; instance signs PoP JWT) | Stolen assertion alone is useless; issuance requires instance-held key | Instances must hold keys; ATTEST draft still maturing |
| 4 | Grant + DPoP (RFC 9449) | Issued ACCESS TOKEN is sender-constrained end to end | RS must validate DPoP proofs -- breaks "only the token endpoint changes" |
| 5 | Grant carrying cnf (instance key) + AS requires matching DPoP proof at the token endpoint | Binds the assertion itself AND the issued token to the instance key | Not an off-the-shelf AS behavior; needs profile text |

Rows 2 and 3 live in the CLIENT AUTH slot and compose with the WAG grant
rather than competing with it. Row 4/5 add sender-constraint on top of any
of the others.

## Current lean

Floor = row 1 hardened (mandatory jti replay rejection, short lifetimes,
assertion valid only at the token endpoint). Recommended hardening where
instances hold keys = row 3 (or row 5 for deployments that already do DPoP).
Row 2 is the IdP-issuer composition and is orthogonal. Open as O1 in
decision-log.md.
