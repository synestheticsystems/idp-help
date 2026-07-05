# OIDC on IDP.to: current status and integration guide

IDP.to is a hosted, passkey-first identity provider. This page describes how to integrate with it over OpenID Connect (OIDC), and — just as important — what is and is not available today.

Read the status section before you plan any work. Some of the pieces you need to complete a login are not live yet.

## Status today

**As of July 3, 2026:**

- OIDC **discovery** and **JWKS** are live in production. You can fetch both right now (see [What's live](#whats-live)).
- The **authorization endpoint** and **token endpoint** are **not yet implemented**. They are advertised in the discovery document, but requests to them currently return `404`.
- **Application / client registration** is not yet available. There is no way to obtain a `client_id` or `client_secret` yet.

The practical consequence: **an external application cannot complete an OIDC login against IDP.to today.** You can fetch the discovery document and JWKS, and you can pre-wire a standard OIDC client library against them, but the interactive flow will not succeed until the authorization and token endpoints ship and client registration opens.

The rest of this page explains what works now, what is coming, and how to prepare.

## What's live

Two endpoints are live in production.

### Discovery

`https://admin.idp.to/.well-known/openid-configuration`

This is the standard OIDC discovery document. It advertises the issuer, the endpoint URLs, and the supported parameters. A conformant OIDC client library reads this document to configure itself, so you generally do not hard-code endpoint URLs yourself.

The live response:

```json
{
  "authorization_endpoint": "https://admin.idp.to/oidc/authorize",
  "claims_supported": [
    "sub",
    "iss",
    "aud",
    "exp",
    "iat",
    "nonce",
    "email",
    "name",
    "org_id",
    "identities"
  ],
  "code_challenge_methods_supported": [
    "S256"
  ],
  "grant_types_supported": [
    "authorization_code"
  ],
  "id_token_signing_alg_values_supported": [
    "RS256"
  ],
  "issuer": "https://admin.idp.to",
  "jwks_uri": "https://admin.idp.to/oidc/jwks",
  "response_types_supported": [
    "code"
  ],
  "scopes_supported": [
    "openid",
    "profile",
    "email"
  ],
  "subject_types_supported": [
    "public"
  ],
  "token_endpoint": "https://admin.idp.to/oidc/token",
  "token_endpoint_auth_methods_supported": [
    "client_secret_basic",
    "client_secret_post",
    "none"
  ]
}
```

A few things to note:

- `authorization_endpoint` and `token_endpoint` are listed here, but they are **not live yet** — see [Status today](#status-today). Discovery advertises where they will be; it does not mean they respond yet.
- The only supported response type is `code`, and the only supported code challenge method is `S256`. IDP.to uses the authorization-code flow with PKCE. There is no implicit flow.
- ID tokens are signed with `RS256`.

### JWKS

`https://admin.idp.to/oidc/jwks`

This is the JSON Web Key Set: the public keys your application uses to verify the signature on ID tokens. The `jwks_uri` in the discovery document points here. Your OIDC library normally fetches and caches this for you.

The live response:

```json
{"keys":[{"alg":"RS256","e":"AQAB","kid":"gjQxgxj1uCd-ysXUEg_qTyT3s1VflmrLVqn7Zh9H5Qs","kty":"RSA","n":"zILOdpaSPdTILLlwP5ehSKIslZzsLJMyb_oHZZ1XZCRaBH4lf4lRoGSgp0IhbNtAM-VPPAawD4a3fi_qf-PReGn2RxtkeHeLwTTCI6TCGMjsUxfpF8gEMK0uTWjZRyGcTkUpI3wEcgSrb_MJEhBjmTEs78cf7yZTnoduuIQtkJ4RPbIT3xZfvcOIlw-lJoI7N4Cs7T6Ypq3W9lzaYlpqonMCjFf3iWN60L_sjUbCY-d4VDQMWHwrwhhRPql0JvAPuP36qvqksGSrY05VVM7ZiQ7-FHWZEg4FquohT8RHmoKEm-UM3Wt8xPKFhNTeuTnr2XLXept54ow5_wi4rju6d1oUqYU7rVNFvSWNYpnoIYZ4-drPDUzjSDBMubJDcoSuxGjxmQv58f706o_OcrC3_IZgfFq_IS3xbdZ3tad-8JljrahAYjopn8TENx4JiFntD-VKnLJPQi68ClcIth67TKQo8p5NxYHbpWdxveY4g1QmdU3c5eflDmjqgXwx9fmib88toc1DegAhkaIl7cFGiyQma2j4rLBYFi-BMGfVhDXakK4ze3x8Usqd1NhdGsmLcXrck7LlQvwWtNjSTeIHR8bMkia6N0h5N53_A6CgfJi2D6G2HaG1g3-U7mxK-YtJESl8HV4NG9ENhp3ClPPiRmBgzqMS_X59g5E-fxifSns","use":"sig"}]}
```

**About the `kid`.** Each key carries a `kid` (key ID). On IDP.to the `kid` is the [RFC 7638](https://www.rfc-editor.org/rfc/rfc7638) JWK thumbprint of the key: a deterministic hash computed from the key's own public parameters. Because it is derived from the key material itself, it is stable and unique to that key — you do not need to coordinate key names out of band. When your library verifies an ID token, it reads the `kid` from the token header and selects the matching key from this set.

**About the key array.** `keys` is an array, and it can hold more than one key at a time. This supports **rotation windows**: when a signing key is being rotated, both the outgoing and incoming keys are published together for an overlap period. Tokens signed with either key verify against the set during that window. This is why you should always select the key by `kid` rather than assuming a single key — a well-behaved client fetches the current JWKS, matches on `kid`, and re-fetches if it sees an unknown `kid`.

## What's coming and where it stands

The following are planned and tracked publicly. Status is as of the date above.

### Authorization endpoint — `/oidc/authorize`

The interactive endpoint that starts a login. It will implement the authorization-code flow with PKCE using `S256` code challenges, consistent with what discovery already advertises.

- Status: not yet implemented (returns `404`).
- Tracking: <https://github.com/synestheticsystems/idp/issues/19>

### Token endpoint — `/oidc/token`

The back-channel endpoint that exchanges an authorization code (plus the PKCE `code_verifier`) for tokens, including the ID token.

- Status: not yet implemented (returns `404`).
- Tracking: <https://github.com/synestheticsystems/idp/issues/20>

### Application / client registration

The mechanism to register an application and obtain credentials (`client_id`, and where applicable `client_secret`) and to configure allowed redirect URIs.

- Status: not yet available.
- Tracking: <https://github.com/synestheticsystems/idp/issues/21>

### ID token claims

Once the flow is live, the ID token will carry:

- `sub` — a permanent, merge-stable identifier for the person. It does not change if the underlying account records are merged, so it is safe to use as your primary user key.
- `email` — the person's email address.
- `name` — the person's display name.
- `org_id` — the organization the person belongs to. **Planned.**
- `identities[]` — the set of linked identities for the person. **Planned.**

`org_id` and `identities` already appear in `claims_supported` in discovery, but treat them as planned until the flow ships and they are documented as populated.

## Integration preview

You can wire up a standard authorization-code + PKCE flow now, pointed at the live discovery document, using any conformant OIDC client library. Nothing here is IDP.to-specific: you resolve the discovery URL, and the library configures itself from it.

The example below uses [`openid-client`](https://github.com/panva/node-openid-client) for Node.js.

> **This example will work once the endpoints ship.** The discovery document it resolves is already real and live. The authorization and token endpoints it drives are not implemented yet, and you will need a `client_id` from client registration, which is not open yet. Until then, the discovery step succeeds and the interactive steps do not.

```js
import * as client from 'openid-client';

// The discovery document is live today. This step already works.
const issuer = new URL('https://admin.idp.to');
const config = await client.discovery(
  issuer,
  'YOUR_CLIENT_ID', // from client registration — not available yet
);

const redirect_uri = 'https://your-app.example.com/callback';

// PKCE: generate a verifier and its S256 challenge.
const code_verifier = client.randomPKCECodeVerifier();
const code_challenge = await client.calculatePKCECodeChallenge(code_verifier);

// Build the authorization URL. Send the user here to sign in.
// (Endpoint not live yet — this will 404 until issue #19 ships.)
const authorizationUrl = client.buildAuthorizationUrl(config, {
  redirect_uri,
  scope: 'openid profile email',
  code_challenge,
  code_challenge_method: 'S256',
});

// ...redirect the user to authorizationUrl, then on your callback route,
// exchange the returned code for tokens (needs the token endpoint, issue #20):
//
// const tokens = await client.authorizationCodeGrant(config, currentUrl, {
//   pkceCodeVerifier: code_verifier,
// });
// const claims = tokens.claims(); // sub, email, name, ...
```

Store the `code_verifier` in the user's session between the redirect and the callback. Keep `redirect_uri` on the exact URL you will register once registration opens.

## End-user experience

When your application sends a user to IDP.to to sign in, they authenticate on IDP.to's hosted pages. Authentication is **passkey-based** (WebAuthn): the user signs in with a passkey rather than a password. Account **recovery is email-based**. There are no passwords anywhere in the flow.

You do not build or host any of this — the sign-in and recovery UI is served by IDP.to. Your application only handles the redirect out and the callback back, and reads the resulting ID token.

---

See also: [help site overview](../).
