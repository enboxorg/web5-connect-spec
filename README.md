# Enbox Connect Protocol Specification

### Specification

Latest Draft: https://web5-connect-spec.pages.dev

### Conformance Artifacts

- **Conformance Test Vectors:** [`test-vectors/`](./test-vectors/)

### Companion Specifications

- **Decentralized Web Node (Core)**: https://dwn-spec.pages.dev (`enboxorg/dwn-spec`)
- **DWN Transport Protocol**: https://dwn-transport-spec.pages.dev (`enboxorg/dwn-transport-spec`)
- **DWN Sync Protocol**: https://dwn-sync-spec.pages.dev (`enboxorg/dwn-sync-spec`)

### Implementation

| Implementer | Language   | Repo                                                                                                                  | Environments     |
| ----------- | ---------- | --------------------------------------------------------------------------------------------------------------------- | ---------------- |
| Enbox       | TypeScript | [enboxorg/enbox](https://github.com/enboxorg/enbox) (`packages/connect`, `packages/auth`, `packages/agent`, `packages/dwn-server`) | Server, Web, CLI |

### Development

This spec is built using [Spec-Up](https://github.com/nickreserved/spec-up), a tool for generating W3C-style spec documents from Markdown.

```bash
npm install
npm run edit    # live-reload dev server
npm run render  # one-shot build
```

The rendered spec is output to `spec/index.html`.
