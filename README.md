# kfire-protocol

Shared protocol definitions for **KFIRE** (Knight FIRE) — an open-source, self-hosted
gaming presence tracker inspired by Xfire. One server instance = one organization
(clan, guild, team) that can see in real time who is playing what among its members.

This repository is the **single source of truth** for the client/server contract:

| File | Purpose |
|------|---------|
| [`openapi.yaml`](./openapi.yaml) | REST API schema (OpenAPI 3.1) — auth, users, presence, sessions |
| [`websocket-events.md`](./websocket-events.md) | WebSocket message definitions for real-time presence |

## Related repositories

- [kfire-server](https://github.com/knightsofeternity/kfire-server) — Go/Fiber backend + admin web UI (AGPL-3.0)
- [kfire-client](https://github.com/knightsofeternity/kfire-client) — Tauri desktop tray client (MIT)

## Consuming the schemas

**Preview the API docs locally:**

```bash
npx @redocly/cli preview-docs openapi.yaml
```

**Lint after editing:**

```bash
npx @redocly/cli lint openapi.yaml
```

**Code generation** (examples):

```bash
# Go server types
go run github.com/oapi-codegen/oapi-codegen/v2/cmd/oapi-codegen -generate types -package api openapi.yaml

# TypeScript client types
npx openapi-typescript openapi.yaml -o types.ts
```

## Versioning

The protocol follows semantic versioning. Breaking changes to the REST API bump the
URL prefix (`/api/v1` → `/api/v2`); breaking changes to WebSocket messages bump the
`protocol_version` exchanged in the `hello` handshake.

## License

[Apache-2.0](./LICENSE)
