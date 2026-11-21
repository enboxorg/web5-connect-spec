# Enbox Connect Protocol Test Vectors

Language-agnostic, deterministic JSON conformance test vectors for the
[Enbox Connect Protocol Specification](https://web5-connect-spec.pages.dev).

All signing and owner DIDs use offline `did:jwk` identities from
[`dwn-spec/test-vectors/identities.json`](https://github.com/enboxorg/dwn-spec/blob/main/test-vectors/identities.json)
(**test keys only — never use in production**).

## Files

| File | Coverage |
|---|---|
| [`jcs-canonicalization.json`](./jcs-canonicalization.json) | RFC 8785 JCS canonical JSON strings and SHA-256 base64url `request_hash` digests, including UTF-16 code-unit key ordering (`U+1F600` before `U+E000`, opposite of UTF-8 byte order), `-0` / integer-float formatting, and lone-surrogate escaping (`\ud800`) |
| [`jwe-envelopes.json`](./jwe-envelopes.json) | Deterministic 5-part Compact JWE (`XC20P`) vectors across all 4 Connect channels: Relay `dir` request, Popup `ECDH-ES` request (`apv`), Popup `ECDH-ES` response (`apu`), and Relay `ECDH-ES` + PIN HKDF response (including intermediate Concat KDF key, final PIN CEK, and wrong-PIN AEAD failure). Verify response expiry against `verifyAtNow` (`1780272010`) |
| [`owner-authentication-proof.json`](./owner-authentication-proof.json) | Offline-verifiable `enbox-connect-owner-authentication+jwt` proofs across `EdDSA` (`Ed25519`), `ES256K` (`secp256k1`), and `ES256` (`P-256`), clock-skew & expiry boundaries (`iat - 30` vs `iat - 31`, `exp - 1` vs `exp`), and `context-mismatch` rejection |
