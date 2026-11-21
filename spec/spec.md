
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

The Enbox Connect Protocol is how an application asks a user's wallet for
access to the user's [Decentralized Web Node](https://dwn-spec.pages.dev), and
how the wallet answers.

An application can ask for three things:

1. **Access.** The wallet installs the DWN modules the application needs,
   and issues time-limited permission grants to a *delegate* DID that the
   application controls. It also delivers the keys the delegate needs to read
   encrypted records.
2. **Renewal.** The wallet issues fresh grants to a delegate the application
   already has.
3. **Proof of identity.** The wallet proves, to a service the application
   names, that the user controls their DID.

The request and the response are signed and end-to-end encrypted. They travel
by one of two routes: through a **relay**, which lets the wallet be on another
device and be reached by a QR code or a link; or directly between two browser
windows, with **`postMessage`**.

## Status of This Document

Enbox Connect Protocol is a _DRAFT_ specification under development by
[Enbox](https://enbox.org). Earlier drafts were titled "Web5 Connect
Protocol". It is a companion to the
[DWN specification](https://dwn-spec.pages.dev) and the
[DWN Transport Protocol](https://dwn-transport-spec.pages.dev),
and describes the protocol as implemented by the reference implementation,
[enboxorg/enbox](https://github.com/enboxorg/enbox).

The specification will be updated to incorporate feedback. We encourage reviewers
to submit [GitHub Issues](https://github.com/enboxorg/web5-connect-spec/issues)
as the means by which to communicate feedback and contributions.

The key words "****MUST****", "****MUST NOT****", "****REQUIRED****",
"****SHALL****", "****SHALL NOT****", "****SHOULD****", "****SHOULD NOT****",
"****RECOMMENDED****", "****MAY****", and "****OPTIONAL****" in this document
are to be interpreted as described in [[spec:RFC2119]].

Where this document gives a number as what "the reference" client, wallet, or
relay does, the number is a default of the reference implementation and not a
requirement.

## Terminology

[[def:Client, Application]]
~ The application that sends a request.

[[def:Provider, Wallet]]
~ The software that holds the user's DIDs and keys. It shows the user the
request, and carries it out if the user approves.

[[def:Owner DID]]
~ The user's DID: the one whose node the application wants to use. In messages
it is called `providerDid`.

[[def:Delegate DID, Delegate]]
~ A DID that acts for the user within the limits of the grants it holds. The
wallet can create one for the application, or the application can supply one
of its own and keep the keys to itself.

[[def:Relay, Connect Server]]
~ An HTTP service that holds an encrypted request until the wallet collects
it, and an encrypted response until the application collects it. It cannot
read either.

[[def:Permission Grant, Permission Grants]]
~ A DWN record by which a DID authorizes another DID to act on its node, for a
stated scope and time. Defined in the
[DWN specification](https://dwn-spec.pages.dev).

[[def:Delegated Grant]]
~ A [[ref:Permission Grant]] with `delegated: true`. Its holder signs messages
*as* the grantor, rather than in its own name.

[[def:Session]]
~ The grants issued by one approval. They share an identifier and an expiry.

[[def:Grant Key]]
~ A record that gives the holder of a grant a key for reading encrypted
records. Defined in the DWN specification.

[[def:Owner Authentication Proof]]
~ A signature by the [[ref:Owner DID]] over a relying service's challenge and
over the request, proving who approved it. See
[Owner Authentication Proofs](#owner-authentication-proofs).

[[def:Wallet URI]]
~ The link, often shown as a QR code, that hands a request to a wallet on the
Relay Transport. It carries the request's location and its decryption key in
the URI fragment.

[[def:PIN]]
~ A short numeric code that the [[ref:Wallet]] shows the user on the Relay
Transport and the user types into the [[ref:Application]]. It is mixed into
the key that encrypts the response, so a response cannot be opened, or forged,
without it. A wallet ****SHOULD**** use **4 decimal digits**, zero-padded: that
is what the reference browser client prompts for. The key derivation itself
accepts any UTF-8 string.

[[def:DENIED Token]]
~ The literal string `DENIED`, sent in place of a response when the user
refuses the request.

## Overview

### Request Types

A request's `requestType` says what is being asked.

| Request Type | Wallet Behavior | Response |
|---|---|---|
| `connect`, or absent | Installs modules and issues grants to a delegate | A [`ConnectResponse`](#connectresponse) |
| `refresh` | Issues new grants to the delegate named in the request. It ****MUST NOT**** create a delegate | A `ConnectResponse` without `delegatePortableDid` |
| `authenticate` | Only proves the user's identity. It issues no grants and creates no delegate | A [`ConnectAuthenticationResponse`](#connectauthenticationresponse) |

A `connect` or `refresh` request may also ask for an
[[ref:Owner Authentication Proof]], and then receives one together with its
grants.

### The Relay Transport

The wallet may be on another device. The two sides share nothing but a relay
and what passes between them through the user: a QR code one way, a
[[ref:PIN]] the other.

1. The application creates a throwaway signing DID, a throwaway X25519 key
   pair for the response, a `nonce`, a `state`, and a random 32-byte key. It
   signs the request, encrypts it with the key, and stores it on the relay
   ([`POST /connect/par`](#1-store-a-request-post-connectpar)).
2. It shows the user a [[ref:Wallet URI]] containing the request's location
   and the key, and starts polling the relay for a response.
3. The wallet opens the URI, fetches the request from the relay, decrypts it,
   and checks its signature. Fetching removes the request from the relay.
4. The wallet shows the request to the user. If the user approves, it carries
   the request out (see [Provider Processing](#provider-processing)).
5. The wallet shows the user a [[ref:PIN]], encrypts the response to the
   application's key and the PIN together, and stores it on the relay. If the
   user refuses, it stores `DENIED` instead.
6. The application collects the response, asks the user for the PIN, and
   decrypts and verifies the response. It tells the relay that it has, so the
   wallet can show the user that the connection was made.

The PIN ties the response to the wallet the user is looking at. Anyone who
saw the QR code can read the request, and so learn where a response should
go; without the PIN, they cannot make one the application will accept.

### The Popup Transport

When the wallet is a web application in the same browser, no relay is needed.
The application opens the wallet in a popup, and the two exchange four
messages with `postMessage`. The browser tells each side the other's origin,
which stands in for the PIN.

1. The application opens the wallet's page in a popup.
2. The wallet creates a throwaway X25519 key pair and posts its public key to
   the window that opened it.
3. The application encrypts the request to that key and posts it.
4. The wallet decrypts and verifies the request, shows it to the user, carries
   it out, and posts the encrypted response, or `DENIED`.
5. The application decrypts and verifies the response, and posts an
   acknowledgement so that the wallet can close.

See [Browser Popup `postMessage` Transport](#browser-popup-postmessage-transport).

## The Relay

A [[ref:Relay]] is a small HTTP service with seven routes under one base URL,
here written `/connect`. A DWN server may host one (see the
[DWN Transport Protocol](https://dwn-transport-spec.pages.dev)).

It keeps four kinds of entries, each for **600 seconds**: an encrypted
request, under a request ID it assigns; a note that a request has been
fetched; an encrypted response, under the request's `state`; and a note that
a response has been received. It sees nothing but ciphertext, the literal
`DENIED`, and those two notes.

A relay ****MUST**** store an entry durably before it answers `201`, and
****MUST**** hand out each request and each response at most once.

A relay ****SHOULD**** also:

- make reading and removing a request or response one atomic step, so that two readers at the same moment cannot both receive it;
- accept a repeated `POST /connect/callback` or `POST /connect/complete` for the same `state`;
- reject an empty `request`, `id_token`, or `state`, and a malformed body, with `400`;
- send `Cache-Control: no-store` on every response; and
- limit the size of the bodies it accepts.

A relay ****MUST**** allow cross-origin requests from any origin (`Access-Control-Allow-Origin: *`, `Access-Control-Allow-Methods: GET, POST, OPTIONS`, `Access-Control-Allow-Headers` permitting `Content-Type` — such as `Content-Type` or `*` — and answering `OPTIONS` with `204`), and its routes are subject to the hosting server's rate limits.

### 1. Store a Request (`POST /connect/par`)

Used by the application.

```http
POST /connect/par HTTP/1.1
Content-Type: application/json

{ "request": "<the encrypted request>" }
```

| Response | Body |
|---|---|
| `201` | `{ "request_uri": "https://relay.example/connect/authorize/<requestId>.jwt", "expires_in": 600 }` |
| `400` | `{ "ok": false, "status": { "code": 400, "message": "Bad Request: Missing 'request' parameter" } }` |

`request` is the encrypted request, a string. `request_uri` is where a wallet
fetches it (the reference client extracts `requestId` by matching
`/connect/authorize/<requestId>.jwt`, so relays ****SHOULD**** preserve that
path structure).

### 2. Fetch a Request (`GET /connect/authorize/:requestId.jwt`)

Used by the wallet.

| Response | Body |
|---|---|
| `200` | The encrypted request, with `Content-Type: application/jwt` |
| `404` | `{ "ok": false, "status": { "code": 404, "message": "Not Found" } }` |

A request can be fetched once. Fetching removes it, and marks it as claimed.

### 3. Ask Whether a Request Was Fetched (`GET /connect/status/:requestId`)

Used by the application, so that it can show that a wallet has picked the
request up, and stop refreshing its QR code. `requestId` is the last path
segment of `request_uri`, without the `.jwt` suffix.

Always answers `200`, with `{ "claimed": true }` or `{ "claimed": false }`.
An unknown or expired request is `false`.

### 4. Store a Response (`POST /connect/callback`)

Used by the wallet. This is the `callbackUrl` of the request's `reply`.

The body is either form-encoded (`Content-Type: application/x-www-form-urlencoded`,
`id_token=<response>&state=<state>`) or JSON (`Content-Type: application/json`,
`{ "id_token": "<response>", "state": "<state>" }`). `id_token` is the
encrypted response, or `DENIED`.

| Response | Body |
|---|---|
| `201` | `{ "ok": true, "status": { "code": 201, "message": "Created" } }` |
| `400` | `{ "ok": false, "status": { "code": 400, "message": "Bad Request" } }` |

### 5. Fetch a Response (`GET /connect/token/:state.jwt`)

Used by the application, which polls it.

| Response | Meaning |
|---|---|
| `200` | The body is the encrypted response, or `DENIED`, with `Content-Type: application/jwt`. A response can be fetched once; fetching removes it |
| `204` | There is no response yet, or it has expired, or it was already fetched |

A relay answers `204`, never `404`, when it has nothing. An application keeps
polling on `204`, and ****SHOULD**** keep polling on any other unsuccessful
status too, until its own timeout.

### 6. Report That a Response Was Received (`POST /connect/complete`)

Used by the application, once, as soon as it has opened and verified the
response. It is best-effort: nothing depends on it but what the wallet shows.

```http
POST /connect/complete HTTP/1.1
Content-Type: application/json

{ "state": "BuIh_cj2gukaJVU4JmkJ5Q" }
```

| Response | Body |
|---|---|
| `201` | `{ "ok": true, "status": { "code": 201, "message": "Created" } }` |
| `400` | `{ "ok": false, "status": { "code": 400, "message": "Bad Request: Missing 'state' parameter" } }` |

### 7. Ask Whether a Response Was Received (`GET /connect/complete/:state`)

Used by the wallet, so that it can show the user that the application is
connected. Its URL is the request's `callbackUrl` with the last segment
replaced by `complete/<state>`, the `state` percent-encoded.

Always answers `200`, with `{ "completed": true }` or
`{ "completed": false }`. The reference wallet polls once a second for up to
60 seconds.

### Finding a Wallet's Relay

An application needs to know which relay a wallet uses.

**The wallet's well-known document.** A web wallet publishes

```
GET https://<wallet origin>/.well-known/enbox-connect
```

```json
{ "connectServerUrl": "https://relay.example/connect" }
```

as JSON, readable from any origin. An application ****SHOULD**** fetch it
with a short timeout (the reference clients use 5 to 6 seconds) and
****MUST**** accept only an `http:` or `https:` URL.

**The user's wallets.** A DID can publish which web wallets it uses, with the
module `https://identity.foundation/modules/connect`. It holds at most one
`wallet` record, readable by anyone, whose data is
`{ "webWallets": ["https://wallet.example"] }`.

```json
{
  "module": "https://identity.foundation/modules/connect",
  "published": true,
  "types": {
    "wallet": {
      "schema": "https://identity.foundation/schemas/connect/wallet",
      "dataFormats": ["application/json"]
    }
  },
  "structure": {
    "wallet": {
      "$recordLimit": { "max": 1 },
      "$actions": [ { "who": "anyone", "can": ["read"] } ]
    }
  }
}
```

::: note
`dwn://connect` is unrelated to this protocol. It is how a desktop
application announces a local DWN server; see the DWN Transport Protocol.
:::

## Requests and Responses

### `ConnectRequest`

```json
{
  "clientDid": "did:jwk:eyJrdHkiOiJPS1AiLCJjcnYiOiJFZDI1NTE5IiwieCI6Ik56LWF4V3M3NjNZcW1NUVZXUFdOVzJ2eWlfMWVLQWFRaWNzN2YtZ0xaVUEifQ",
  "appName": "Notes",
  "appIcon": "https://notes.example/icon.png",
  "applicationId": "https://notes.example",
  "clientMetadata": {
    "origin": "https://notes.example",
    "userAgent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7)",
    "platform": "macOS",
    "language": "en-US",
    "languages": ["en-US", "en"],
    "timezone": "America/New_York"
  },
  "authentication": {
    "version": 1,
    "audience": "https://notes.example",
    "challenge": "OzcBUfNeDjydxX9F3oN2bmKdfgAHyzLa"
  },
  "permissionRequests": [
    {
      "moduleDefinition": {
        "module": "https://notes.example/modules/notes/v1",
        "published": true,
        "types": {
          "note": {
            "schema": "https://notes.example/schemas/note/v1",
            "dataFormats": ["application/json", "text/markdown"]
          }
        },
        "structure": {
          "note": {}
        }
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
  "requestType": "connect",
  "expectedProviderDid": "did:dht:i9xkp8ddcbcg8jwq54ox699wuzxyifsqx4jru45zodqu453ksz6y",
  "supportedDidMethods": ["did:dht", "did:jwk"],
  "nonce": "N8WrKpg8XNb9Wh_QuWlHbw",
  "state": "BuIh_cj2gukaJVU4JmkJ5Q",
  "responseKey": {
    "kty": "OKP",
    "crv": "X25519",
    "x": "Jm8Bu6RVgHAdojIx4k68U42MgRrlEYwDBGhNXHDdFWc"
  },
  "reply": {
    "mode": "direct_post",
    "callbackUrl": "https://dwn.example/connect/callback"
  }
}
```

- `clientDid` — ****REQUIRED****. A `did:jwk` with an Ed25519 key, created for this request. It ****MUST**** be the signer of the inner JWT.
- `appName` — ****REQUIRED****. A display name.
- `appIcon`, `applicationId`, `clientMetadata` — ****OPTIONAL****. Self-reported display hints. A wallet ****MUST NOT**** treat them as verified.
- `authentication` — ****OPTIONAL****; ****REQUIRED**** when `requestType` is `"authenticate"`. Asks for an [[ref:Owner Authentication Proof]]. An object with exactly:
  - `version` — the number `1`.
  - `audience` — an `https:` URL of at most 2048 characters with no user information, query, or fragment, written in its normalized form (either the bare origin, or the form a URL parser serializes it to).
  - `challenge` — 22 to 128 base64url characters, issued by the relying service for one use.
- `permissionRequests` — ****REQUIRED****. An array of `{ moduleDefinition, permissionScopes }` objects with no other properties. `moduleDefinition` is a DWN module definition; `permissionScopes` is an array of DWN permission scopes for that module (see [Permission Model](#permission-model)). It is `[]` when `requestType` is `"authenticate"`.
- `requestedSessionTtlSeconds` — ****OPTIONAL****. The grant lifetime the application would like, in seconds. Advisory. It ****MUST**** be a finite number that rounds down to at least 1. Not allowed with `"authenticate"`.
- `delegateDid` — ****OPTIONAL****. A DID the application already controls and wants the grants issued to. ****REQUIRED**** for `"refresh"`; not allowed with `"authenticate"`. When omitted on `"connect"`, the wallet creates a new delegate DID and returns its keys in `delegatePortableDid`.
- `requestType` — ****OPTIONAL****. `"connect"` (the default), `"refresh"`, or `"authenticate"`.
- `expectedProviderDid` — ****OPTIONAL****. The owner DID the application expects. A wallet ****MUST**** refuse to answer as any other DID.
- `supportedDidMethods` — ****REQUIRED****. The DID methods the application accepts for the owner DID, such as `["did:dht", "did:jwk"]`.
- `nonce`, `state` — ****REQUIRED****. Random strings; the reference client uses 16 random bytes, base64url-encoded. The response echoes both. `state` also names the response on the relay, so on the Relay Transport it ****MUST**** be URL-path-safe (for example, unpadded base64url, since `GET /connect/token/:state.jwt` does not percent-encode or decode `:state`).
- `responseKey` — ****REQUIRED****. A fresh X25519 public key, as a JWK with no private part. The response is encrypted to it.
- `reply` — ****REQUIRED****. `{ "mode": "direct_post", "callbackUrl": "<url>" }` on the Relay Transport, or `{ "mode": "post_message" }` on the popup transport.

A request that contains `authentication` is validated more strictly, because
the proof covers the whole request: `clientDid` ****MUST**** be a bare `did:jwk`
(no path, query, or fragment); `appName` is 1 to 256 characters; `nonce` and
`state` are 22 to 128 base64url characters; `permissionRequests` has at most 32
entries; `supportedDidMethods` has 1 to 8 distinct entries of the form
`did:<method>`; `callbackUrl`, when present, is 1 to 2048 characters; and the
request in canonical form is at most 262,144 bytes (see
[Owner Authentication Proofs](#owner-authentication-proofs)).

### `ConnectResponse`

A delegation response ****MUST NOT**** contain `responseType`.

```json
{
  "providerDid": "did:dht:i9xkp8ddcbcg8jwq54ox699wuzxyifsqx4jru45zodqu453ksz6y",
  "delegateDid": "did:jwk:eyJhbGciOiJFZERTQSIsImNydiI6IkVkMjU1MTkiLCJraWQiOiJkZWxlZ2F0ZS1lZDI1NTE5LTAiLCJrdHkiOiJPS1AiLCJ4IjoiTHBmVjBBeVhacEowSVZkbzY4eUJwckFNUmpQUG5acDkydzRpcXUyaHZmZyJ9",
  "aud": "did:jwk:eyJrdHkiOiJPS1AiLCJjcnYiOiJFZDI1NTE5IiwieCI6Ik56LWF4V3M3NjNZcW1NUVZXUFdOVzJ2eWlfMWVLQWFRaWNzN2YtZ0xaVUEifQ",
  "iat": 1780272000,
  "exp": 1780272600,
  "nonce": "N8WrKpg8XNb9Wh_QuWlHbw",
  "state": "BuIh_cj2gukaJVU4JmkJ5Q",
  "delegateGrants": [
    {
      "recordId": "bafyreig3v56ki2375j2s4m5u7u6n4f3q2v5w7x8y9z2a3b4c5d6e7f8g9h",
      "contextId": "bafyreig3v56ki2375j2s4m5u7u6n4f3q2v5w7x8y9z2a3b4c5d6e7f8g9h",
      "descriptor": {
        "interface": "Records",
        "method": "Write",
        "module": "https://identity.foundation/dwn/permissions",
        "modulePath": "grant",
        "recipient": "did:jwk:eyJhbGciOiJFZERTQSIsImNydiI6IkVkMjU1MTkiLCJraWQiOiJkZWxlZ2F0ZS1lZDI1NTE5LTAiLCJrdHkiOiJPS1AiLCJ4IjoiTHBmVjBBeVhacEowSVZkbzY4eUJwckFNUmpQUG5acDkydzRpcXUyaHZmZyJ9",
        "tags": {
          "module": "https://notes.example/modules/notes/v1"
        },
        "dataCid": "bafkreihdwdcefgh4dqkjv67uzcmw7ojee6xedzdetojuzjevtenxquvyku",
        "dataSize": 302,
        "dataFormat": "application/json",
        "dateCreated": "2026-06-01T00:00:00.000000Z",
        "messageTimestamp": "2026-06-01T00:00:00.000000Z"
      },
      "encodedData": "eyJkYXRlRXhwaXJlcyI6IjIwMjYtMDYtMDJUMDA6MDA6MDAuMDAwMDAwWiIsInNjb3BlIjp7ImludGVyZmFjZSI6IlJlY29yZHMiLCJtZXRob2QiOiJSZWFkIiwibW9kdWxlIjoiaHR0cHM6Ly9ub3Rlcy5leGFtcGxlL21vZHVsZXMvbm90ZXMvdjEifSwiZGVsZWdhdGVkIjp0cnVlLCJjb25uZWN0U2Vzc2lvbiI6eyJpZCI6InNlc3NfOGYzYTFjMmUiLCJjcmVhdGVkQXQiOiIyMDI2LTA2LTAxVDAwOjAwOjAwLjAwMDAwMFoiLCJleHBpcmVzQXQiOiIyMDI2LTA2LTAyVDAwOjAwOjAwLjAwMDAwMFoiLCJhcHBOYW1lIjoiTm90ZXMifX0",
      "authorization": {
        "signature": {
          "payload": "eyJkZXNjcmlwdG9yQ2lkIjoiYmFmeXJlaWczdjU2a2kifQ",
          "signatures": [
            {
              "protected": "eyJhbGciOiJFZERTQSIsImtpZCI6ImRpZDpkaHQ6aTl4a3A4ZGRjYmNnOGp3cTU0b3g2OTl3dXp4eWlmc3F4NGpydTQ1em9kcXU0NTNrc3o2eSMwIn0",
              "signature": "MEUCIQDf1a2b3c4d5e6f7g8h9i0j1k2l3m4n5o6p7q8r9s0t1u2v3w"
            }
          ]
        }
      }
    }
  ],
  "delegatePortableDid": {
    "uri": "did:jwk:eyJhbGciOiJFZERTQSIsImNydiI6IkVkMjU1MTkiLCJraWQiOiJkZWxlZ2F0ZS1lZDI1NTE5LTAiLCJrdHkiOiJPS1AiLCJ4IjoiTHBmVjBBeVhacEowSVZkbzY4eUJwckFNUmpQUG5acDkydzRpcXUyaHZmZyJ9",
    "document": {
      "@context": ["https://www.w3.org/ns/did/v1"],
      "id": "did:jwk:eyJhbGciOiJFZERTQSIsImNydiI6IkVkMjU1MTkiLCJraWQiOiJkZWxlZ2F0ZS1lZDI1NTE5LTAiLCJrdHkiOiJPS1AiLCJ4IjoiTHBmVjBBeVhacEowSVZkbzY4eUJwckFNUmpQUG5acDkydzRpcXUyaHZmZyJ9",
      "verificationMethod": [
        {
          "id": "did:jwk:eyJhbGciOiJFZERTQSIsImNydiI6IkVkMjU1MTkiLCJraWQiOiJkZWxlZ2F0ZS1lZDI1NTE5LTAiLCJrdHkiOiJPS1AiLCJ4IjoiTHBmVjBBeVhacEowSVZkbzY4eUJwckFNUmpQUG5acDkydzRpcXUyaHZmZyJ9#0",
          "type": "JsonWebKey",
          "controller": "did:jwk:eyJhbGciOiJFZERTQSIsImNydiI6IkVkMjU1MTkiLCJraWQiOiJkZWxlZ2F0ZS1lZDI1NTE5LTAiLCJrdHkiOiJPS1AiLCJ4IjoiTHBmVjBBeVhacEowSVZkbzY4eUJwckFNUmpQUG5acDkydzRpcXUyaHZmZyJ9",
          "publicKeyJwk": {
            "alg": "EdDSA",
            "crv": "Ed25519",
            "kid": "delegate-ed25519-0",
            "kty": "OKP",
            "x": "LpfV0AyXZpJ0IVdo68yBprAMRjPPnZp92w4iqu2hvfg"
          }
        }
      ],
      "authentication": ["did:jwk:eyJhbGciOiJFZERTQSIsImNydiI6IkVkMjU1MTkiLCJraWQiOiJkZWxlZ2F0ZS1lZDI1NTE5LTAiLCJrdHkiOiJPS1AiLCJ4IjoiTHBmVjBBeVhacEowSVZkbzY4eUJwckFNUmpQUG5acDkydzRpcXUyaHZmZyJ9#0"],
      "assertionMethod": ["did:jwk:eyJhbGciOiJFZERTQSIsImNydiI6IkVkMjU1MTkiLCJraWQiOiJkZWxlZ2F0ZS1lZDI1NTE5LTAiLCJrdHkiOiJPS1AiLCJ4IjoiTHBmVjBBeVhacEowSVZkbzY4eUJwckFNUmpQUG5acDkydzRpcXUyaHZmZyJ9#0"],
      "capabilityInvocation": ["did:jwk:eyJhbGciOiJFZERTQSIsImNydiI6IkVkMjU1MTkiLCJraWQiOiJkZWxlZ2F0ZS1lZDI1NTE5LTAiLCJrdHkiOiJPS1AiLCJ4IjoiTHBmVjBBeVhacEowSVZkbzY4eUJwckFNUmpQUG5acDkydzRpcXUyaHZmZyJ9#0"],
      "capabilityDelegation": ["did:jwk:eyJhbGciOiJFZERTQSIsImNydiI6IkVkMjU1MTkiLCJraWQiOiJkZWxlZ2F0ZS1lZDI1NTE5LTAiLCJrdHkiOiJPS1AiLCJ4IjoiTHBmVjBBeVhacEowSVZkbzY4eUJwckFNUmpQUG5acDkydzRpcXUyaHZmZyJ9#0"],
      "keyAgreement": ["did:jwk:eyJhbGciOiJFZERTQSIsImNydiI6IkVkMjU1MTkiLCJraWQiOiJkZWxlZ2F0ZS1lZDI1NTE5LTAiLCJrdHkiOiJPS1AiLCJ4IjoiTHBmVjBBeVhacEowSVZkbzY4eUJwckFNUmpQUG5acDkydzRpcXUyaHZmZyJ9#0"]
    },
    "metadata": {},
    "privateKeys": [
      {
        "alg": "EdDSA",
        "crv": "Ed25519",
        "d": "blgEg4ByowZHhjyf72q7bE1zAmYrbe4moVc0TKnVhPA",
        "kid": "delegate-ed25519-0",
        "kty": "OKP",
        "x": "LpfV0AyXZpJ0IVdo68yBprAMRjPPnZp92w4iqu2hvfg"
      },
      {
        "crv": "X25519",
        "d": "wmbgMZdCwzfyrLZvTqbt_jS1DwHu1SXLWytmshpt0Ss",
        "kid": "delegate-x25519-0",
        "kty": "OKP",
        "x": "Aoww6kbdmqQ9gStDZzjE56lta7sqfXbx89qyDZ6MX9s"
      }
    ]
  },
  "sessionRevocations": [
    {
      "grantId": "bafyreig3v56ki2375j2s4m5u7u6n4f3q2v5w7x8y9z2a3b4c5d6e7f8g9h",
      "revocationGrantId": "bafyreih4w67lj3486k3t5n6v8v7o5g4r3w6x8y9z0a3b4c5d6e7f8g9h0i"
    }
  ],
  "authenticationProof": "eyJhbGciOiJFZERTQSIsImtpZCI6ImRpZDpkaHQ6aTl4a3A4ZGRjYmNnOGp3cTU0b3g2OTl3dXp4eWlmc3F4NGpydTQ1em9kcXU0NTNrc3o2eSMwIiwidHlwIjoiZW5ib3gtY29ubmVjdC1vd25lci1hdXRoZW50aWNhdGlvbitqd3QifQ.eyJhdWQiOiJodHRwczovL25vdGVzLmV4YW1wbGUiLCJleHAiOjE3ODAyNzIzMDAsImlhdCI6MTc4MDI3MjAwMCwiaXNzIjoiZGlkOmRodDppOXhrcDhkZGNiY2c4andxNTRveDY5OXd1enh5aWZzcXg0anJ1NDV6b2RxdTQ1M2tzejZ5Iiwibm9uY2UiOiJPemNCVWZOZURqeWR4WDlGM29OMmJtS2RmZ0FIeXpMYSIsInB1cnBvc2UiOiJlbmJveC1jb25uZWN0LWF1dGhlbnRpY2F0aW9uIiwicmVxdWVzdF9oYXNoIjoieUZuZzlHOTMtTTRzLWNXNnZZakdKMkRtSDVMeElDNUt5SVkxazh4MG9CMCIsInN1YiI6ImRpZDpkaHQ6aTl4a3A4ZGRjYmNnOGp3cTU0b3g2OTl3dXp4eWlmc3F4NGpydTQ1em9kcXU0NTNrc3o2eSJ9.MDEyMzQ1Njc4OWFiY2RlZjAxMjM0NTY3ODlhYmNkZWYwMTIzNDU2Nzg5YWJjZGVmMDEyMzQ1Njc4OWFiY2RlZg"
}
```

- `exp` is `iat + 600`.
- `delegateDid` is the DID the grants were issued to. When the request supplied `delegateDid`, it ****MUST**** be that DID.
- `delegatePortableDid` is present exactly when the wallet created the delegate DID. It is a portable DID object `{ uri, document, metadata, privateKeys }` containing the delegate's DID document (a single `#0` Ed25519 `JsonWebKey` verification method referenced across all verification relationships) and two private JWKs in `privateKeys`: the delegate's `Ed25519` signing key and the `X25519` private key converted from it. It ****MUST**** be absent when the request supplied `delegateDid`.
- `delegateGrants` contains every grant issued, as `RecordsWrite` messages with their data in `encodedData`: the session grants followed by their companion revocation grants (abbreviated to one grant in the example above for brevity).
- `sessionRevocations` pairs each session grant with its revocation grant (see [Provider Processing](#provider-processing)).
- `authenticationProof` is present exactly when the request contained `authentication`.

### `ConnectAuthenticationResponse`

An authentication-only response has exactly these eight properties, and no
delegation properties:

```json
{
  "responseType": "authentication",
  "providerDid": "did:dht:i9xkp8ddcbcg8jwq54ox699wuzxyifsqx4jru45zodqu453ksz6y",
  "aud": "did:jwk:eyJrdHkiOiJPS1AiLCJjcnYiOiJFZDI1NTE5IiwieCI6Ik56LWF4V3M3NjNZcW1NUVZXUFdOVzJ2eWlfMWVLQWFRaWNzN2YtZ0xaVUEifQ",
  "iat": 1780272000,
  "exp": 1780272600,
  "nonce": "N8WrKpg8XNb9Wh_QuWlHbw",
  "state": "BuIh_cj2gukaJVU4JmkJ5Q",
  "authenticationProof": "eyJhbGciOiJFZERTQSIsImtpZCI6ImRpZDpkaHQ6aTl4a3A4ZGRjYmNnOGp3cTU0b3g2OTl3dXp4eWlmc3F4NGpydTQ1em9kcXU0NTNrc3o2eSMwIiwidHlwIjoiZW5ib3gtY29ubmVjdC1vd25lci1hdXRoZW50aWNhdGlvbitqd3QifQ.eyJhdWQiOiJodHRwczovL25vdGVzLmV4YW1wbGUiLCJleHAiOjE3ODAyNzIzMDAsImlhdCI6MTc4MDI3MjAwMCwiaXNzIjoiZGlkOmRodDppOXhrcDhkZGNiY2c4andxNTRveDY5OXd1enh5aWZzcXg0anJ1NDV6b2RxdTQ1M2tzejZ5Iiwibm9uY2UiOiJPemNCVWZOZURqeWR4WDlGM29OMmJtS2RmZ0FIeXpMYSIsInB1cnBvc2UiOiJlbmJveC1jb25uZWN0LWF1dGhlbnRpY2F0aW9uIiwicmVxdWVzdF9oYXNoIjoieUZuZzlHOTMtTTRzLWNXNnZZakdKMkRtSDVMeElDNUt5SVkxazh4MG9CMCIsInN1YiI6ImRpZDpkaHQ6aTl4a3A4ZGRjYmNnOGp3cTU0b3g2OTl3dXp4eWlmc3F4NGpydTQ1em9kcXU0NTNrc3o2eSJ9.MDEyMzQ1Njc4OWFiY2RlZjAxMjM0NTY3ODlhYmNkZWYwMTIzNDU2Nzg5YWJjZGVmMDEyMzQ1Njc4OWFiY2RlZg"
}
```

## Cryptography

A request or response is protected twice. It is first **signed**, as a JWT, so
that its recipient knows it was not altered. The JWT is then **encrypted**, as
a JWE, so that nobody else can read it.

### Signing

The payload is signed as a compact JWS ([[spec:RFC7515]]) with the header

```json
{
  "alg": "EdDSA",
  "kid": "did:jwk:eyJrdHkiOiJPS1AiLCJjcnYiOiJFZDI1NTE5IiwieCI6Ik56LWF4V3M3NjNZcW1NUVZXUFdOVzJ2eWlfMWVLQWFRaWNzN2YtZ0xaVUEifQ#0",
  "typ": "JWT"
}
```

- The signer is always a `did:jwk` with an Ed25519 key. A verifier
  ****MUST**** require `alg` to be `EdDSA` and `kid` to name a `did:jwk`, and
  takes the public key from the DID itself. Nothing is resolved over a
  network.
- A **request** is signed by the DID in its `clientDid`. A wallet ****MUST****
  reject a request whose signer is any other.
- A **response** is signed by the delegate, when the wallet created it; and
  otherwise by a `did:jwk` that the wallet creates for the one response.

The signature on a response shows only that the response is intact. It is not
by the [[ref:Owner DID]], and proves nothing about who approved the request.
What makes a response trustworthy is how it is encrypted, described next.
When an application needs proof of who approved, it asks for an
[[ref:Owner Authentication Proof]].

### Encryption

Every envelope is a 5-part compact JWE ([[spec:RFC7516]])
`<protected>..<iv>.<ciphertext>.<tag>` whose content is encrypted with
XChaCha20-Poly1305, `"enc": "XC20P"`:

- a 256-bit content encryption key;
- a random 24-byte nonce, in the JWE's initialization vector (`<iv>`);
- the ciphertext without authentication tag (`<ciphertext>`);
- the detached 16-byte Poly1305 authentication tag (`<tag>`);
- the ASCII bytes of the base64url-encoded protected header (`<protected>`) as additional authenticated data; and
- an empty encrypted-key part, since no key is wrapped.

The header always has `"cty": "JWT"`, and a `typ` of `enbox-connect-req` for a
request or `enbox-connect-res` for a response.

How the content encryption key is agreed depends on the envelope:

| Envelope | Algorithm | Content Encryption Key |
|---|---|---|
| Request on the Relay Transport | `dir` | The random 32-byte key in the [[ref:Wallet URI]] |
| Request on the popup transport | `ECDH-ES` | Derived from the wallet's X25519 key, and bound to the wallet's origin |
| Any response | `ECDH-ES` | Derived from the request's `responseKey`, bound to the request's `state`, and on the Relay Transport to the [[ref:PIN]] |

A recipient ****MUST**** check, before anything else, that `alg` and `enc` are
exactly the ones this table gives for the envelope it expects, and that `typ`
and `cty` are as stated. It ****MUST**** reject any other envelope.

#### Deriving a Key with `ECDH-ES`

The sender creates a fresh X25519 key pair ([[spec:RFC7748]]) for the one
envelope, and puts the public key in the header as `epk`
(`{ "kty": "OKP", "crv": "X25519", "x": "<base64url>" }`, with no private or
extra parameters). Both sides compute the X25519 shared secret `Z`, the sender
from its private key and the recipient's public key, the recipient from its
private key and `epk`.

An implementation ****MUST**** reject a `Z` that is all zeros, which is what a
malicious public key of low order produces. It ****MUST**** also refuse to
combine the two halves of the same key pair.

The key is then derived with the Concat KDF of [[spec:RFC7518]], section
4.6.2:

```
key = SHA-256(
  0x00000001 || Z ||
  uint32be(5) || "XC20P" ||
  uint32be(len(apu)) || apu ||
  uint32be(len(apv)) || apv ||
  uint32be(256)
)
```

`apu` and `apv` here are the *decoded* bytes of the header parameters of the
same names. An absent parameter is the empty string, and contributes only
`uint32be(0)`.

- **A popup request** has `apv`, the base64url encoding of the wallet's
  origin (for example `https://wallet.example`), and no `apu`. The wallet
  ****MUST**** check that `apv` is its own origin. A request encrypted for one
  wallet is thereby useless to another.
- **A response** has `apu`, the base64url encoding of the request's `state`,
  and no `apv`. The application ****MUST**** check that `apu` is the `state`
  it sent.

```json
{
  "alg": "ECDH-ES",
  "enc": "XC20P",
  "typ": "enbox-connect-res",
  "cty": "JWT",
  "apu": "QnVJaF9jajJndWthSlZVNEpta0o1UQ",
  "epk": {
    "kty": "OKP",
    "crv": "X25519",
    "x": "XDGgBcvJe7NdqS18OGAyK3w4XMvAMFNxLuPwR-6cPpw"
  }
}
```

#### Adding the PIN

On the Relay Transport, the key of a response is passed once more through
HKDF-SHA-256 ([[spec:RFC5869]]), with the [[ref:PIN]] as salt:

```
key = HKDF-SHA-256(
  ikm    = key,
  salt   = UTF8(pin),
  info   = UTF8("enbox/connect/v2/pin"),
  length = 32 bytes
)
```

The PIN appears nowhere in the envelope. A recipient with the wrong PIN
derives a different key, and decryption fails. On the popup transport no PIN is
supplied and this step is skipped completely (note that passing an empty string
`""` as the PIN still runs HKDF with a 0-byte salt and produces a different key
from omitting the PIN).

#### Checking a Response

Having decrypted a response and verified its signature, an application
****MUST**** check, using the current Unix time in whole seconds (`now`), that:

- `aud` is the `clientDid` of its request;
- `nonce` and `state` are those of its request;
- `providerDid` is the `expectedProviderDid` of its request, if it set one; and
- `exp > iat`, `iat <= now + 60`, and `now < exp`.

## Owner Authentication Proofs

When a request contains `authentication`, whatever its `requestType`, the
wallet answers with an [[ref:Owner Authentication Proof]]: a signature by the
owner DID itself, over the relying service's challenge and over the request
exactly as the client sent it.

The delegate credentials of a `connect` response show what an application may
do. They do not prove who approved it, because the response is not signed by
the owner DID. A service that signs users in needs the proof.

### Proof Compact JWS Format

A proof is a Compact JWS.

- **Protected header**, with exactly these three properties:
  ```json
  {
    "alg": "EdDSA",
    "kid": "did:dht:i9xkp8ddcbcg8jwq54ox699wuzxyifsqx4jru45zodqu453ksz6y#0",
    "typ": "enbox-connect-owner-authentication+jwt"
  }
  ```
  `alg` follows from the signing key: `EdDSA` for Ed25519, `ES256K` for
  secp256k1, `ES256` for P-256. The signature is always 64 bytes; for the two
  ECDSA algorithms it is fixed-width `R || S` (never ASN.1 `DER`), and verifiers
  ****MUST**** accept both low-`S` and high-`S` signatures.
- **Payload**, with exactly these eight properties:
  ```json
  {
    "iss": "did:dht:i9xkp8ddcbcg8jwq54ox699wuzxyifsqx4jru45zodqu453ksz6y",
    "sub": "did:dht:i9xkp8ddcbcg8jwq54ox699wuzxyifsqx4jru45zodqu453ksz6y",
    "aud": "https://notes.example",
    "nonce": "OzcBUfNeDjydxX9F3oN2bmKdfgAHyzLa",
    "request_hash": "yFng9G93-M4s-cW6vYjGJ2DmH5LxIC5KyIY1k8x0oB0",
    "purpose": "enbox-connect-authentication",
    "iat": 1780272000,
    "exp": 1780272300
  }
  ```
- **Serialization**: the header and the payload are each written in the
  canonical form described below, then base64url-encoded. The signature is
  over the ASCII string `<header>.<payload>`.
- **Lifetime and size**: `iat` and `exp` are non-negative safe integers; a
  wallet sets `exp` to `iat + 300`. Using the current Unix time in whole
  seconds (`now`), a verifier rejects a proof when `iat < 0`, `exp <= iat`,
  `exp - iat > 300`, `iat > now + 30`, or `now >= exp`. A proof is at most
  `16,384` characters.
- **`request_hash`** is the unpadded base64url SHA-256 digest of the UTF-8
  bytes of the *canonical form* of the request exactly as the client created it
  (following [[spec:RFC8785]] JSON Canonicalization Scheme): compact JSON with
  no whitespace, object keys sorted by UTF-16 code unit (which differs from
  UTF-8 byte order when keys mix `U+E000..U+FFFF` with supplementary characters
  `U+10000+`), finite numbers formatted per RFC 8785 (integer-valued floats
  omit `.0` and `-0` serializes as `0`), strings escaped per RFC 8785 (with
  lone surrogates escaped as lowercase `\uXXXX`), and absent properties
  omitted. Only plain objects, arrays, strings, finite numbers, booleans, and
  `null` can be canonicalized. The canonical request is at most `262,144` bytes
  and at most 32 levels deep.

### Wallet and Verifier Requirements

A **wallet** that produces a proof:

- ****MUST**** know the origin of the requester by its own means, not from the
  request: from the browser, on the popup transport, or from the relying
  service's own policy, on the relay transport. That origin ****MUST**** be
  `https:` and ****MUST**** equal the origin of `request.authentication.audience`.
- ****MUST**** sign as an identity it owns. A delegate cannot produce a proof.
- ****MUST**** sign with a key listed uniquely under both `authentication` and
  `verificationMethod` in the owner DID's document, with `controller` equal to
  the owner DID. A relative reference such as `#key-1` is read as
  `<providerDid>#key-1`. A document with more than 64 `authentication` or
  `verificationMethod` entries is refused.
- ****SHOULD**** choose and check the key before it changes anything, but sign
  only at the end of the approval, so that the time the approval takes does
  not eat into the proof's 300 seconds.

A **verifier**:

- ****MUST**** check that each of the three parts of the proof is unpadded
  base64url; that `kid` is at most 2048 characters and has the form
  `<providerDid>#<fragment>`; and that `providerDid` is a bare DID of at most
  2048 characters, differs from `clientDid` and from `delegateDid`, uses one of
  the request's `supportedDidMethods`, and equals `expectedProviderDid` when
  the request set one.
- ****MUST**** resolve `providerDid` afresh, refuse a deactivated DID or a
  document whose `id` does not equal `providerDid` or that has more than 64
  `authentication` or `verificationMethod` entries, and verify that `kid`
  resolves uniquely under `authentication` (and, if referenced by string, once
  in `verificationMethod`), has `controller` equal to `providerDid`, and yields
  a public JWK (no private parameters such as `d` or `k`). If the key has an
  `alg`, it ****MUST**** match the proof's.
- ****MUST**** check that `iss` and `sub` are `providerDid`; that `aud`, `nonce`,
  and `request_hash` match the request it issued; and that `purpose` is
  `enbox-connect-authentication`.
- ****MUST**** check the time window, re-check it after DID resolution and
  **before** consuming the replay challenge (so an expired proof cannot burn an
  outstanding challenge), and check it again immediately before accepting the
  proof.
- ****MUST****, if it is a relying service, consume each `(audience, challenge)`
  pair durably and atomically, so that a proof is accepted once. A client
  running in a browser can verify a proof cryptographically but cannot provide
  this guarantee; the service the user is signing in to has to.
- reports failure as one of: `invalid-request`, `invalid-proof`,
  `unsupported-key`, `invalid-signature`, `context-mismatch`, `expired-proof`,
  `did-resolution`, `replayed-proof`.

A client that asks for a proof ****MUST**** give the exact request to the
relying service *before* sending it to the wallet, so that the service can
later recompute `request_hash`. A response that lacks a proof that was asked
for, or carries one that was not, ****MUST**** be rejected.

## Provider Processing

This section defines what a wallet does with a `connect` or `refresh` request
the user approves. An `authenticate` request involves none of it: the wallet
only produces the [[ref:Owner Authentication Proof]].

The work has an order, because each step depends on the one before. A wallet
****MUST NOT**** send an approving response unless every step marked
*required* has succeeded.

### Before Changing Anything

A wallet first checks the request, and refuses it if any check fails:

1. If `expectedProviderDid` is set, the identity that answers ****MUST**** be that DID.
2. Every `moduleDefinition` is a valid DWN module definition with a normalized module URI, and no module appears twice.
3. No `moduleDefinition` contains `$keyAgreement` or `$encryption` anywhere. Encryption keys always come from the wallet.
4. Every scope in `permissionScopes` is a valid DWN permission scope whose `module` equals its `moduleDefinition.module`.
5. No scope is `Modules`/`Configure`. The wallet installs modules itself; it does not delegate that.
6. `requestedSessionTtlSeconds`, if present (and not overridden by a wallet-approved TTL), is finite and rounds down to at least 1.
7. If the request contains `authentication`, the checks in [Wallet and Verifier Requirements](#wallet-and-verifier-requirements) pass.

### The Delegate

- If the request has no `delegateDid`, the wallet creates the delegate: a new
  `did:jwk` with an Ed25519 key (with `use` omitted on the JWK). It returns the
  delegate's keys in `delegatePortableDid`, including the X25519 private key
  converted from the Ed25519 seed (`SHA-512(seed)[0..31]` clamped per
  [[spec:RFC8032]]), which is the delegate's root key for DWN encryption.
- If the request has a `delegateDid`, that DID is the delegate, and the wallet
  returns no key material. `delegateDid` ****MUST**** be a bare DID; if it is a
  `did:jwk` and the session requires grant keys, its JWK ****MUST**** omit
  `"use": "sig"` so that `did:jwk` resolution exposes `#0` under `keyAgreement`.

### The Session

The grants of one approval form a session. Its lifetime is the value the user
approved, or else `requestedSessionTtlSeconds`, or else **3600 seconds**; in
every case at most **7,776,000 seconds** (90 days). Every grant of the session
expires at the same instant.

Each session grant carries a `connectSession` object in its data, so that a
wallet can later show the user which application holds which grants:

| Property | Limit | Meaning |
|---|---|---|
| `id` | 128 characters | ****REQUIRED****. An identifier shared by the grants of one approval |
| `createdAt`, `expiresAt` | | ****REQUIRED****. Timestamps. `expiresAt` equals the grants' `dateExpires` |
| `applicationId` | 512 | From the request |
| `appName` | 128 | From the request |
| `appIcon` | 2048 | From the request |
| `origin` | 512 | The requester's origin. A wallet that knows the origin by its own means (see [Browser Popup `postMessage` Transport](#browser-popup-postmessage-transport)) ****MUST**** use that value instead of the request's |
| `userAgent` | 512 | From `clientMetadata` |
| `platform` | 128 | From `clientMetadata` |
| `language` | 64 | From `clientMetadata` |
| `languages` | 16 entries of 64 | From `clientMetadata` |
| `timezone` | 128 | From `clientMetadata` |
| `transport` | | `"relay"` or `"postMessage"` |

Everything except `id`, `createdAt`, `expiresAt`, and a wallet-verified
`origin` is the application's own claim. A wallet omits empty (`""`) optional
string values (and empty entries in `languages`) and truncates remaining values
to the limits above. `connectSession` has no effect on authorization.

### Modules (required)

For each requested module, the wallet makes sure the owner's nodes have it
installed. Whenever a module requires key agreement (`encryptionRequired: true`
on any type, or `$role: true` on any path in `structure`), the wallet derives
the owner's module and path keys as the
[DWN specification](https://dwn-spec.pages.dev#module-keys) defines and injects
`$keyAgreement` onto **both** top-level `definition.$keyAgreement` **and every
non-`$ref` rule set in `structure`**.

| Installed on the owner's node | The wallet |
|---|---|
| Nothing | Installs the requested definition |
| The requested definition, missing `$keyAgreement` on top-level `definition` or any non-`$ref` path when key agreement is required | Installs it again, with keys on all of them |
| The requested definition, with the owner's keys | Does nothing |
| A different definition (comparing everything except `$keyAgreement` and `$encryption`) | ****MUST NOT**** replace it unless the user explicitly approves the replacement |
| Any definition whose `$keyAgreement` keys are not the owner's | ****MUST**** refuse the request |

A module that is requested with no scopes and is named in another requested
module's `uses` is a dependency. If it is already installed, the installed
definition is kept. A module ****MUST**** be installed after the modules it
`uses`.

The wallet installs each module on its own node and on the owner's remote
endpoints. Before it continues, it ****MUST**** confirm, by querying, that
every endpoint it could reach now holds the definition, and it ****MUST****
have reached at least one.

### Session Grants (required)

For each requested scope the wallet writes one [[ref:Permission Grant]] from
the owner to the delegate, with the scope exactly as requested, the session's
expiry, and the `connectSession` object.

| Scope | Delegated |
|---|---|
| `Records`/`Read`, `Records`/`Write`, `Records`/`Delete` | `true` |
| `Messages`/`Read` | `true` |
| `Modules`/`Query` | `false` |

Each grant ****MUST**** be accepted (status `202`, or `409` if it is already
there) by at least one of the owner's remote endpoints before the wallet
responds. The delegate will present the grant to those endpoints, and a grant
they do not hold authorizes nothing.

### Grant Keys (required)

A grant conveys authority but no keys. A module needs grant keys whenever its
definition requires key agreement (`encryptionRequired: true` on any type, or
`$role: true` on any path) and the request asks for at least one
`Records`/`Read` scope on that module. For each such module, the wallet
inspects every grant-key-eligible session grant issued for it — both
`Records`/`Read` and `Records`/`Write` grants (see
[Grant Key Coverage](https://dwn-spec.pages.dev#grant-key-coverage)) — and
writes one grant key record for every key those grants cover.

- For a delegate the wallet created, the records are `grantKey` records,
  encrypted to the delegate's derived key at
  `["modulePath", "https://identity.foundation/dwn/modules/encryption", "grantKey"]`.
- For a delegate the application supplied, the wallet resolves the delegate's
  DID (`keyAgreement[0]`, accepting X25519 or converting an Ed25519 key via the
  birational map to X25519) and writes `wrappedGrantKey` records wrapped to that
  root X25519 key.

Each record ****MUST**** be accepted (`202` or `409`) by at least one of the
owner's remote endpoints before the wallet responds.

### Revocation Grants

For each session grant the wallet also writes a companion grant that lets the
delegate revoke that one grant, and nothing else:

```json
{
  "interface": "Records",
  "method": "Write",
  "module": "https://identity.foundation/dwn/permissions",
  "contextId": "bafyreig3v56ki2375j2s4m5u7u6n4f3q2v5w7x8y9z2a3b4c5d6e7f8g9h"
}
```

The companion grant is `delegated: true`, expires with the session, and has no
`connectSession`. The wallet sends it best-effort to the owner's remote
endpoints (awaiting delivery attempts but not requiring any endpoint to accept
it); a client makes sure of that itself before it relies on the grant (see
[Ending a Session](#ending-a-session)).

The response's `delegateGrants` lists the session grants and then the
companion grants. `sessionRevocations` holds one
`{ "grantId", "revocationGrantId" }` pair per session grant.

### Reference Timing

The reference wallet waits at most 10 seconds for a DID to resolve and for
each request to an endpoint, caps fan-out concurrency at 8, and gives the
entire session-grant delivery batch across all endpoints 20 seconds in total.
These are not requirements.

## Client Processing

### Checking the Response

After opening the response and making the checks in
[Checking a Response](#checking-a-response), and before using anything in it,
a client ****MUST**** also check that:

1. if it supplied a `delegateDid`, the response's `delegateDid` is that DID; and if it did not, the response contains `delegatePortableDid`, whose DID is then the delegate;
2. every grant in `delegateGrants` is from `providerDid` to the delegate;
3. every grant listed as a `revocationGrantId` in `sessionRevocations` has the scope `Records`/`Write` on `https://identity.foundation/dwn/permissions` with `contextId` equal to the paired `grantId`;
4. every other grant is within a scope the client requested: it has the same `interface` and `method` as a requested scope, and every property of that requested scope has the same value in the grant; and
5. if `permissionRequests` is non-empty, at least one such grant is present.

A response that fails a check ****MUST**** be rejected. The grants already
exist on the owner's nodes by then, so the client ****SHOULD**** tell the user
to revoke the session in their wallet.

Because of rule 5, a request must ask for at least one scope. A request whose
modules are all dependencies, with no scopes at all, cannot succeed.

### Using the Grants

A client stores the grants where its own node can find them, and invokes them
as the DWN specification defines: `delegated` grants by signing as the
delegate on the owner's behalf, and the `Modules`/`Query` grant by
`permissionGrantId`.

What a client may synchronize follows from its `Messages`/`Read` grants (see
the [DWN Sync Protocol](https://dwn-sync-spec.pages.dev)): a grant with no
`module` allows everything, and a whole-module grant allows that module
(excluding `https://identity.foundation/dwn/permissions` and grants narrowed by
`modulePath` or `contextId`). Expired or revoked grants allow nothing.

A client groups grants into sessions by `connectSession.id`, and when multiple
complete sessions exist across the owner and delegate partitions it selects the
one with the latest `connectSession.createdAt` (tie-broken by the greatest
`connectSession.id`). It determines the session's state from its grants'
`dateExpires` (not `connectSession.expiresAt`), using `connectSession.createdAt`
only to compute the 10% `expiring-soon` window:

| State | Condition |
|---|---|
| `none` | No complete session exists |
| `revoked` | Any grant of the session has been revoked |
| `expired` | The earliest `dateExpires` among the session's grants has passed |
| `expiring-soon` | That time is near. By default: within one hour, or within 10% of the session's lifetime, whichever is less |
| `active` | Otherwise |

### Renewing a Session

To renew, a client sends a request with `requestType: "refresh"`, the
`delegateDid` it already has, and `expectedProviderDid` set to the owner DID.
The wallet processes it like a `connect` request for a supplied delegate, and
issues a new session to the same delegate. The client keeps its delegate
identity and adds the new grants. A session may be renewed before or after it
expires.

### Ending a Session

To disconnect, a client revokes each of its session grants by writing a
revocation record to the owner's node, as the delegate, under the companion
grant that `sessionRevocations` pairs with it. No wallet interaction is
needed. The DWN specification defines the revocation record and its effect.

Before writing a revocation, the client ****SHOULD**** send the companion
grant to the owner's endpoint itself, since the wallet's own delivery of it is
not guaranteed. A client ****SHOULD**** also remember revocations it could not
deliver, and retry them the next time it runs, so that a disconnect made
offline still takes effect.

## Transport Bindings

### Relay Transport

#### The Wallet URI

The application hands the request to the wallet as a [[ref:Wallet URI]]:

```
<wallet URI>#request_uri=<request_uri, percent-encoded>&encryption_key=<key, base64url>
```

- `<wallet URI>` is the address of the wallet: a web page, such as
  `https://wallet.example/connect/app`, or an application link, such as
  `enbox://connect`, which is what the reference client uses when it is given
  no other.
- `request_uri` is the value the relay returned when the request was stored.
- `encryption_key` is the 32-byte key that encrypts the request.

Both parameters are in the **fragment**. A browser never sends a fragment to
a server, so the key cannot end up in a web server's or a proxy's logs. A
wallet ****MUST**** read the parameters from the fragment only, and
****MUST**** ignore any of the same name in the query string. It
****MUST**** reject a key that does not decode to exactly 32 bytes.

A wallet URI shown as a QR code has to fit in one. The reference client draws
QR codes that hold at most 213 bytes, so wallet and relay URLs need to be
short.

An application that opens the wallet as a link in the same browser
****SHOULD**** open it in a new tab with `rel="noopener noreferrer"`, so that
its own page stays open to collect the response, and the wallet gets no
handle on it.

#### While Waiting

- **Keeping the request alive.** A request on the relay expires after
  `expires_in` seconds. Until a wallet has fetched it, an application that is
  still showing a QR code ****SHOULD**** store a fresh request before the old
  one expires, and show the new Wallet URI. The reference client does so two
  minutes before expiry.
- **Polling.** The reference client polls for the response every 3 seconds,
  for up to 300 seconds.
- **The PIN.** An application that lets the user try the PIN again
  ****MUST**** limit the attempts; the reference browser client allows 6. It
  ****SHOULD**** give up at once on any failure that is not a wrong PIN, such
  as a malformed envelope or an unexpected algorithm.

### Browser Popup `postMessage` Transport

The wallet's page for this transport is at
`https://<wallet origin>/dweb-connect`. Everything that passes between the two
windows is encrypted; there is no unencrypted fallback.

**Opening the popup.** The application opens the page directly in the handler
of the user's click, before doing anything asynchronous, or a popup blocker
will stop it. If the user closes the popup before the wallet has announced
itself, the exchange has failed. If they close it afterwards, the reference
client waits 1.5 seconds for a response that may already be on its way, and
then treats the request as denied. It waits at most 300 seconds overall.

**The four messages.**

1. Wallet to application, when the wallet's page has loaded:
   ```json
   {
     "type": "enbox-connect-loaded",
     "walletEpk": {
       "kty": "OKP",
       "crv": "X25519",
       "x": "jZOuuZo39lnOAui2YmbmLXO9zLOjODQCBxKcJmbENBY"
     },
     "walletOrigin": "https://wallet.example"
   }
   ```
   `walletEpk` is an X25519 public key the wallet has created for this
   exchange. The application ****MUST**** check that it is one, and that
   `walletOrigin` is the origin of the page it opened.
2. Application to wallet:
   ```json
   {
     "type": "enbox-connect-request",
     "jwe": "eyJhbGciOiJFQ0RILUVTIiwiZW5jIjoiWEMyMFAiLCJ0eXAiOiJlbmJveC1jb25uZWN0LXJlcSIsImN0eSI6IkpXVCJ9..bm9uY2UyNGJ5dGVzMTIzNDU2Nzg5MDEy.Y2lwaGVydGV4dA.dGFnMTZieXRlczEyMzQ1Ng"
   }
   ```
   The request is encrypted to `walletEpk`, with `apv` set to `walletOrigin`
   (see [Deriving a Key with `ECDH-ES`](#deriving-a-key-with-ecdh-es)). Its
   `reply` is `{ "mode": "post_message" }`.
3. Wallet to application:
   ```json
   {
     "type": "enbox-connect-response",
     "payload": "eyJhbGciOiJFQ0RILUVTIiwiZW5jIjoiWEMyMFAiLCJ0eXAiOiJlbmJveC1jb25uZWN0LXJlcyIsImN0eSI6IkpXVCJ9..bm9uY2UyNGJ5dGVzMTIzNDU2Nzg5MDEy.Y2lwaGVydGV4dA.dGFnMTZieXRlczEyMzQ1Ng"
   }
   ```
   (When the user refuses the request, `payload` is the literal string `"DENIED"`.)
4. Application to wallet, as soon as it has opened and verified the response:
   ```json
   { "type": "enbox-connect-ack" }
   ```
   The wallet can then close its window. The reference wallet waits 5 seconds
   for this message.

**Knowing who is on the other side.** This transport has no PIN. What takes
its place is that the browser tells each window the origin of every message
it receives, and delivers a posted message only to the origin the sender
names.

- Each side ****MUST**** post only to the specific origin of the other, never
  to `*`.
- Each side ****MUST**** ignore any message whose origin, or whose source
  window, is not the one it expects.
- The wallet learns the application's origin from its own configuration, or
  else from the page's referrer. If it has neither, it ****MUST**** refuse
  to proceed. An application page therefore ****MUST NOT**** suppress the
  referrer, whether with `Referrer-Policy: no-referrer` or by opening the
  popup with `noreferrer`.
- The wallet ****MUST**** check that the request's `apv` is its own origin,
  and that its `reply.mode` is `post_message`.

The application's origin, as the browser reports it, is what this
specification calls the *verified requester origin*. The wallet records it in
`connectSession.origin`, and requires it to match the audience of an
[[ref:Owner Authentication Proof]].

## Permission Model

A request lists each module the application needs, with the scopes it wants
there. For a module it will read and write, an application normally asks for:

```json
[
  { "interface": "Modules", "method": "Query", "module": "https://notes.example/modules/notes/v1" },
  { "interface": "Messages", "method": "Read", "module": "https://notes.example/modules/notes/v1" },
  { "interface": "Records", "method": "Read", "module": "https://notes.example/modules/notes/v1" },
  { "interface": "Records", "method": "Write", "module": "https://notes.example/modules/notes/v1" },
  { "interface": "Records", "method": "Delete", "module": "https://notes.example/modules/notes/v1" }
]
```

- `Modules`/`Query` lets the delegate read the installed definition.
- `Messages`/`Read` lets it synchronize the module.
- `Records`/`Read` covers querying, counting, and subscribing as well as
  reading. The DWN specification has no separate scopes for those.
- `Modules`/`Configure` cannot be requested. The wallet installs modules
  itself.

A scope may be narrower than a whole module, with a `modulePath` or a
`contextId`, as the DWN specification allows.

A module that is needed only because another requested module `uses` it is
listed with an empty `permissionScopes`. The wallet installs it and issues no
grant for it.

Which of the resulting grants are delegated is given under
[Session Grants](#session-grants-required).

## Security and Privacy Considerations

- **The relay is not trusted.** It holds only ciphertext, for at most 600
  seconds, and hands each item out once. Its two status routes answer with a
  single boolean. A relay can withhold or delay messages, but cannot read or
  alter them.
- **The request key stays off the network.** It travels only in the fragment
  of the Wallet URI, which browsers do not send to servers.
- **The response is bound to its request.** It is encrypted to a key that
  exists only for this request, with the request's `state` mixed into the key
  derivation, and it must echo the request's `nonce`.
- **The PIN binds the response to the wallet in front of the user.** Someone
  who saw the QR code can read the request, but cannot produce a response the
  application will accept without also knowing the PIN.
- **Origins take the PIN's place in a browser.** Each side posts only to the
  other's exact origin and ignores messages from anywhere else, and a request
  is bound to the wallet's origin by `apv`.
- **Algorithms are fixed.** A recipient accepts exactly one `alg` and one
  `enc` for the envelope it expects, checks `typ` and `cty`, and rejects weak
  X25519 inputs. Signatures are always by `did:jwk`, so verifying one needs no
  network access.
- **What the application says about itself is unverified.** Its name, icon,
  and device details are shown to the user and recorded, within fixed size
  limits, as claims. Only an origin the wallet learned by its own means is
  more than a claim.
- **An application cannot choose the user's encryption keys.** A request that
  supplies `$keyAgreement` is refused. The wallet derives every key from the
  owner's root key, and refuses to proceed where it finds an installed module
  with keys that are not the owner's.
- **An application can keep its delegate's keys to itself.** By supplying
  `delegateDid`, it receives grants, and keys wrapped to that DID, without the
  wallet ever holding the delegate's private keys.
- **A wallet's answer is checked.** An application rejects grants from anyone
  but the owner, to anyone but its delegate, or beyond what it asked for.
- **A delegate can end its own session, and nothing more.** Each revocation
  grant covers a single session grant.
- **Owner authentication is separate from delegation.** The proof is signed by
  the owner's own key, which never leaves the wallet. It is bound to the
  relying service's origin, to a single-use challenge, and to the exact
  request; it lives at most 300 seconds; and the relying service accepts each
  challenge once.

## References

[DWN Specification](https://dwn-spec.pages.dev)
~ Enbox Decentralized Web Node Specification.

[DWN Transport Protocol Specification](https://dwn-transport-spec.pages.dev)
~ Enbox Decentralized Web Node Transport Protocol Specification.

[DWN Sync Protocol Specification](https://dwn-sync-spec.pages.dev)
~ Enbox Decentralized Web Node Sync Protocol Specification.

[[spec]]
