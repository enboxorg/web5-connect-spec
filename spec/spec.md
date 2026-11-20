
Enbox Connect Protocol
==================

**Specification Status:** Draft

**Latest Draft:**
  https://web5-connect-spec.pages.dev
<!-- -->

**Editors:**
~ [Liran Cohen](https://www.linkedin.com/in/itsliran/) (Enbox)

**Participate:**
~ [GitHub repo](https://github.com/enboxorg/web5-connect-spec)
~ [File a bug](https://github.com/enboxorg/web5-connect-spec/issues)
~ [Commit history](https://github.com/enboxorg/web5-connect-spec/commits/main)

------------------------------------

## Abstract

The Enbox Connect Protocol (v2) defines a unified, transport-agnostic,
end-to-end encrypted handshake between an application ([[ref:Client]]) and a
user's wallet or identity agent ([[ref:Provider]]) for:

1. **Delegated DWN access (`connect`)** — configuring requested DWN modules,
   issuing scoped, time-bounded [[ref:Permission Grants]] for a
   [[ref:Delegate DID]] (wallet-minted or app-supplied), delivering module
   decryption keys (`grantKey` / `wrappedGrantKey`), and issuing per-grant
   self-revocation grants.
2. **Delegated grant refresh (`refresh`)** — renewing expiring grants for an
   existing local [[ref:Delegate DID]] pinned to a specific [[ref:Provider]] DID
   without minting a new identity.
3. **Owner authentication (`authenticate` or attached to `connect`)** —
   cryptographically proving control of the user's owner DID
   (`enbox-connect-owner-authentication+jwt`) bound to a relying-service HTTPS
   audience, single-use challenge, and canonical request digest.

Enbox Connect v2 removes all SIOPv2 and OpenID Connect payload framing. One
canonical signed-and-encrypted envelope format runs over two first-class
end-to-end encrypted transports: a stateless HTTP **Relay Transport** (with
fragment-only wallet URIs, QR codes, and HKDF PIN strengthening) and a
**Browser Popup `postMessage` Transport** (with ephemeral ECDH-ES and strict
origin/source pinning).

## Status of This Document

Enbox Connect Protocol is a _DRAFT_ specification under development by
[Enbox](https://enbox.org) (maintained in the `enboxorg/web5-connect-spec`
repository; earlier drafts were titled "Web5 Connect Protocol"). It is a
companion specification to the
[DWN specification](https://github.com/enboxorg/dwn-spec) and the
[DWN Transport Protocol specification](https://github.com/enboxorg/dwn-transport-spec),
and reflects the current implementation in
[enboxorg/enbox](https://github.com/enboxorg/enbox) (`@enbox/connect`,
`@enbox/dwn-server`, `@enbox/agent`, `@enbox/auth`, `@enbox/browser`,
`@enbox/cli`, `@enbox/crypto`, `@enbox/dwn-sdk-js`, and `@enbox/modules`).

The specification will be updated to incorporate feedback. We encourage reviewers
to submit [GitHub Issues](https://github.com/enboxorg/web5-connect-spec/issues)
as the means by which to communicate feedback and contributions.

The key words "****MUST****", "****MUST NOT****", "****REQUIRED****",
"****SHALL****", "****SHALL NOT****", "****SHOULD****", "****SHOULD NOT****",
"****RECOMMENDED****", "****MAY****", and "****OPTIONAL****" in this document
are to be interpreted as described in [[spec:RFC2119]].

## Terminology

[[def:Client, Application, Relying Party]]
~ The application requesting delegated DWN access and/or owner authentication
from a user's [[ref:Provider]].

[[def:Provider, Wallet, Identity Agent]]
~ The trusted software managing the user's owner DID(s), private keys, and DWN
access. Evaluates requests, obtains user consent, prepares modules, issues
grants and encryption keys, and signs owner authentication proofs.

[[def:Connect Server, Relay]]
~ A stateless store-and-forward HTTP intermediary exposing 7 endpoints under
`/connect/*` with a 600-second TTL. Sees only ciphertext blobs and opaque
session-progress flags.

[[def:Delegate DID]]
~ A DID (typically `did:jwk`) authorized via [[ref:Permission Grants]] to act
for a user session. May be minted by the [[ref:Provider]] (returning
`delegatePortableDid` with Ed25519 + derived X25519 keys) or pre-supplied by
the [[ref:Client]] (`request.delegateDid`, keeping delegate private keys local
to the app).

[[def:Permission Grant, Permission Grants]]
~ An immutable DWN record in the built-in Permissions Module at
`https://identity.foundation/dwn/permissions` authorizing a grantee DID for a
bounded scope and lifetime. See the
[DWN specification](https://github.com/enboxorg/dwn-spec).

[[def:Delegated Grant]]
~ A [[ref:Permission Grant]] with `delegated: true` allowing the grantee to sign
DWN messages on behalf of the grantor via `authorDelegatedGrant` (or
`ownerDelegatedGrant` on `RecordsWrite`).

[[def:ConnectSessionMetadata]]
~ Bounded session metadata (`id`, `createdAt`, `expiresAt`, optional display
hints, and `transport`: `"relay" | "postMessage"`) embedded in
`PermissionGrantData.connectSession` on each user-facing session grant.

[[def:SessionRevocation]]
~ A `{ grantId, revocationGrantId }` pair linking each user-facing session grant
to a companion `contextId`-scoped `Records.Write` grant on the Permissions
Module that lets the [[ref:Client]] self-revoke its session on disconnect.

[[def:Grant Key]]
~ A persisted decryption/sealing key record stored in the grantor's DWN under
the built-in Encryption Module (`https://identity.foundation/dwn/modules/encryption`)
as either an encrypted `grantKey` record (for wallet-minted delegates) or a
plaintext `wrappedGrantKey` envelope record (for pre-supplied delegates).

[[def:Owner Authentication Proof]]
~ A Compact JWS (`typ: "enbox-connect-owner-authentication+jwt"`) signed by an
active `authentication` verification method of the user's owner `providerDid`,
proving owner identity bound to an HTTPS `audience`, single-use `challenge`, and
the SHA-256 digest of the canonical `ConnectRequest`.

[[def:Wallet URI]]
~ A URI constructed by the [[ref:Client]] for the Relay Transport carrying
`request_uri` and the 32-byte symmetric `encryption_key` strictly in the URI
fragment (`#`), presented as a QR code or clickable deep link.

[[def:PIN]]
~ A short numeric code (**4 zero-padded ASCII decimal digits** in standard
browser UI for interoperability with `@enbox/browser`'s hardcoded 4-digit modal;
`CryptoUtils.randomPin` supports 3–10 digits and the HKDF wire binding accepts
any UTF-8 string) generated by the [[ref:Provider]] on the Relay Transport and
entered by the user into the [[ref:Client]]. The PIN strengthens the response
Content Encryption Key (CEK) via HKDF-SHA-256 (`info = "enbox/connect/v2/pin"`),
providing both confidentiality and integrity protection.

[[def:DENIED Token]]
~ The literal string `"DENIED"` (`CONNECT_DENIED_TOKEN`) transmitted in place of
a sealed response JWE when the user rejects the connect request.

## Overview

### Actors

1. **Client (Application / Relying Party)** — Requests delegated DWN module
   permissions, grant refreshes, and/or owner authentication proofs.
2. **Provider (Wallet)** — Holds the user's owner DID(s) and root keys.
   Executes the 5-phase approval ceremony or authentication-only flow.
3. **Connect Server (Relay)** — Optional untrusted intermediary used on the
   Relay Transport. Stores single-use encrypted blobs and observational
   claimed/completed markers for 600 seconds. Not used on the Browser Popup
   `postMessage` transport.

### Handshake Modes (`ConnectRequestType`)

| `requestType` | Purpose | Key Constraints |
|---|---|---|
| Omitted or `"connect"` | Initial delegated connection (optionally accompanied by an [[ref:Owner Authentication Proof]]) | Returns `ConnectResponse` with `delegateDid`, `delegateGrants`, `sessionRevocations`, optional `delegatePortableDid`, and optional `authenticationProof`. |
| `"refresh"` | Renew expiring grants for an existing local delegate (optionally accompanied by an [[ref:Owner Authentication Proof]]) | `delegateDid` is ****REQUIRED****; `expectedProviderDid` is optional on the wire (set by `AuthManager.refresh` to pin the owner DID). Provider ****MUST NOT**** mint a new delegate DID or return `delegatePortableDid`. |
| `"authenticate"` | Owner authentication without delegation | `permissionRequests` ****MUST**** be `[]`; `delegateDid` and `requestedSessionTtlSeconds` ****MUST**** be omitted; `authentication` is ****REQUIRED****. Returns `ConnectAuthenticationResponse` (`responseType: "authentication"`), never creates an `AgentSession` or grants. (Supported by `ConnectClient`, `WalletConnect`, and popup flows; `@enbox/browser`'s `runRelayConnect` / `BrowserConnectHandler` modal relay runner does not support `authenticate`.) |

### Protocol Flows

**Relay Transport (Cross-Device QR / Same-Device Tab / Deep Link):**
1. Client generates ephemeral Ed25519 `did:jwk` (`clientDid`), fresh X25519 key
   pair (`responseKey`), random `nonce` & `state` (16 bytes base64url each), and
   32-byte symmetric `requestKey`.
2. Client signs `ConnectRequest` as a `did:jwk` EdDSA JWT, seals it with Compact
   JWE (`alg: "dir"`, `enc: "XC20P"`, `typ: "enbox-connect-req"`), and pushes it
   to `POST /connect/par`, receiving `{ request_uri, expires_in: 600 }`.
3. Client presents the fragment-only [[ref:Wallet URI]]
   `{walletUri}#request_uri=...&encryption_key=...` (as QR code or link) and
   polls `GET /connect/token/:state.jwt` (`204` while pending) and optionally
   `GET /connect/status/:requestId` (`{ claimed }`).
4. Provider parses fragment parameters, fetches `GET /connect/authorize/:requestId.jwt`
   (single-use; relay records `claimed`), opens JWE, verifies `did:jwk` JWT,
   obtains consent, runs the approval or authentication ceremony, generates a
   [[ref:PIN]], seals `ConnectResponse` / `ConnectAuthenticationResponse` with
   `alg: "ECDH-ES"` + `enc: "XC20P"` + HKDF-SHA-256 PIN strengthening, and posts
   `id_token` & `state` to `POST /connect/callback` (or `id_token=DENIED`).
5. Client retrieves the sealed response from `GET /connect/token/:state.jwt`,
   prompts for the PIN (`ConnectClient` performs a single attempt; browser
   `runRelayConnect` allows up to 6 in-memory retries), decrypts & verifies the
   response (and any owner authentication proof), immediately sends a
   best-effort `POST /connect/complete` beacon (which the wallet may observe via
   `GET /connect/complete/:state` to confirm response receipt), and then
   performs downstream grant validation (`validateConnectResultGrants`) and
   two-phase local DWN import.

**Browser Popup `postMessage` Transport (Same-Browser Window):**
1. Client synchronously opens `https://<wallet-origin>/dweb-connect` inside the
   user click gesture.
2. Wallet generates an ephemeral X25519 key pair (`walletEpk`) and posts
   `{ type: "enbox-connect-loaded", walletEpk, walletOrigin }` to `window.opener`
   pinned to `dappOrigin`.
3. Client verifies `event.origin`, `event.source`, `walletEpk`, and
   `walletOrigin`, seals `ConnectRequest` using `alg: "ECDH-ES"` to `walletEpk`
   with `apv = base64url(UTF8(walletOrigin))`, and posts
   `{ type: "enbox-connect-request", jwe }` pinned to `walletOrigin`.
4. Wallet verifies `event.origin`, `event.source`, `apv`, and
   `reply.mode === "post_message"`, runs approval/authentication using
   `verifiedRequesterOrigin = dappOrigin` (no PIN required), seals the response
   using `alg: "ECDH-ES"` to `request.responseKey` with `apu = base64url(UTF8(state))`,
   and posts `{ type: "enbox-connect-response", payload }`.
5. Client decrypts & verifies the response (and any owner authentication proof),
   immediately sends `{ type: "enbox-connect-ack" }` so the wallet can close
   cleanly, and then performs downstream grant validation and two-phase local
   DWN import.

## Connect Server (Relay) Endpoints and Discovery

All relay state uses a **600-second** (`10`-minute) TTL across 4 key namespaces
(`request:${requestId}`, `claimed:${requestId}`, `response:${state}`,
`complete:${state}`), stored in `cacheEntries` (SQL `varchar(512)` primary key),
with durability awaits before returning `201` and single-use delete awaits
before returning `200` on `/authorize` and `/token`.

::: note
**Relay implementation notes (`dwn-server`)**:
- `ConnectServer` performs separate non-transactional `SELECT` → `DELETE` (and
  `INSERT` for `claimed:`) queries via `SqlTtlCache`, so single-use deletion is
  not transactional/atomic under concurrent requests; `SqlTtlCache.insert` uses
  a bare `INSERT` with no upsert, so duplicate `POST /connect/callback` or
  `POST /connect/complete` calls for an existing `state` currently return HTTP
  `500`. Production relays ****SHOULD**** enforce atomic single-use consumption
  and handle duplicate keys explicitly.
- `dwn-server` does not currently emit `Cache-Control: no-store` or
  `X-Content-Type-Options: nosniff` headers or enforce dedicated `/connect/*`
  body-size limits, and only `GET /connect/complete/:state` URL-decodes its path
  segment (`decodeURIComponent`). Relays ****SHOULD**** return
  `Cache-Control: no-store` and reject oversized payloads or malformed
  `requestId`/`state` values.
:::

### 1. Pushed Authorization Request (`POST /connect/par`)

```http
POST /connect/par HTTP/1.1
Content-Type: application/json

{ "request": "<Compact JWE string>" }
```

- **201 Created** (`application/json`):
  ```json
  {
    "request_uri": "https://dwn.example/connect/authorize/<uuid-v4>.jwt",
    "expires_in": 600
  }
  ```
- **400 Bad Request**:
  - Missing `request`:
    ```json
    { "ok": false, "status": { "code": 400, "message": "Bad Request: Missing 'request' parameter" } }
    ```
  - Nested `request.request_uri` present:
    ```json
    { "ok": false, "status": { "code": 400, "message": "Bad Request: 'request_uri' parameter is not allowed in PAR" } }
    ```
  (Note: malformed JSON bodies on `POST /connect/par` and JSON `POST /connect/callback`
  currently throw before error translation and return HTTP `500` in `dwn-server`;
  `POST /connect/complete` catches JSON parse errors and returns `400`.)

### 2. Request Retrieval (`GET /connect/authorize/:requestId.jwt`)

- **Side effect on hit**: Deletes `request:${requestId}` (single-use; relays
  ****SHOULD**** perform this atomically) and records `claimed:${requestId}`
  (`{ claimedAt }`, TTL 600s).
- **200 OK** (`Content-Type: application/jwt`): Raw stored Compact JWE string.
- **404 Not Found**:
  ```json
  { "ok": false, "status": { "code": 404, "message": "Not Found" } }
  ```

### 3. Claimed Status (`GET /connect/status/:requestId`)

Non-consuming observational endpoint allowing the client UI to show that the
wallet has scanned/fetched the request and stop auto-reminting QR pointers.
(Clients derive `requestId` from `request_uri` matching
`/\/connect\/authorize\/([^/]+)\.jwt$/` and resolve `status/${requestId}`
against `connectServerUrl`.)

- **200 OK** (`application/json`, always `200` even for unknown/expired IDs):
  ```json
  { "claimed": true }
  ```
  or `{ "claimed": false }` when unclaimed, unknown, or expired.

### 4. Callback Submission (`POST /connect/callback`)

Accepts either `application/x-www-form-urlencoded` (sent by `postRelayResponse`:
`id_token=<JWE-or-DENIED>&state=<state>`) or `application/json`
(`{ "id_token": "...", "state": "..." }`).

- **201 Created** (`application/json`):
  ```json
  { "ok": true, "status": { "code": 201, "message": "Created" } }
  ```
- **400 Bad Request** (missing `id_token` or `state`):
  ```json
  { "ok": false, "status": { "code": 400, "message": "Bad Request" } }
  ```
  (Note: current `dwn-server` checks `idToken !== undefined && state != undefined`,
  so empty strings are stored as `""` and later treated as `204` by `/token`;
  relays ****SHOULD**** reject empty/null `id_token` or `state` with `400`.)

### 5. Token Retrieval (`GET /connect/token/:state.jwt`)

- **When ready**: Deletes `response:${state}` (single-use; relays ****SHOULD****
  perform this atomically) and returns **`200 OK`** (`Content-Type: application/jwt`)
  with the raw sealed JWE string or literal `"DENIED"`.
- **When pending, expired, or already consumed**: Returns
  **`204 No Content`** with an empty body (never `404`). `RelayClientTransport.pollToken`
  treats any non-2xx HTTP status (including `404`, `429`, `5xx`) or empty 2xx
  body as pending and continues polling until `timeoutMs`.

### 6. Completion Beacon (`POST /connect/complete`)

Best-effort client signal (`keepalive: true`) sent immediately after
successfully opening and verifying the response envelope (before downstream
grant validation and local DWN import).

- **Request** (`application/json`): `{ "state": "<state>" }`
- **201 Created**: `{ "ok": true, "status": { "code": 201, "message": "Created" } }`
- **400 Bad Request**: `{ "ok": false, "status": { "code": 400, "message": "Bad Request: Missing 'state' parameter" } }`

### 7. Completion Poll (`GET /connect/complete/:state`)

Non-consuming observational endpoint polled by the wallet (`pollRelayComplete`,
resolved relative to `callbackUrl` as `complete/${encodeURIComponent(state)}`,
default 1s interval, 60s timeout; short-circuits to `false` on HTTP `404`) to
transition its pairing UI to "connected".

- **200 OK** (`application/json`, always `200` even for unknown/expired states):
  ```json
  { "completed": true }
  ```
  or `{ "completed": false }` when not yet completed, unknown, or expired.

### CORS, Rate Limiting, and Local-Node Isolation

- Standard public relay routes return CORS headers:
  `Access-Control-Allow-Origin: *`, `Access-Control-Allow-Methods: GET, POST, OPTIONS`,
  `Access-Control-Allow-Headers: *`, `Access-Control-Expose-Headers: dwn-response`,
  `Access-Control-Max-Age: 86400`; `OPTIONS` returns `204`.
- Server-level per-IP rate limiting applies (default `30 req/s`, burst `50`,
  evaluated prior to `OPTIONS` preflight handling; returns `429 Too Many Requests`,
  `Retry-After`, `{"error":"Rate limit exceeded"}`).
- When the host `dwn-server` runs under the Local Node Profile, `/connect/*`
  routes are protected by local-node bearer authentication and origin pairing.

### Wallet and Relay Discovery

1. **Well-Known Wallet Document (`GET https://<wallet-origin>/.well-known/enbox-connect`)**:
   Advertises the wallet's relay endpoint (`Content-Type: application/json`;
   browser deployments require CORS `Access-Control-Allow-Origin: *`, and
   production endpoints ****SHOULD**** use `https:` — `http:` only for local
   development/loopback):
   ```json
   {
     "connectServerUrl": "https://dwn.example/connect"
   }
   ```
   (`@enbox/browser` fetches with a 6-second timeout and caches results per modal
   session; `@enbox/cli` uses a 5-second timeout and validates/normalizes to
   `http:`/`https:`.)
2. **Connect DWN Module (`https://identity.foundation/modules/connect`)**:
   A published DWN module (`@enbox/modules`) allowing a DID to advertise its
   associated web wallets (`published: true`, type `wallet` with
   `schema: "https://identity.foundation/schemas/connect/wallet"`,
   `dataFormats: ["application/json"]`, `$recordLimit: { "max": 1 }`,
   `who: "anyone", can: ["read"]`, payload
   `{ "webWallets": ["https://wallet.example"] }`).
3. **Note on `dwn://connect`**: `dwn://connect?callback=<url>` is a separate
   local-DWN endpoint discovery scheme for desktop apps (see the DWN Transport
   Protocol specification), not an Enbox Connect wallet URI.

## Wire Request and Response Objects

### `ConnectRequest`

```json
{
  "clientDid": "did:jwk:eyJrdHkiOiJPS1AiLCJjcnYiOiJFZDI1NTE5IiwieCI6Ii4uLiJ9",
  "appName": "Notes",
  "appIcon": "https://notes.example/icon.png",
  "applicationId": "https://notes.example",
  "clientMetadata": {
    "origin": "https://notes.example",
    "userAgent": "Mozilla/5.0 ...",
    "platform": "macOS",
    "language": "en-US",
    "languages": ["en-US", "en"],
    "timezone": "America/New_York"
  },
  "authentication": {
    "version": 1,
    "audience": "https://notes.example",
    "challenge": "<base64url-22-to-128-chars>"
  },
  "permissionRequests": [
    {
      "moduleDefinition": {
        "module": "https://notes.example/modules/notes/v1",
        "published": true,
        "types": { ... },
        "structure": { ... }
      },
      "permissionScopes": [
        { "interface": "Modules", "method": "Query", "module": "https://notes.example/modules/notes/v1" },
        { "interface": "Messages", "method": "Read", "module": "https://notes.example/modules/notes/v1" },
        { "interface": "Records", "method": "Read", "module": "https://notes.example/modules/notes/v1" },
        { "interface": "Records", "method": "Write", "module": "https://notes.example/modules/notes/v1" },
        { "interface": "Records", "method": "Delete", "module": "https://notes.example/modules/notes/v1" }
      ]
    }
  ],
  "requestedSessionTtlSeconds": 86400,
  "delegateDid": "did:jwk:...",
  "requestType": "connect",
  "expectedProviderDid": "did:dht:...",
  "supportedDidMethods": ["did:dht", "did:jwk"],
  "nonce": "<16-random-bytes-base64url>",
  "state": "<16-random-bytes-base64url>",
  "responseKey": { "kty": "OKP", "crv": "X25519", "x": "<base64url>" },
  "reply": { "mode": "direct_post", "callbackUrl": "https://dwn.example/connect/callback" }
}
```

- `clientDid` — ****REQUIRED****. Ephemeral `did:jwk` (Ed25519); ****MUST**** equal the inner JWT signer DID. When `authentication` is present, must parse via `Did.parse` with `method === "jwk"` and no path, query, or fragment.
- `appName` — ****REQUIRED**** (`1..256` chars when `authentication` is present).
- `appIcon`, `applicationId`, `clientMetadata` — ****OPTIONAL**** self-reported display hints.
- `authentication` — ****OPTIONAL**** (REQUIRED when `requestType === "authenticate"`; also permitted on `"connect"` and `"refresh"`). Exact keys `["version", "audience", "challenge"]`: `version === 1`, `audience` is a canonical `https:` URL (`audience === parsed.href || audience === parsed.origin`, `<= 2048` chars, no userinfo/query/hash), `challenge` matches `/^[A-Za-z0-9_-]{22,128}$/`.
- `permissionRequests` — ****REQUIRED**** array of `{ moduleDefinition, permissionScopes }` (strictly 2 keys; `moduleDefinition` validated with `requireKeyAgreement: false`, each scope validated against `PermissionScope` JSON schema; must be `[]` when `requestType === "authenticate"`, max 32 entries when `authentication` is present).
- `requestedSessionTtlSeconds` — ****OPTIONAL**** advisory session lifetime in seconds (forbidden for `authenticate`; must be finite and floor to `>= 1` whole second).
- `delegateDid` — ****OPTIONAL**** pre-supplied delegate DID URI (REQUIRED for `refresh`, forbidden for `authenticate`).
- `requestType` — ****OPTIONAL**** (`"connect" | "refresh" | "authenticate"`).
- `expectedProviderDid` — ****OPTIONAL**** owner DID pin (set by `AuthManager.refresh` on refresh/reconnect, though optional on the wire).
- `supportedDidMethods` — ****REQUIRED**** `string[]` (default `["did:dht", "did:jwk"]`; when `authentication` is present, `1..8` unique entries each matching `/^did:[a-z0-9]+$/`).
- `nonce`, `state` — ****REQUIRED**** 16-byte cryptographically random `base64url` strings (validated against `/^[A-Za-z0-9_-]{22,128}$/` when `authentication` is present).
- `responseKey` — ****REQUIRED**** fresh public-only X25519 JWK (`{ kty: "OKP", crv: "X25519", x }`, no `d`).
- `reply` — ****REQUIRED****: `{ mode: "direct_post", callbackUrl }` (relay; `callbackUrl` `1..2048` chars when `authentication` is present) or `{ mode: "post_message" }` (popup).

### `ConnectResponse` (Delegation Response)

MUST NOT include `responseType`:

```json
{
  "providerDid": "did:dht:...",
  "delegateDid": "did:jwk:...",
  "aud": "<request.clientDid>",
  "iat": 1700000000,
  "exp": 1700000600,
  "nonce": "<request.nonce>",
  "state": "<request.state>",
  "delegateGrants": [ ... ],
  "delegatePortableDid": { ... },
  "sessionRevocations": [
    { "grantId": "<session-grant-recordId>", "revocationGrantId": "<revocation-grant-recordId>" }
  ],
  "authenticationProof": "<optional-owner-auth-compact-jws>"
}
```

- `exp` = `iat + 600` (`CONNECT_RESPONSE_TTL_SECONDS = 600`).
- `delegatePortableDid` is present **iff** the wallet minted the delegate DID (containing both its Ed25519 private key and derived X25519 private key); it is **omitted** when `request.delegateDid` was pre-supplied.
- `delegateGrants` contains all data-encoded `RecordsWrite` messages for both session grants and companion self-revocation grants.

### `ConnectAuthenticationResponse` (Authentication-Only Response)

Enforces strictly these 8 keys (delegation fields are forbidden):

```json
{
  "responseType": "authentication",
  "providerDid": "did:dht:...",
  "aud": "<request.clientDid>",
  "iat": 1700000000,
  "exp": 1700000600,
  "nonce": "<request.nonce>",
  "state": "<request.state>",
  "authenticationProof": "<owner-auth-compact-jws>"
}
```

## Cryptography

### Inner JWT Signing (`did:jwk`-Pinned EdDSA)

Both `ConnectRequest` and `ConnectResponse` / `ConnectAuthenticationResponse`
are first signed as Compact JWS tokens (`packages/connect/src/jwt.ts`, using
standard `JSON.stringify`):
- Protected header: `{ "alg": "EdDSA", "kid": "<did:jwk:...>#0", "typ": "JWT" }`.
- Signature: Ed25519 over ASCII `b64u(header) + "." + b64u(payload)`.
- **Verification pinning**: `verifyJwt()` requires `alg === "EdDSA"`,
  `kid` beginning with **`did:jwk:`**, and resolved `method.id === header.kid`
  (pure local key decoding, zero network I/O; `verifyJwt` does not enforce
  `header.typ`). Request opening enforces `signerDid === payload.clientDid`.
  Response signing uses the minted delegate `did:jwk` or a fresh ephemeral
  `did:jwk` (for pre-supplied delegates and authentication-only responses).

### Outer Compact JWE Envelopes (`XC20P` + Mandatory Allow-Lists)

All Enbox Connect envelopes use standard 5-part [[spec:RFC7516]] Compact JWE
with content encryption **`enc: "XC20P"`** (XChaCha20-Poly1305, 256-bit CEK,
24-byte random IV, 16-byte auth tag, empty encrypted-key segment `""`, AAD =
ASCII `b64u(protectedHeader)`).

Decryption ****MUST**** enforce explicit algorithm allow-lists before key
management executes:
- Relay request: `allowedAlgs: ["dir"]`, `allowedEncs: ["XC20P"]`.
- Popup request: `allowedAlgs: ["ECDH-ES"]`, `allowedEncs: ["XC20P"]`.
- All responses: `allowedAlgs: ["ECDH-ES"]`, `allowedEncs: ["XC20P"]`.

#### Request Sealing

1. **Relay Transport (`mode: "dir"`)**:
   - Protected header: `{ "alg": "dir", "cty": "JWT", "enc": "XC20P", "typ": "enbox-connect-req" }`.
   - Encrypted directly with the random 32-byte `requestKey` carried only in the
     [[ref:Wallet URI]] fragment.
2. **Popup Transport (`mode: "ecdh-es"`)**:
   - Protected header:
     ```json
     {
       "alg": "ECDH-ES",
       "apv": "<base64url(UTF8(walletOrigin))>",
       "cty": "JWT",
       "enc": "XC20P",
       "typ": "enbox-connect-req",
       "epk": { "kty": "OKP", "crv": "X25519", "x": "<base64url>" }
     }
     ```
   - Computes X25519 shared secret `Z` to `walletEpk` (rejecting all-zero
     low-order outputs and same-keypair inputs), then derives CEK via
     [[spec:RFC7518]] §4.6.2 Concat KDF SHA-256 with empty `PartyUInfo`
     (`uint32be(0)`) and `PartyVInfo = uint32be(len(UTF8(walletOrigin))) || UTF8(walletOrigin)`.
     Wallet verifies `typ`, `cty`, and `apv === base64url(UTF8(walletOrigin))`.

#### Response Sealing (`ECDH-ES` + HKDF-SHA-256 PIN Strengthening)

Responses use dedicated X25519 keys (`epk` generated fresh by the wallet,
paired with `request.responseKey` — **no** `kid` header and **no**
Ed25519→X25519 envelope conversion):

1. **Protected Header**:
   ```json
   {
     "alg": "ECDH-ES",
     "apu": "<base64url(UTF8(request.state))>",
     "cty": "JWT",
     "enc": "XC20P",
     "typ": "enbox-connect-res",
     "epk": { "kty": "OKP", "crv": "X25519", "x": "<base64url>" }
   }
   ```
2. **X25519 Key Agreement with Low-Order Point Rejection**:
   Compute `Z = X25519(ephemeralPrivate, request.responseKey)` (or
   `X25519(responsePrivateKey, epk)` on decrypt). Same-keypair inputs and any
   all-zero output `Z` ****MUST**** fail immediately.
3. **Concat KDF ([[spec:RFC7518]] §4.6.2, SHA-256)**:
   ```
   cek = SHA-256(
     0x00000001 || Z ||
     (uint32be(5) || UTF8("XC20P")) ||
     (uint32be(len(UTF8(state))) || UTF8(state)) ||
     uint32be(0) ||
     uint32be(256)
   )
   ```
4. **PIN CEK Strengthening (Relay Transport only — NOT AAD)**:
   When a [[ref:PIN]] is present, strengthen `cek` via [[spec:RFC5869]]
   HKDF-SHA-256:
   ```
   cek = HKDF-SHA-256(
     ikm    = cek,
     salt   = UTF8(pin),
     info   = UTF8("enbox/connect/v2/pin"),
     length = 256 bits (32 bytes)
   )
   ```
   The PIN is never placed in AAD; an incorrect PIN derives a completely
   different 256-bit CEK and fails AEAD tag verification.
5. **Context Verification**: On open, the client verifies `typ === "enbox-connect-res"`,
   `cty === "JWT"`, `apu === base64url(UTF8(expected.state))`, inner `did:jwk`
   EdDSA JWT, `payload.aud === expected.clientDid`, `payload.nonce === expected.nonce`,
   `payload.state === expected.state`, optional `expectedProviderDid`,
   `payload.exp > payload.iat`, `payload.iat <= now + 60`, and `now < payload.exp`.
   (Note: `assertConnectResponse` does not enforce `exp - iat <= 600` on open.)

## Owner Authentication Proofs

When `request.authentication` is present (in `connect`, `refresh`, or
`authenticate` mode), the wallet produces an [[ref:Owner Authentication Proof]]
proving control of the user's owner DID (`providerDid`) bound to the exact
untouched `ConnectRequest`.

### Proof Compact JWS Format

- **Protected Header** (exact 3 keys `["alg", "kid", "typ"]`, canonical JSON):
  ```json
  {
    "alg": "EdDSA",
    "kid": "<providerDid>#<fragment>",
    "typ": "enbox-connect-owner-authentication+jwt"
  }
  ```
  Supported `(alg, curve, signature length)` tuples: `EdDSA` (`OKP`/`Ed25519`, 64 bytes), `ES256K` (`EC`/`secp256k1`, raw 64-byte `R || S`), `ES256` (`EC`/`P-256`, raw 64-byte `R || S`).
- **Payload** (exact 8 keys, canonical JSON):
  ```json
  {
    "iss": "<providerDid>",
    "sub": "<providerDid>",
    "aud": "<request.authentication.audience>",
    "nonce": "<request.authentication.challenge>",
    "request_hash": "<base64url(SHA-256(UTF8(canonical(originalRequest))))>",
    "purpose": "enbox-connect-authentication",
    "iat": 1700000000,
    "exp": 1700000300
  }
  ```
- **Canonical JSON & Constants**: `exp = iat + 300` (max age `300s`), clock skew
  `30s`, max canonical request `262,144` bytes (256 KiB, max depth `32`), max
  proof `16,384` bytes (16 KiB). `canonical()` produces compact JSON (no
  whitespace) using ES2019 well-formed `JSON.stringify` for strings/keys
  (escaping lone UTF-16 surrogates prior to UTF-8 encoding), finite numbers only
  (rejecting `NaN`/`Infinity`), plain objects only (omitting `undefined`
  properties and sorting keys by UTF-16 code units), and rejecting `undefined`
  array elements or non-plain objects (`Date`, `RegExp`, `Map`, `Set`).

### Wallet and Verifier Invariants

- **Wallet invariants**:
  - Requires `verifiedRequesterOrigin` (from popup `event.origin` or
    service-owned relay policy; ****MUST**** be `https:`, even in local
    development) matching `new URL(request.authentication.audience).origin`.
  - `providerDid` ****MUST**** be a locally owned identity (delegates rejected).
  - Signing key ****MUST**** be uniquely listed in `didDocument.authentication`
    (both `authentication` and `verificationMethod` capped at 64 entries;
    relative `#fragment` references are expanded to `<providerDid>#<fragment>`)
    with `controller === providerDid` and signed via KMS without exporting owner
    private keys.
  - Key selection & validation are prepared before side effects, while signing
    is deferred until the end of the approval ceremony so setup time does not
    consume the 300-second proof TTL.
- **Verifier invariants (`verifyOwnerAuthenticationProof`)**:
  - Fixed opaque error codes: `invalid-request`, `invalid-proof`,
    `unsupported-key`, `invalid-signature`, `context-mismatch`, `expired-proof`,
    `did-resolution`, `replayed-proof`.
  - Requires all 3 Compact JWS segments to match unpadded `/^[A-Za-z0-9_-]+$/`,
    `header.kid <= 2048` chars matching `<providerDid>#<non-empty-fragment>`,
    public-only `publicKeyJwk` (with `publicKeyJwk.alg` matching the signing
    algorithm if present), and `providerDid` syntax (no path/query/fragment,
    `<= 2048` chars, `!== clientDid`, `!== delegateDid`, in
    `supportedDidMethods`, matches `expectedProviderDid` if set).
  - Resolves `providerDid` freshly (rejecting deactivated DIDs) and checks the
    300s/30s time window **three times** (before DID resolution, after signature
    verification before replay consumption, and after replay consumption).
  - Supports an optional `replayStore` (`ConnectAuthenticationReplayStore`):
    browser-side `ConnectClient` invokes `verifyOwnerAuthenticationProof`
    **without** `replayStore` (producing cryptographic verification only).
    Relying services ****MUST**** perform durable, atomic transaction-keyed
    `(audience, challenge)` consumption server-side (either by passing
    `replayStore` to `verifyOwnerAuthenticationProof` or atomically consuming
    the challenge when issuing an account session) before accepting
    authentication.
  - `ConnectClient` invokes `onAuthenticationRequest(clone(request))` **before**
    transmitting the request so the relying service durably binds the exact
    request; omitting a requested proof or returning an unsolicited proof fails
    closed.

## Provider Processing (5-Phase Approval Ceremony)

For `connect` and `refresh` requests, the wallet runs preflight validation and
then executes 5 progress phases (`delegate`, `modules`, `permission-grants`,
`grant-keys`, `revocations`) identically across transports (with a 10-second
`CONNECT_DID_RESOLUTION_TIMEOUT_MS` on DID resolution and fail-closed rejection
if zero remote DWN endpoints resolve or respond):

**Preflight & Auth Preparation**: Validates `permissionRequests` (schema
validation, every scope's `scope.module === moduleDefinition.module`, no
duplicate or non-normalized module URIs, finite TTL flooring to `>= 1` second,
rejecting `Records.Query`/`Subscribe`/`Count`), `expectedProviderDid`, and
`verifiedRequesterOrigin`; prepares owner authentication key if requested.
(Note: in current `enbox`, `Modules.Configure` passes preflight schema
validation and is rejected by `assertConnectGrantScope` during Phase 3
`permission-grants`; implementations ****SHOULD**** reject `Modules.Configure`
in preflight prior to any DWN mutations.)

1. **Delegate Resolution (`delegate`)**:
   - *Wallet-minted (`request.delegateDid` absent)*: Mints a fresh Ed25519
     `did:jwk`, derives and appends its X25519 private key to
     `delegatePortableDid.privateKeys` (required for DWN encryption), and uses
     it as `responseSigner`.
   - *Pre-supplied (`request.delegateDid` present)*: Validates DID URI syntax,
     omits `delegatePortableDid`, mints an ephemeral `did:jwk` solely as
     `responseSigner`, and resolves the pre-supplied delegate's root X25519
     public key whenever any requested module requires key agreement and
     includes a `Records.Read` scope.
2. **Module Preparation (`modules`)**:
   - Rejects requester-supplied `$keyAgreement` / `$encryption` metadata (keys
     are always wallet-derived).
   - Resolves `uses` dependency composition plan (including zero-scope
     `installationOnly` dependencies and depth-first propagation of locally
     installed out-of-batch `uses` dependencies missing on remotes), verifies
     installed `$keyAgreement` public keys by JWK thumbprint (rejecting foreign
     keys as non-overridable conflicts, and rejecting conflicting authored
     definitions unless explicitly owner-approved via `approvedModuleOverrides`),
     configures modules in topological `uses`-dependency order locally and
     across remote endpoints (concurrency 8, 10s per-request timeout), and
     **fail-closed re-queries** changed endpoints to verify convergence.
3. **Permission Grants (`permission-grants`)**:
   - Rejects `Modules.Configure` scopes (`assertConnectGrantScope`).
   - Builds `ConnectSessionMetadata` (required `id` <= 128, `createdAt`,
     `expiresAt`; optional `applicationId` <= 512, `appName` <= 128,
     `appIcon` <= 2048, `origin` <= 512, `userAgent` <= 512, `platform` <= 128,
     `language` <= 64, `languages` <= 16 items of <= 64 chars, `timezone` <= 128,
     `transport`: `"relay" | "postMessage"`; clamped to `PermissionGrantData`
     schema limits, with `origin` overridden by `verifiedRequesterOrigin` on any
     transport supplying it).
   - Computes grant TTL: default **`3600` seconds (1 hour)**, hard maximum
     **`7,776,000` seconds (90 days)**; provider override takes precedence over
     advisory `requestedSessionTtlSeconds`.
   - Creates grants locally and fans out to remote endpoints (10s per request,
     20s batch budget); **fail-closed** — every grant must be accepted (`202` or
     `409`) by at least one remote endpoint.
4. **Encryption Grant Keys (`grant-keys`)**:
   - Module gate: triggered whenever a module requires key agreement
     (`Modules.requiresKeyAgreement`) and the request includes at least one
     `Records.Read` scope for that module (a module requesting only
     `Records.Write` without `Records.Read` does not trigger connect grant-key
     issuance).
   - Once triggered, `createGrantKeyRecordsForGrants` writes immutable records
     to the owner's DWN under `https://identity.foundation/dwn/modules/encryption`
     for every eligible **non-`contextId`-scoped** `Records.Read` grant (module/path
     keys plus local read-referenced `$role` paths) **and** `Records.Write` grant
     (covered local `$role` paths carrying `$keyAgreement`):
     - Wallet-minted delegate → DWN-encrypted `grantKey` record (encrypted to
       grantee's derived `["modulePath", "https://identity.foundation/dwn/modules/encryption", "grantKey"]` key).
     - Pre-supplied delegate → plaintext `wrappedGrantKey` record carrying an
       `enbox/wrapped-grant-key@1` envelope wrapped to the delegate's root
       X25519 public key.
   - Fail-closed fan-out: every record must be accepted (`202` or `409`) by at
     least one remote endpoint.
5. **Per-Grant Self-Revocation Grants (`revocations`)**:
   - For each user-facing session grant, creates a companion `delegated: true`
     grant (same `dateExpires`, no `connectSession` metadata) with scope:
     ```json
     {
       "interface": "Records",
       "method": "Write",
       "module": "https://identity.foundation/dwn/permissions",
       "contextId": "<session-grant-recordId>"
     }
     ```
   - Best-effort remote fan-out; included in `delegateGrants` and paired in
     `sessionRevocations`.

## Client Processing

1. **Grant Validation (`validateConnectResultGrants`)**: Before importing,
   verifies every grant's `grantor === providerDid` and `grantee === delegateDid`,
   verifies every non-revocation grant's scope is a subset of a requested scope,
   verifies revocation grants match `Records.Write` on
   `https://identity.foundation/dwn/permissions` with `contextId === grantId`,
   and requires at least one non-revocation grant whenever `permissionRequests`
   is non-empty (consequently, a request containing *only* `installationOnly`
   modules fails client validation unless accompanied by at least one module
   requesting scopes).
2. **Two-Phase Local Grant Import**:
   - Phase 1 (rollbackable): writes all grants to the delegate's local DWN
     partition (`signAsOwner: true`), rolling back newly created (`202`) records
     on any failure.
   - Phase 2: writes all grants to `providerDid`'s local DWN partition via
     `processRawMessage`.
   - Sync scope (`deriveSyncScopeFromGrants`) is derived from non-expired
     `Messages.Read` grants: an unscoped `Messages.Read` grant maps to `'all'`,
     while module-wide `Messages.Read` grants collect their module URIs
     (excluding `PermissionsModule` and path/context-scoped grants).
3. **Session Status, Refresh, and Self-Revocation**:
   - Connection status classifies sessions as `active`, `expiring-soon`
     (default threshold `min(3600s, 10% of lifetime)`), `expired`, `revoked`, or
     `none`, using the earliest `grant.dateExpires` (never display
     `connectSession.expiresAt`).
   - `AuthManager.refresh` sends `requestType: "refresh"` with the existing
     `delegateDid` and `expectedProviderDid`, imports renewed grants without
     replacing the delegate identity, and remembers per-`(delegateDid, providerDid)`
     wallet routes.
   - On `disconnect()`, the app first proactively ensures each companion
     revocation grant exists on remote owner DWN endpoints
     (`ensureRevocationGrantOnRemote`, healing any best-effort fan-out failure),
     then writes `grant/revocation` records under `PermissionsModule` (see the
     [DWN specification](https://github.com/enboxorg/dwn-spec) for `parentId`
     and `tags.module` requirements) without wallet interaction, backed by a
     durable revocation retry journal (`enbox:auth:revocationRetryContext`, max
     4,096 entries) so offline/interrupted disconnects finish on next launch.

## Transport Bindings

### Relay Transport and Wallet URI Format

- **Fragment-Only Wallet URI**:
  ```
  {walletUri}#request_uri={percentEncodedRequestUri}&encryption_key={base64url32ByteKey}
  ```
  - Default custom native scheme: `enbox://connect`
  - Default browser wallet path: `https://<wallet-origin>/connect/app`
  - `parseWalletConnectUri()` reads **ONLY `URL.hash.slice(1)`** and ignores
    query parameters (`?`) so the 32-byte `encryption_key` never leaks to HTTP
    servers, CDNs, or access logs. Validates decoded key length === 32 bytes.
- **QR Code**: Byte-mode QR versions 1–10, Error Correction Level M (15%),
  quiet zone 2, imposing a hard **213-byte** UTF-8 Wallet URI cap
  (`packages/browser/src/ui/qr.ts`), so wallet and relay base URLs must remain
  compact.
- **Same-Device Handoff**: Clickable QR cards and "Continue in Wallet" links
  open with `target="_blank" rel="noopener noreferrer"` so the requesting tab
  and its relay polling session remain alive without exposing `window.opener`.
- **Auto-Reminting**: Before `onClaimed` or PIN entry, browser modals re-mint
  fresh PAR pointers at `max(10_000ms, expiresIn * 1000 - 120_000ms)` (leaving a
  2-minute safety margin).
- **Polling & PIN Retry**: Per-HTTP timeout `30s`, token poll interval `3s`,
  overall timeout `300s`, visibility-aware sleep, `AbortSignal` cancellation.
  `ConnectClient` performs a single PIN decryption attempt; `@enbox/browser`'s
  `runRelayConnect` allows up to **6** interactive PIN retries in memory
  (currently failing closed immediately only on errors prefixed with `"Connect:"`;
  implementations ****SHOULD**** also fail closed on structural JWE errors such
  as `InvalidJwe` or `AlgorithmNotSupported`).

### Browser Popup `postMessage` Transport

Provides PIN-less, zero-relay E2E encrypted connection between a browser tab and
a web wallet hosted at `https://<wallet-origin>/dweb-connect` (**zero plaintext
fallback**):

- **Synchronous Popup Launch**: Client opens `/dweb-connect` (window name
  `enbox-dweb-connect`, `width=500,height=700`) synchronously inside the click
  handler before async key generation to survive popup blockers. Default timeout
  `300s`, close poll `500ms`. Closing the popup *before* `enbox-connect-loaded`
  immediately rejects with `PopupWindowClosedError` (no grace period); closing
  *after* `enbox-connect-loaded` waits a `1,500ms` grace window before resolving
  `"DENIED"`.
- **Exact 4 Wire Messages**:
  1. Wallet → Client (`loaded` beacon):
     ```json
     {
       "type": "enbox-connect-loaded",
       "walletEpk": { "kty": "OKP", "crv": "X25519", "x": "<base64url>" },
       "walletOrigin": "https://wallet.example"
     }
     ```
     (Client validates `assertX25519PublicJwk(walletEpk)` and
     `walletOrigin === pinnedWalletOrigin`, failing closed otherwise.)
  2. Client → Wallet (ECDH-ES sealed request):
     ```json
     {
       "type": "enbox-connect-request",
       "jwe": "<Compact JWE: alg=ECDH-ES, enc=XC20P, typ=enbox-connect-req, cty=JWT, apv=base64url(UTF8(walletOrigin))>"
     }
     ```
  3. Wallet → Client (ECDH-ES sealed response or `"DENIED"`):
     ```json
     {
       "type": "enbox-connect-response",
       "payload": "<Compact JWE: alg=ECDH-ES, enc=XC20P, typ=enbox-connect-res, cty=JWT, apu=base64url(UTF8(state))> | \"DENIED\""
     }
     ```
  4. Client → Wallet (completion ack; sent immediately after opening/verifying
     the response; wallet waits up to `5,000ms`):
     ```json
     { "type": "enbox-connect-ack" }
     ```
- **Mandatory Pinning & `Referrer-Policy`**: Neither side ever uses
  `targetOrigin: "*"`. Wallet resolves `dappOrigin` from explicit config or
  `document.referrer` (failing closed if absent — client pages ****MUST NOT****
  serve `Referrer-Policy: no-referrer` or open the popup with `noreferrer`).
  Both sides drop any `MessageEvent` where
  `event.origin !== expectedOrigin || event.source !== expectedSource`. Wallet
  validates `apv === base64url(UTF8(walletOrigin))` and
  `request.reply.mode === "post_message"`, and propagates `dappOrigin` as
  `verifiedRequesterOrigin`.

## Permission Model

### High-Level Shorthands and Baseline Scopes

- Supported high-level permission shorthands: **`read`**, **`write`**, **`delete`**
  (default: `["read", "write", "delete"]`). `query`, `subscribe`, and `configure`
  shorthands do not exist. `installationOnly: true` with `permissions: []`
  requests dependency module installation with zero scopes (must accompany at
  least one scoped module in a request so `validateConnectResultGrants` observes
  at least one non-revocation grant).
- Baseline scopes added per requested module (unless `installationOnly`):
  1. `{ "interface": "Modules", "method": "Query", "module": "<moduleUri>" }`
  2. `{ "interface": "Messages", "method": "Read", "module": "<moduleUri>" }`
- **Forbidden connect scopes**: `Records.Query`, `Records.Subscribe`,
  `Records.Count` (use unified `Records.Read`), and `Modules.Configure` (the
  wallet configures modules directly during approval).

### Delegated vs. Standard Grants

| Scope | `delegated` | Rationale |
|---|---|---|
| `Records` / `Read`, `Write`, `Delete` | `true` | Delegate reads, writes, and deletes records acting as the user |
| `Messages` / `Read` | `true` | Delegate syncs and subscribes to the user's module feed |
| `Modules` / `Query` | `false` | Direct query of installed module definitions |
| `Records` / `Write` on `https://identity.foundation/dwn/permissions` (`contextId: grantId`) | `true` | Least-privilege self-revocation of one specific session grant |

## Security and Privacy Considerations

1. **Fragment-only relay key secrecy**: `request_uri` and `encryption_key` travel
   exclusively in `#fragment` (query params ignored), preventing leakage to web
   servers, proxies, CDNs, and `Referer` headers.
2. **PIN strengthens the CEK**: Passing the PIN as salt to HKDF-SHA-256
   (`info: "enbox/connect/v2/pin"`) gives both confidentiality and integrity
   protection against an adversary with access to the relay blob and even the
   ECDH secret.
3. **Algorithm allow-lists & low-order point defense**: Every `CompactJwe.decrypt`
   enforces strict `allowedAlgs`/`allowedEncs`, `typ`/`cty` checks, all-zero
   X25519 shared-secret and same-keypair rejection, public-only
   `responseKey`/`walletEpk` checks, and `did:jwk:`-pinned EdDSA JWT verification.
4. **Channel & context binding**: `apv` binds popup requests to `walletOrigin`,
   `apu` binds responses to `request.state`, popup wallets enforce
   `reply.mode === "post_message"`, and `postMessage` pins `targetOrigin`,
   `event.origin`, and `event.source` (never `"*"`).
5. **Untrusted requester metadata**: Display hints are truncated to strict schema
   bounds, and any transport supplying `verifiedRequesterOrigin` overwrites
   `connectSession.origin` while keeping the untouched request for `request_hash`.
6. **Fail-closed module encryption integrity**: Requesters cannot supply
   `$keyAgreement` keys; wallets derive keys from the owner's root key, verify
   thumbprints, forbid `Modules.Configure` delegation, and re-query endpoints for
   convergence.
7. **Pre-supplied delegate key custody**: Apps may supply `delegateDid` so
   delegate signing keys never leave the app; encryption keys arrive via
   `wrappedGrantKey` envelopes wrapped to the delegate's public X25519 key.
8. **Client-side grant validation & least-privilege self-revocation**: Clients
   reject over-scoped or mismatched grants before importing, and revocation
   grants are confined to `contextId === grantId` backed by a durable retry
   journal.
9. **Owner authentication separation & replay defense**: Owner proofs never
   export owner keys, bind HTTPS `audience` + `challenge` + canonical
   `request_hash`, check `300s`/`30s` lifetimes three times, reject deactivated
   DIDs, and require server-side atomic `(audience, challenge)` consumption.
10. **Relay data minimization**: Relay blobs expire after 600s and are deleted
    on single-use read; `/status/:requestId` and `/complete/:state` carry only
    booleans and reveal neither DIDs nor grants; same-device links use
    `rel="noopener noreferrer"`.

## Normative References

[DWN Specification](https://dwn-spec.pages.dev)
~ Enbox Decentralized Web Node Specification.

[DWN Transport Protocol Specification](https://dwn-transport-spec.pages.dev)
~ Enbox Decentralized Web Node Transport Protocol Specification.

[DWN Sync Protocol Specification](https://dwn-sync-spec.pages.dev)
~ Enbox Decentralized Web Node Sync Protocol Specification.

[[spec:RFC7515]]
~ [JSON Web Signature (JWS)](https://datatracker.ietf.org/doc/html/rfc7515). M. Jones, J. Bradley, N. Sakimura. May 2015.

[[spec:RFC7516]]
~ [JSON Web Encryption (JWE)](https://datatracker.ietf.org/doc/html/rfc7516). M. Jones, J. Hildebrand. May 2015.

[[spec:RFC7518]]
~ [JSON Web Algorithms (JWA)](https://datatracker.ietf.org/doc/html/rfc7518). M. Jones. May 2015.

[[spec:RFC7748]]
~ [Elliptic Curves for Security](https://datatracker.ietf.org/doc/html/rfc7748). A. Langley, M. Hamburg, S. Turner. January 2016.

[[spec:RFC5869]]
~ [HMAC-based Extract-and-Expand Key Derivation Function (HKDF)](https://datatracker.ietf.org/doc/html/rfc5869). H. Krawczyk, P. Eronen. May 2010.

[[spec:RFC2119]]
~ [Key words for use in RFCs to Indicate Requirement Levels](https://datatracker.ietf.org/doc/html/rfc2119). S. Bradner. March 1997.
