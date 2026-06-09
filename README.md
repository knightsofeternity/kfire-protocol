<p align="center">
  <img src=".github/logo.png" alt="KFIRE" width="300" />
</p>

<h1 align="center">kfire-protocol</h1>

<p align="center">
  The shared API & WebSocket contract for KFIRE - the source of truth between client and server.
</p>

<p align="center">
  <a href="./LICENSE"><img alt="License: Apache-2.0" src="https://img.shields.io/badge/license-Apache--2.0-fb923c"></a>
  <img alt="OpenAPI" src="https://img.shields.io/badge/OpenAPI-3.1-6BA539?logo=openapiinitiative&logoColor=white">
</p>

---

**KFIRE** (Knight FIRE) is an open-source, self-hosted gaming presence tracker inspired by
Xfire. This repository is the **single source of truth** for the client/server contract.

| File | Purpose |
|------|---------|
| [`openapi.yaml`](./openapi.yaml) | REST API (OpenAPI 3.1): auth, device pairing, users, games, presence, sessions, connectors, admin |
| [`websocket-events.md`](./websocket-events.md) | WebSocket message definitions for real-time presence |

## Related repositories

- [kfire-server](https://github.com/knightsofeternity/kfire-server) - backend + admin web UI (AGPL-3.0)
- [kfire-client](https://github.com/knightsofeternity/kfire-client) - desktop tray client (MIT)

## Using the schemas

```bash
# Preview the API docs
npx @redocly/cli preview-docs openapi.yaml

# Lint after editing
npx @redocly/cli lint openapi.yaml

# Generate types
go run github.com/oapi-codegen/oapi-codegen/v2/cmd/oapi-codegen -generate types -package api openapi.yaml
npx openapi-typescript openapi.yaml -o types.ts
```

## Versioning

Semantic versioning. Breaking REST changes bump the URL prefix (`/api/v1` → `/api/v2`);
breaking WebSocket changes bump the `protocol_version` exchanged in the `hello` handshake.

## License

[Apache-2.0](./LICENSE)
