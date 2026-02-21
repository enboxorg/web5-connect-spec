
Web5 Connect Protocol
==================

**Specification Status:** Draft

**Latest Draft:**
  TBD
<!-- -->

**Editors:**
~ [Liran Cohen](https://www.linkedin.com/in/itsliran/) (Enbox)

**Participate:**
~ [GitHub repo](https://github.com/enboxorg/web5-connect-spec)
~ [File a bug](https://github.com/enboxorg/web5-connect-spec/issues)
~ [Commit history](https://github.com/enboxorg/web5-connect-spec/commits/main)

------------------------------------

## Abstract

The Web5 Connect Protocol enables applications to request delegated access to a
user's [Decentralized Web Node (DWN)](https://github.com/enboxorg/dwn-spec) through
their wallet or identity agent. It builds on SIOPv2 (Self-Issued OpenID Provider v2)
and OAuth 2.0 Pushed Authorization Requests (PAR) to establish a secure, end-to-end
encrypted channel between an application and a wallet, mediated by a stateless
connect server.

Upon successful connection, the application receives:

- A **delegate DID** — a new DID created by the wallet that the application uses to
  act on behalf of the user's identity.
- **Permission grants** — DWN authorization records that grant the delegate DID
  specific capabilities (read, write, sync, etc.) scoped to one or more protocols.

This specification defines the complete protocol flow, message formats, encryption
schemes, and security properties required for interoperable wallet-to-application
connections.

## Status of This Document

Web5 Connect Protocol is a _DRAFT_ specification under development by
[Enbox](https://enbox.org). It is a companion specification to the
[DWN specification](https://github.com/enboxorg/dwn-spec) and the
[DWN Transport Protocol specification](https://github.com/enboxorg/dwn-transport-spec).

The specification will be updated to incorporate feedback. We encourage reviewers
to submit [GitHub Issues](https://github.com/enboxorg/web5-connect-spec/issues)
as the means by which to communicate feedback and contributions.

## Terminology

[[def:Client, Application, Relying Party]]
~ The application requesting access to a user's DWN. The Client initiates the
connect flow by generating an auth request and presenting a connection URI
(QR code or deep link) to the user.

[[def:Provider, Wallet, Identity Agent]]
~ The software that manages the user's DIDs, keys, and DWN access. The Provider
receives the auth request, presents it to the user for consent, creates a
delegate DID and permission grants, and returns them to the Client.

[[def:Connect Server, Relay]]
~ A stateless intermediary that relays encrypted messages between the
[[ref:Client]] and [[ref:Provider]]. The Connect Server never sees plaintext —
it only stores and serves opaque encrypted blobs. The Connect Server ****MAY****
be co-located with a DWN Server or operated independently.

[[def:Delegate DID]]
~ A new DID (typically `did:jwk`) created by the [[ref:Provider]] specifically
for a connection session. The [[ref:Client]] uses this DID to sign DWN messages
on behalf of the user. The delegate DID's private keys are transferred to the
Client as part of the auth response.

[[def:Permission Grant]]
~ A DWN authorization record (a `RecordsWrite` to the permissions protocol at
`https://tbd.website/dwn/permissions`) that grants the [[ref:Delegate DID]]
specific operations on the user's DWN. See the
[DWN specification](https://github.com/enboxorg/dwn-spec) Permissions section.

[[def:Delegated Grant]]
~ A [[ref:Permission Grant]] with `delegated: true` that allows the grantee to
act on behalf of the grantor (the user). When the [[ref:Client]] uses a delegated
grant, the DWN treats the message as if it came from the user.

[[def:Wallet URI]]
~ A URI constructed by the [[ref:Client]] that contains the auth request location
and a symmetric encryption key. Presented to the user as a QR code or deep link
to launch the [[ref:Provider]].

[[def:PIN]]
~ A short numeric code displayed by the [[ref:Provider]] to the user during the
connect flow. The user communicates this PIN to the [[ref:Client]] application
out-of-band. The PIN is used as additional authenticated data (AAD) in the
auth response encryption, ensuring that only someone who received the PIN from
the wallet can decrypt the response.


## Overview {#overview}

### Actors {#actors}

The Web5 Connect Protocol involves three actors:

1. **Client (Application)** — A web or native application that wants to interact
   with a user's DWN. The Client does not hold the user's real DID; it operates
   through a [[ref:Delegate DID]] with explicit [[ref:Permission Grants]].

2. **Provider (Wallet)** — The user's trusted identity manager. The Provider holds
   the user's real DID(s) and private keys. It evaluates permission requests,
   obtains user consent, creates delegate credentials, and returns them encrypted
   to the Client.

3. **Connect Server (Relay)** — A simple store-and-forward service that holds
   encrypted blobs temporarily. It exposes four HTTP endpoints. The Connect Server
   sees only ciphertext and cannot decrypt or tamper with the exchanged data.

### Protocol Flow {#protocol-flow}

```
Client                     Connect Server                   Provider (Wallet)
  |                              |                               |
  |  1. POST /connect/par       |                               |
  |  (encrypted auth request)   |                               |
  |----------------------------->|                               |
  |                              |                               |
  |  2. PushedAuthResponse       |                               |
  |  {request_uri, expires_in}  |                               |
  |<-----------------------------|                               |
  |                              |                               |
  |  3. Construct Wallet URI     |                               |
  |  (request_uri + enc key)    |                               |
  |  Display QR / deep link      |                               |
  |                              |                               |
  |                              |  4. GET /connect/authorize/:id.jwt
  |                              |  (fetch encrypted auth request)
  |                              |<------------------------------|
  |                              |                               |
  |                              |  5. Return encrypted blob     |
  |                              |------------------------------>|
  |                              |                               |
  |                              |  6. Decrypt, verify, display   |
  |                              |     User approves + sees PIN  |
  |                              |                               |
  |                              |  7. Create delegate DID       |
  |                              |     Create permission grants  |
  |                              |                               |
  |                              |  8. POST /connect/callback    |
  |                              |  (encrypted auth response)    |
  |                              |<------------------------------|
  |                              |                               |
  |  9. GET /connect/token/:state.jwt                            |
  |  (poll for auth response)   |                               |
  |----------------------------->|                               |
  |                              |                               |
  | 10. Return encrypted blob   |                               |
  |<-----------------------------|                               |
  |                              |                               |
  | 11. User enters PIN          |                               |
  | 12. Decrypt + verify         |                               |
  | 13. Import delegate DID      |                               |
  |     + permission grants     |                               |
```

### Security Goals {#security-goals}

- **End-to-end encryption:** The Connect Server never sees plaintext auth requests
  or responses.
- **Man-in-the-middle resistance:** The auth response encryption uses ECDH key
  agreement plus a user-communicated PIN as authenticated data.
- **Replay protection:** Random `nonce` and `state` values prevent replay attacks.
- **Provider DID privacy:** The user's real DID is only revealed inside the
  encrypted auth response, never to the Connect Server.
- **One-time retrieval:** Auth requests and responses are deleted from the Connect
  Server after retrieval, preventing replay.


## Connect Server Endpoints {#connect-server-endpoints}

The Connect Server exposes four HTTP endpoints. All paths are relative to the
Connect Server's base URL.

### Pushed Authorization Request {#endpoint-par}

```
POST /connect/par
```

Stores an encrypted auth request blob from the [[ref:Client]].

**Request:**

- `Content-Type`: `application/json`
- Body:

```json
{
  "request": "<JWE compact serialization>"
}
```

The `request` field ****MUST**** contain the encrypted auth request as a JWE
compact serialization string (see [Auth Request Encryption](#auth-request-encryption)).

The request object ****MUST NOT**** contain a `request_uri` field.

**Response:**

- Status: `201 Created`
- Body:

```json
{
  "request_uri": "https://connect.example.com/connect/authorize/abc123.jwt",
  "expires_in": 600
}
```

| Field | Type | Description |
|---|---|---|
| `request_uri` | `string` | URL where the Provider can fetch the auth request |
| `expires_in` | `number` | TTL in seconds before the stored request expires |

The server ****MUST**** generate a unique, unguessable identifier for the request
and construct the `request_uri` as `{baseUrl}/connect/authorize/{id}.jwt`.

**Errors:**
- `400 Bad Request` if the body is missing, malformed, or contains a `request_uri`.

### Authorization Request Retrieval {#endpoint-authorize}

```
GET /connect/authorize/:id.jwt
```

Retrieves a stored encrypted auth request. Used by the [[ref:Provider]].

**Response:**
- Status: `200 OK`
- `Content-Type`: `application/jwt`
- Body: The JWE compact serialization string (the raw encrypted blob).

The server ****MUST**** delete the stored request after successful retrieval
(one-time use).

**Errors:**
- `404 Not Found` if the request ID does not exist or has expired.

### Callback {#endpoint-callback}

```
POST /connect/callback
```

Stores an encrypted auth response from the [[ref:Provider]].

**Request:**

- `Content-Type`: `application/json`
- Body:

```json
{
  "id_token": "<JWE compact serialization>",
  "state": "<state value from auth request>"
}
```

| Field | Type | Description |
|---|---|---|
| `id_token` | `string` | The encrypted auth response as a JWE compact serialization |
| `state` | `string` | The `state` value from the original auth request, used to correlate the response |

**Response:**
- Status: `201 Created`

**Errors:**
- `400 Bad Request` if required fields are missing.

### Token Retrieval {#endpoint-token}

```
GET /connect/token/:state.jwt
```

Retrieves a stored encrypted auth response. Used by the [[ref:Client]].

The `:state` path parameter ****MUST**** match the `state` value from the original
auth request.

**Response:**
- Status: `200 OK`
- `Content-Type`: `application/jwt`
- Body: The JWE compact serialization string (the encrypted auth response).

The server ****MUST**** delete the stored response after successful retrieval
(one-time use).

**Errors:**
- `404 Not Found` if no response has been submitted for this state, or if it has
  expired.

The [[ref:Client]] ****SHOULD**** poll this endpoint at a regular interval
(****RECOMMENDED**** every 3 seconds) until a response is available or a timeout
is reached (****RECOMMENDED**** 5-minute timeout).

### Endpoint Discovery {#endpoint-discovery}

The Connect Server URL is provided by the [[ref:Client]] application at connection
time and included in the [[ref:Wallet URI]]. The Connect Server is independent of
any specific DWN Server — it ****MAY**** be co-located with a DWN Server or operated
as a standalone service.

Connect Servers ****MUST**** support HTTPS. Connect Servers ****MUST**** include
CORS headers allowing cross-origin requests from browser-based Clients:

| Header | Value |
|---|---|
| `Access-Control-Allow-Origin` | `*` |
| `Access-Control-Allow-Methods` | `GET, POST, OPTIONS` |
| `Access-Control-Allow-Headers` | `*` |


## Auth Request {#auth-request}

The auth request is a signed JWT encrypted in a JWE, containing the application's
identity, requested permissions, and protocol definitions.

### SIOPv2 Auth Request Object {#siop-auth-request}

The auth request payload extends the SIOPv2 Authentication Request with Web5-specific
fields. The base SIOPv2 fields are:

| Field | Type | Required | Description |
|---|---|---|---|
| `client_id` | `string` | Yes | DID of the Client's ephemeral DID (e.g., `did:jwk:...`) |
| `scope` | `string` | Yes | ****MUST**** be `"openid did:jwk"` |
| `response_type` | `string` | Yes | ****MUST**** be `"id_token"` |
| `redirect_uri` | `string` | Yes | The callback endpoint URL (`{connectServerUrl}/connect/callback`) |
| `response_mode` | `string` | Yes | ****MUST**** be `"direct_post"` |
| `state` | `string` | Yes | Random 16-byte base64url-encoded anti-CSRF token |
| `nonce` | `string` | Yes | Random 16-byte base64url-encoded anti-replay token |
| `client_metadata` | `object` | No | Client metadata (see below) |

#### Client Metadata

```json
{
  "client_metadata": {
    "subject_syntax_types_supported": ["did:dht", "did:jwk"],
    "client_name": "My Application",
    "client_uri": "https://myapp.example.com",
    "logo_uri": "https://myapp.example.com/logo.png"
  }
}
```

The `subject_syntax_types_supported` array indicates which DID methods the Client
can work with. The Provider ****SHOULD**** select a DID method supported by the Client.

### Web5 Connect Extensions {#web5-extensions}

The auth request includes additional fields beyond SIOPv2:

| Field | Type | Required | Description |
|---|---|---|---|
| `displayName` | `string` | Yes | Human-readable application name, displayed to the user in the Provider's consent UI |
| `permissionRequests` | `array` | Yes | Array of permission request objects |

#### Permission Requests {#permission-requests}

Each element of `permissionRequests` is an object with:

| Field | Type | Description |
|---|---|---|
| `protocolDefinition` | `object` | The full DWN protocol definition object. The Provider installs this protocol on the user's DWN if not already present. |
| `permissionScopes` | `array` | Array of DWN permission scope objects describing the specific operations requested |

::: example Auth Request Permission Requests
```json
{
  "permissionRequests": [
    {
      "protocolDefinition": {
        "protocol": "https://social.example/protocol",
        "published": true,
        "types": {
          "post": { "schema": "https://social.example/post", "dataFormats": ["application/json"] },
          "comment": { "schema": "https://social.example/comment", "dataFormats": ["application/json"] }
        },
        "structure": {
          "post": {
            "$actions": [
              { "who": "anyone", "can": ["create", "read"] }
            ],
            "comment": {
              "$actions": [
                { "who": "anyone", "can": ["create", "read"] }
              ]
            }
          }
        }
      },
      "permissionScopes": [
        { "interface": "Records", "method": "Write", "protocol": "https://social.example/protocol" },
        { "interface": "Records", "method": "Read", "protocol": "https://social.example/protocol" },
        { "interface": "Records", "method": "Query", "protocol": "https://social.example/protocol" },
        { "interface": "Records", "method": "Subscribe", "protocol": "https://social.example/protocol" },
        { "interface": "Records", "method": "Delete", "protocol": "https://social.example/protocol" },
        { "interface": "Protocols", "method": "Query", "protocol": "https://social.example/protocol" },
        { "interface": "Messages", "method": "Read", "protocol": "https://social.example/protocol" },
        { "interface": "Messages", "method": "Sync", "protocol": "https://social.example/protocol" },
        { "interface": "Messages", "method": "Subscribe", "protocol": "https://social.example/protocol" }
      ]
    }
  ]
}
```
:::

#### Baseline Scopes {#baseline-scopes}

To enable sync and basic protocol interaction, the following scopes ****SHOULD****
be included as baseline for every protocol permission request, regardless of the
application's specific needs:

| Scope | Purpose |
|---|---|
| `Protocols` / `Query` (scoped to protocol) | Verify protocol availability |
| `Messages` / `Read` (scoped to protocol) | Read messages during sync |
| `Messages` / `Sync` (scoped to protocol) | Compare tree state during sync |
| `Messages` / `Subscribe` (scoped to protocol) | Receive real-time sync events |

::: note
A future revision may simplify the three Messages scopes (`Read`, `Sync`,
`Subscribe`) into a single unified `Messages` / `Read` scope, aligning with how
protocol `$actions` already treats `read` as a single action covering read, query,
subscribe, and count operations. Client and Provider implementations ****SHOULD****
be prepared for this simplification.
:::

### Auth Request Signing {#auth-request-signing}

The auth request ****MUST**** be signed as a JWT using the Client's ephemeral DID:

**JWT Header:**
```json
{
  "alg": "EdDSA",
  "kid": "did:jwk:<client-ephemeral-did>#0",
  "typ": "JWT"
}
```

**JWT Payload:** The complete auth request object (SIOPv2 fields + Web5 extensions).

**Signature:** Ed25519 EdDSA, signed with the Client's ephemeral DID's signing key.

The Client ****MUST**** generate a new ephemeral `did:jwk` (Ed25519) for each
connect session. This DID is used for:
1. Signing the auth request JWT.
2. ECDH key agreement to derive the auth response decryption key.

### Auth Request Encryption {#auth-request-encryption}

The signed JWT ****MUST**** be encrypted as a JWE compact serialization using
XChaCha20-Poly1305 with a random symmetric key:

**JWE Protected Header:**
```json
{
  "alg": "dir",
  "cty": "JWT",
  "enc": "XC20P",
  "typ": "JWT"
}
```

| Field | Value | Description |
|---|---|---|
| `alg` | `"dir"` | Direct key agreement — the encryption key is used directly, not wrapped |
| `cty` | `"JWT"` | The content type is a JWT |
| `enc` | `"XC20P"` | XChaCha20-Poly1305 content encryption |
| `typ` | `"JWT"` | The overall token type |

**Encryption parameters:**
- **Key:** 32 random bytes generated by the Client.
- **Nonce:** 24 random bytes (XChaCha20 uses a 192-bit nonce).
- **AAD (Additional Authenticated Data):** The base64url-encoded protected header.
- **Auth tag:** 16 bytes, produced by the Poly1305 MAC.

**JWE Compact Serialization:**
```
base64url(protectedHeader) . "" . base64url(nonce) . base64url(ciphertext) . base64url(authTag)
```

The second segment (encrypted key) is empty because `alg: "dir"` uses the key
directly.

### Encryption Key Distribution {#encryption-key-distribution}

The 32-byte symmetric encryption key is distributed to the [[ref:Provider]] via
the [[ref:Wallet URI]] (see [Wallet URI Format](#wallet-uri-format)). This key is
shared out-of-band (via QR code or deep link), not through the Connect Server.


## Provider Processing {#provider-processing}

This section describes how the [[ref:Provider]] processes an incoming connect request.

### Request Retrieval and Decryption {#request-retrieval}

1. The Provider extracts the `request_uri` and `encryption_key` from the
   [[ref:Wallet URI]].
2. The Provider fetches the encrypted auth request:
   ```
   GET {request_uri}
   ```
3. The Provider decrypts the JWE using the `encryption_key`:
   - Decode the `encryption_key` from base64url to 32 raw bytes.
   - Split the JWE compact serialization into its 5 components.
   - Decrypt using XChaCha20-Poly1305 with the protected header as AAD.
   - The result is the signed JWT string.

### JWT Verification {#jwt-verification}

1. Parse the JWT header to extract the `kid` field.
2. Resolve the Client's ephemeral DID from the `kid` (the DID portion before `#`).
3. Find the matching public key in the resolved DID document's `verificationMethod`.
4. Verify the Ed25519 EdDSA signature.
5. The verified payload is the `Web5ConnectAuthRequest` object.

The Provider ****MUST**** reject the request if:
- The DID cannot be resolved.
- The `kid` does not match any verification method.
- The signature is invalid.

### User Consent {#user-consent}

The Provider ****MUST**** present the auth request to the user for approval:

- Display the `displayName` of the requesting application.
- Display the `client_metadata` if present (app name, URI, logo).
- List the requested permissions in human-readable form.
- Allow the user to select which of their DIDs to connect.
- Display a randomly generated PIN (numeric, ****RECOMMENDED**** 4-6 digits)
  that the user must communicate to the Client application out-of-band.

The Provider ****MUST NOT**** proceed without explicit user consent.

### Delegate DID Creation {#delegate-did-creation}

Upon user approval, the Provider creates a new [[ref:Delegate DID]]:

1. Generate a new `did:jwk` with an Ed25519 key pair.
2. Export the DID as a `PortableDid` object containing:
   - `uri` — The DID URI string.
   - `document` — The DID document.
   - `privateKeys` — Array of private key JWK objects.

The Delegate DID is disposable — it exists solely for this connection session.
The Provider ****SHOULD**** create a new Delegate DID for each connect session.

### Protocol Installation {#protocol-installation}

For each `permissionRequest` in the auth request:

1. Query the user's DWN for the protocol: `ProtocolsQuery` with
   `filter.protocol` matching the `protocolDefinition.protocol`.
2. **If the protocol is not installed:**
   - Create a `ProtocolsConfigure` message with the provided `protocolDefinition`.
   - Process the message on the user's remote DWN(s).
   - Process the message on the user's local DWN.
3. **If the protocol is already installed:**
   - Send the existing `ProtocolsConfigure` message to the user's remote DWN(s)
     to ensure it is available for the Client's sync operations.

::: note
Installing the protocol on the remote DWN before granting permissions ensures that
when the Client begins syncing, the protocol definition is already present. Without
this, the Client's sync would fail because protocol-scoped records cannot be
processed without the protocol definition.
:::

### Permission Grant Creation {#permission-grant-creation}

For each permission scope in the auth request:

1. Determine whether the grant should be [[ref:Delegated Grant]] or a standard grant:
   - **Delegated (`delegated: true`):** Used for `Records` interface scopes (all
     methods) and `Protocols.Configure`. Delegated grants allow the Delegate DID
     to act on behalf of the user's real DID.
   - **Standard (`delegated: false`):** Used for `Messages` interface scopes
     (`Read`, `Sync`, `Subscribe`) and `Protocols.Query`. These give the Delegate
     DID direct permissions without delegation.

2. Create a permission grant record:
   ```json
   {
     "delegated": true,
     "grantedTo": "<delegate-did-uri>",
     "scope": {
       "interface": "Records",
       "method": "Write",
       "protocol": "https://social.example/protocol"
     },
     "dateExpires": "2040-06-25T16:09:16.693356Z"
   }
   ```

3. Process the grant on the user's local DWN (with `store: true`).
4. Send the grant to the user's remote DWN(s) so the Client can sync it down.

::: note
Grant expiry is an area for future improvement. A fixed far-future expiry is used
currently. Future revisions of this specification may define expiry negotiation
between the Client and Provider, and a grant renewal protocol.
:::

#### Delegated vs. Standard Grants {#delegated-vs-standard}

| Scope | Delegated | Rationale |
|---|---|---|
| `Records.Write` | Yes | Client creates records as the user |
| `Records.Read` | Yes | Client reads records as the user |
| `Records.Query` | Yes | Client queries records as the user |
| `Records.Subscribe` | Yes | Client subscribes as the user |
| `Records.Delete` | Yes | Client deletes records as the user |
| `Records.Count` | Yes | Client counts records as the user |
| `Protocols.Configure` | Yes | Client manages protocols as the user |
| `Protocols.Query` | No | Direct permission is sufficient |
| `Messages.Read` | No | Direct permission for sync |
| `Messages.Sync` | No | Direct permission for sync |
| `Messages.Subscribe` | No | Direct permission for sync |

All permission scopes in a request ****MUST**** include a `protocol` matching
the `protocolDefinition.protocol` of the corresponding permission request.
The Provider ****MUST**** validate this constraint and reject scopes that
reference a protocol not included in the request.


## Auth Response {#auth-response}

### SIOPv2 Auth Response Object {#siop-auth-response}

The auth response payload extends the SIOPv2 Authentication Response:

| Field | Type | Description |
|---|---|---|
| `iss` | `string` | The user's selected DID (the Provider's real identity) |
| `sub` | `string` | The [[ref:Delegate DID]] URI |
| `aud` | `string` | The Client's ephemeral DID (`client_id` from the auth request) |
| `iat` | `number` | Issued-at timestamp (Unix epoch seconds) |
| `exp` | `number` | Expiry timestamp (****MUST**** be `iat + 600`, i.e., 10 minutes) |
| `nonce` | `string` | Echoed from the auth request |

### Web5 Connect Extensions {#web5-response-extensions}

| Field | Type | Description |
|---|---|---|
| `delegateGrants` | `array` | Array of DWN `RecordsWrite` messages representing the permission grants |
| `delegatePortableDid` | `object` | The [[ref:Delegate DID]] as a `PortableDid` (including private keys) |

#### Delegate Grants {#delegate-grants}

Each element of `delegateGrants` is a full DWN `RecordsWrite` message with `encodedData`:

```json
{
  "recordId": "bafyrei...",
  "contextId": "bafyrei...",
  "descriptor": {
    "interface": "Records",
    "method": "Write",
    "protocol": "https://tbd.website/dwn/permissions",
    "protocolPath": "grant",
    "recipient": "<delegate-did-uri>",
    "dataCid": "bafkrei...",
    "dataSize": 156,
    "dateCreated": "2024-08-02T06:36:37.675594Z",
    "messageTimestamp": "2024-08-02T06:36:37.675594Z",
    "dataFormat": "application/json"
  },
  "authorization": {
    "signature": {
      "payload": "...",
      "signatures": [{ "protected": "...", "signature": "..." }]
    }
  },
  "encodedData": "<base64url-encoded-grant-data>"
}
```

The `encodedData` contains the base64url-encoded JSON grant payload including the
scope, expiry, and delegation flag.

#### Delegate Portable DID {#delegate-portable-did}

```json
{
  "uri": "did:jwk:eyJr...",
  "document": {
    "id": "did:jwk:eyJr...",
    "verificationMethod": [{
      "id": "did:jwk:eyJr...#0",
      "type": "JsonWebKey",
      "controller": "did:jwk:eyJr...",
      "publicKeyJwk": { "crv": "Ed25519", "kty": "OKP", "x": "..." }
    }],
    "authentication": ["did:jwk:eyJr...#0"],
    "assertionMethod": ["did:jwk:eyJr...#0"]
  },
  "privateKeys": [{
    "crv": "Ed25519",
    "d": "...",
    "kty": "OKP",
    "x": "..."
  }]
}
```

### Auth Response Signing {#auth-response-signing}

The auth response ****MUST**** be signed as a JWT using the [[ref:Delegate DID]]
(not the Provider's real DID):

**JWT Header:**
```json
{
  "alg": "EdDSA",
  "kid": "did:jwk:<delegate-did>#0",
  "typ": "JWT"
}
```

**JWT Payload:** The complete auth response object (SIOPv2 fields + Web5 extensions).

**Signature:** Ed25519 EdDSA, signed with the Delegate DID's signing key.

::: note
Signing with the Delegate DID rather than the Provider's real DID preserves privacy —
the Connect Server and any interceptor cannot correlate the response to the user's
real identity by inspecting the JWT header.
:::

### Auth Response Encryption {#auth-response-encryption}

The signed JWT ****MUST**** be encrypted as a JWE compact serialization using an
ECDH-derived key and XChaCha20-Poly1305, with the [[ref:PIN]] as additional
authenticated data.

#### ECDH Key Agreement {#ecdh-key-agreement}

1. Take the Delegate DID's Ed25519 private key.
2. Take the Client's ephemeral Ed25519 public key (resolved from the `client_id` DID).
3. Convert both keys from Ed25519 to X25519:
   - Private key: `Ed25519.convertPrivateKeyToX25519(delegatePrivateKey)`
   - Public key: `Ed25519.convertPublicKeyToX25519(clientPublicKey)`
4. Compute the X25519 shared secret:
   - `sharedSecret = X25519.sharedSecret(delegateX25519Private, clientX25519Public)`
5. Derive the encryption key via HKDF-SHA256:
   - `key = HKDF-SHA256(ikm: sharedSecret, salt: empty, info: empty, length: 32)`

The resulting 32-byte key is the symmetric encryption key for the JWE.

::: note
This works because X25519 shared secret is commutative:
`X25519(a, G*b) == X25519(b, G*a)`. The Client can derive the same key using its
own private key and the Delegate DID's public key (extracted from the `kid` in the
JWE header).
:::

#### PIN as AAD {#pin-as-aad}

The [[ref:PIN]] is incorporated into the encryption as Additional Authenticated Data
(AAD). The AAD is constructed by appending the PIN to the JWE protected header object:

```json
{
  "alg": "dir",
  "cty": "JWT",
  "enc": "XC20P",
  "typ": "JWT",
  "kid": "did:jwk:<delegate-did>#0",
  "pin": "9876"
}
```

The `pin` field is included in the AAD computation but ****MUST NOT**** be included
in the actual JWE protected header. The actual protected header is:

```json
{
  "alg": "dir",
  "cty": "JWT",
  "enc": "XC20P",
  "typ": "JWT",
  "kid": "did:jwk:<delegate-did>#0"
}
```

The `kid` field in the protected header carries the Delegate DID's key ID, enabling
the Client to resolve the Delegate DID and perform the same ECDH derivation.

The PIN ensures that even if an attacker intercepts the JWE and knows the Client's
ephemeral DID (which is public), they cannot decrypt the response without the PIN.
An incorrect PIN causes an AEAD authentication failure (the Poly1305 tag check fails).

#### JWE Construction {#jwe-construction}

- **Encryption:** XChaCha20-Poly1305 (`XC20P`).
- **Nonce:** 24 random bytes.
- **AAD:** Base64url encoding of the JSON object `{protectedHeader fields + "pin": "<pin>"}`.
- **Compact serialization:** Same structure as the auth request JWE.


## Client Processing {#client-processing}

This section describes how the [[ref:Client]] receives and processes the auth response.

### Polling for Response {#polling}

After presenting the [[ref:Wallet URI]] to the user, the Client polls the
[Token Retrieval](#endpoint-token) endpoint:

```
GET {connectServerUrl}/connect/token/{state}.jwt
```

- Poll interval: ****RECOMMENDED**** 3 seconds.
- Timeout: ****RECOMMENDED**** 5 minutes.
- The Client ****MUST**** stop polling on `200 OK` (response available) or after
  the timeout expires.
- A `404` response indicates the Provider has not yet submitted the response.

### PIN Collection {#pin-collection}

When a response is received, the Client ****MUST**** collect the PIN from the user.
The PIN was displayed to the user by the [[ref:Provider]] during the consent step.
The user communicates it to the Client out-of-band (e.g., types it into a form field).

### Response Decryption {#response-decryption}

1. Split the JWE compact serialization into its 5 components.
2. Parse the protected header to extract the `kid` field.
3. Extract the Delegate DID from the `kid` (the DID portion before `#`).
4. Resolve the Delegate DID to obtain its public key.
5. Derive the ECDH shared key:
   - Convert the Client's ephemeral Ed25519 private key to X25519.
   - Convert the Delegate DID's Ed25519 public key to X25519.
   - Compute `sharedSecret = X25519(clientX25519Private, delegateX25519Public)`.
   - Derive `key = HKDF-SHA256(ikm: sharedSecret, salt: empty, info: empty, length: 32)`.
6. Construct the AAD by adding the user-entered PIN to the protected header object.
7. Decrypt using XChaCha20-Poly1305 with the derived key and AAD.
8. If decryption fails (authentication tag mismatch), the PIN is incorrect. The
   Client ****SHOULD**** prompt the user to re-enter the PIN.

### JWT Verification {#response-jwt-verification}

After decryption, verify the JWT:

1. Extract the `kid` from the JWT header.
2. Resolve the Delegate DID.
3. Verify the Ed25519 EdDSA signature against the Delegate DID's public key.
4. Validate the `aud` field matches the Client's ephemeral DID (`client_id`).
5. Validate the `nonce` field matches the nonce from the original auth request.
6. Validate the `exp` field has not passed.

### Grant and DID Import {#grant-import}

After successful verification:

1. Extract `connectedDid` from the `iss` field — this is the user's real DID.
2. Extract `delegatePortableDid` — import the Delegate DID and its private keys
   into the Client's local key store.
3. Extract `delegateGrants` — import each permission grant record into the Client's
   local DWN.
4. The Client is now connected and can use the Delegate DID with the permission
   grants to interact with the user's DWN.


## Wallet URI Format {#wallet-uri-format}

The [[ref:Wallet URI]] is a URL constructed by the [[ref:Client]] that bootstraps
the connect flow by directing the [[ref:Provider]] to fetch the encrypted auth request.

### URI Structure {#uri-structure}

```
{walletBaseUri}?request_uri={encodedRequestUri}&encryption_key={encodedKey}
```

| Parameter | Encoding | Description |
|---|---|---|
| `request_uri` | URL-encoded | The `request_uri` returned by the [PAR endpoint](#endpoint-par) |
| `encryption_key` | Base64url | The 32-byte symmetric encryption key for the auth request |

::: example Wallet URI
```
https://wallet.example.com/?request_uri=https%3A%2F%2Fconnect.example.com%2Fconnect%2Fauthorize%2Fabc123.jwt&encryption_key=dGhpcyBpcyBhIHRlc3Qga2V5IGZvciBleGFtcGxl
```
:::

### QR Code Encoding {#qr-code}

When the connection is initiated cross-device (e.g., desktop browser to mobile wallet),
the Client ****SHOULD**** encode the Wallet URI as a QR code for scanning.

The QR code ****SHOULD**** use error correction level M (medium, 15% recovery).

### Deep Link Handling {#deep-link}

When the connection is initiated on the same device, the Client ****MAY**** use the
Wallet URI as a deep link to launch the Provider application directly.

The `walletBaseUri` ****MAY**** use a custom URI scheme (e.g., `web5://`) or an
HTTPS URL that the Provider application handles via universal links or app links.


## Permission Model {#permission-model}

### Mapping User Permissions to DWN Scopes {#permission-mapping}

Applications typically request permissions using high-level terms. The following
mapping defines how these translate to DWN permission scopes:

| User Permission | DWN Scopes Generated |
|---|---|
| `write` | `Records.Write` scoped to protocol |
| `read` | `Records.Read` scoped to protocol |
| `delete` | `Records.Delete` scoped to protocol |
| `query` | `Records.Query` scoped to protocol |
| `subscribe` | `Records.Subscribe` scoped to protocol |
| `configure` | `Protocols.Configure` scoped to protocol |

In addition to the user-requested permissions, the [Baseline Scopes](#baseline-scopes)
are always included.

### Required Baseline Scopes {#required-baselines}

Every protocol permission request ****MUST**** include the following baseline scopes
to ensure the Client can sync and interact with the protocol:

1. `Protocols.Query` — To verify the protocol is installed.
2. `Messages.Read` — To read messages during sync.
3. `Messages.Sync` — To perform set reconciliation during sync.
4. `Messages.Subscribe` — To receive real-time sync events.

Without these scopes, the Client cannot synchronize the protocol's data between its
local DWN and the user's remote DWN.

### Grant Expiry {#grant-expiry}

Permission grants ****MUST**** include a `dateExpires` field. Providers ****SHOULD****
set a reasonable expiry based on the application's needs and the sensitivity of the
data.

::: note
This specification does not currently define a grant renewal protocol. Applications
with expired grants must initiate a new connect flow. Future revisions may define
a renewal mechanism.
:::

### Grant Revocation {#grant-revocation}

The user ****MAY**** revoke permission grants at any time by creating a
`grant/revocation` record in the permissions protocol (as defined in the DWN specification).

When a grant is revoked, subsequent messages from the [[ref:Client]] using that
grant will be rejected by the DWN with status `401`.

Clients ****SHOULD**** handle `401` responses gracefully and inform the user that
their permissions have been revoked.


## Security Considerations {#security-considerations}

### Connect Server Trust Model {#server-trust-model}

The Connect Server is treated as an **untrusted relay**. It stores and forwards
encrypted blobs but cannot read or modify their contents. The security of the
protocol does not depend on the Connect Server's integrity — a compromised server
can only:

- Refuse to deliver messages (denial of service).
- Observe timing and metadata (which DIDs are connecting, when).
- Return stale or previously seen encrypted blobs (prevented by one-time retrieval
  and nonce validation).

A compromised server ****CANNOT****:
- Read the auth request or response contents.
- Forge or modify the encrypted payloads (AEAD authentication prevents tampering).
- Impersonate the Provider or Client.

### End-to-End Encryption {#e2e-encryption}

Two layers of encryption protect the protocol:

1. **Auth request:** Encrypted with a random 256-bit symmetric key shared via QR
   code or deep link. The Connect Server never sees this key.
2. **Auth response:** Encrypted with an ECDH-derived key (Ed25519 → X25519 conversion,
   then HKDF-SHA256). The shared secret can only be computed by holders of the
   Client's ephemeral private key and the Delegate DID's private key.

Both layers use XChaCha20-Poly1305 (AEAD), providing both confidentiality and
integrity.

### PIN Security {#pin-security}

The PIN provides a second factor of authentication for the auth response:

- Even if an attacker compromises the Connect Server and obtains the encrypted
  response, they cannot decrypt it without both the ECDH shared secret (requires
  the Client's private key) AND the correct PIN.
- The PIN is never transmitted over the network — it is communicated out-of-band
  (displayed on the Provider, entered on the Client).
- The PIN is included in the AEAD's Additional Authenticated Data (AAD), not as
  input to key derivation. This means the PIN provides authentication, not
  confidentiality — a brute-force attack on a short PIN is feasible if the attacker
  already has the ECDH shared secret.

Providers ****SHOULD**** use PINs of at least 4 digits. Providers ****MAY**** use
longer PINs or alphanumeric codes for higher-security applications.

### Ephemeral DID Security {#ephemeral-did-security}

- The Client's ephemeral DID is single-use and generated fresh for each connect session.
  Compromising one session's ephemeral DID does not affect other sessions.
- The Delegate DID's private keys are transferred to the Client in the encrypted
  response. The Client ****MUST**** store these keys securely (e.g., in an encrypted
  vault).
- The Delegate DID's private keys ****SHOULD NOT**** be stored in plaintext on disk
  or in browser localStorage without encryption.

### Replay Protection {#replay-protection}

- The `state` parameter is a random 16-byte value that correlates the auth request
  with the response. A replayed response for a different session will have a
  mismatched `state`.
- The `nonce` parameter is echoed from the request to the response and validated
  by the Client.
- Auth request and response blobs are deleted from the Connect Server after retrieval
  (one-time use), preventing replay of the same flow.

### One-Time Retrieval {#one-time-retrieval}

Connect Servers ****MUST**** delete stored auth requests and responses after they
are retrieved. This ensures:

- A stored blob cannot be retrieved multiple times.
- An attacker who gains access to the server after the flow completes finds no
  sensitive data.

Connect Servers ****MUST**** expire stored blobs after their TTL
(****RECOMMENDED**** 10 minutes for auth requests, 10 minutes for auth responses).


## Privacy Considerations {#privacy-considerations}

### Provider DID Non-Disclosure {#provider-non-disclosure}

The user's real DID (the `iss` field in the auth response) is only revealed inside
the encrypted auth response. The Connect Server and any network observer cannot
determine which DID the user selected.

### Delegate DID Unlinkability {#delegate-unlinkability}

Each connect session creates a fresh Delegate DID. Different applications connected
to the same user's DID receive different Delegate DIDs. This prevents colluding
applications from correlating a user across services by comparing Delegate DIDs.

However, the Delegate DIDs all receive grants from the same Provider DID, so
correlation is possible at the DWN level if two applications access the same DWN
and can observe each other's grant records.

### Connect Server Data Minimization {#data-minimization}

Connect Servers ****SHOULD**** minimize the metadata they retain:

- ****MUST NOT**** log the contents of encrypted blobs.
- ****SHOULD NOT**** retain metadata (source IPs, timestamps) beyond what is
  necessary for rate limiting and abuse prevention.
- ****SHOULD**** delete all session data after the flow completes or the TTL expires.


## Normative References

[[spec]]
~ [DWN Specification](https://github.com/enboxorg/dwn-spec). Enbox.

[[spec]]
~ [DWN Transport Protocol Specification](https://github.com/enboxorg/dwn-transport-spec). Enbox.

[[spec:RFC9126]]
~ [OAuth 2.0 Pushed Authorization Requests](https://datatracker.ietf.org/doc/html/rfc9126). T. Lodderstedt, B. Campbell, N. Sakimura, D. Tonge, F. Skokan. September 2021.

[[spec:RFC7516]]
~ [JSON Web Encryption (JWE)](https://datatracker.ietf.org/doc/html/rfc7516). M. Jones, J. Hildebrand. May 2015.

[[spec:RFC7748]]
~ [Elliptic Curves for Security](https://datatracker.ietf.org/doc/html/rfc7748). A. Langley, M. Hamburg, S. Turner. January 2016.

[[spec:RFC5869]]
~ [HMAC-based Extract-and-Expand Key Derivation Function (HKDF)](https://datatracker.ietf.org/doc/html/rfc5869). H. Krawczyk, P. Eronen. May 2010.

[[spec:RFC7515]]
~ [JSON Web Signature (JWS)](https://datatracker.ietf.org/doc/html/rfc7515). M. Jones, J. Bradley, N. Sakimura. May 2015.

[[spec:RFC2119]]
~ [Key words for use in RFCs to Indicate Requirement Levels](https://datatracker.ietf.org/doc/html/rfc2119). S. Bradner. March 1997.
