# Rollin Ace Live/Replay Task API (Baseball Realtime Report Auto-Operations)

> **Language / 语言**: [中文](README.md) · [English](README.en.md)

A public API documentation and examples repository for integrators. This repo provides the "baseball realtime report" live / replay auto-operations task API. Integrators **only need to know the domain and endpoints in this documentation** — no need to understand the backend:

| API | Domain | Description | Docs |
|---|---|---|---|
| **Live/Replay Task API** | `https://gateway.yakidev.top` (unified gateway) | Manage the live / replay auto-operations tasks of "baseball realtime report": create tasks (`live`/`replay`, auto-started on creation), query task status, close tasks, query usage quota | [doc/API_REFERENCE.en.md](doc/API_REFERENCE.en.md) |

Task API capabilities:

- **Create tasks** (live `live` / replay `replay`, auto-started on creation)
- **Query task status** (poll whether finished / errored)
- **Close tasks** (reclaim resources)
- **Query usage quota** (confirm your own quota is sufficient before creating)

---

## Quickstart

### 1. Get credentials

Use either credential on requests:

- **Method A (registration, recommended)**: apply to the service provider for registration, get `cliId` + `secret`
- **Method B (JWT)**: `Authorization: Bearer <token>`

> Credentials are issued by the service provider — keep them safe, **never hardcode them into frontends or commit them to code repos**. If credentials leak in plaintext, contact the service provider immediately to rotate.

### 2. Call the first endpoint

```bash
BASE=https://gateway.yakidev.top
CLI_ID=<your-cli-id>
CLI_SECRET=<your-cli-secret>

# Create a replay task
curl -s -X POST "$BASE/v1/rollinace-api/live/tasks" \
  -H "Content-Type: application/json" \
  -H "X-Cli-Id: $CLI_ID" \
  -H "X-Cli-Secret: $CLI_SECRET" \
  -d '{"gameId":"2021039309","board":"npb","mode":"replay"}'
```

Successful response:

```json
{ "code": 0, "taskId": "8f3a9c2e1b6d", "matchId": "L76HNWWV" }
```

> `matchId` is the live-room ID, **asynchronously backfilled** by the server after the room is created: it may not be ready when the create response returns (empty string then); you can get it via the task-status query endpoint later.

> **Judge success by `code == 0`, not just the HTTP status code.**

### 3. Manage tasks

```bash
TASK_ID=8f3a9c2e1b6d

# Query status
curl -s "$BASE/v1/rollinace-api/live/tasks/$TASK_ID" \
  -H "X-Cli-Id: $CLI_ID" -H "X-Cli-Secret: $CLI_SECRET"

# Close task (idempotent, callable repeatedly)
curl -s -X DELETE "$BASE/v1/rollinace-api/live/tasks/$TASK_ID" \
  -H "X-Cli-Id: $CLI_ID" -H "X-Cli-Secret: $CLI_SECRET"

# Confirm your own quota before creating
curl -s "$BASE/v1/rollinace-api/live/quota" \
  -H "X-Cli-Id: $CLI_ID" -H "X-Cli-Secret: $CLI_SECRET"
```

---

## Endpoint overview (prefix `/v1/rollinace-api/live`)

| Method | Path | Description |
|---|---|---|
| `POST` | `/v1/rollinace-api/live/tasks` | Create a task (live/replay, auto-started on creation) |
| `GET` | `/v1/rollinace-api/live/tasks/{taskId}` | Query task status |
| `DELETE` | `/v1/rollinace-api/live/tasks/{taskId}` | Close a task (idempotent) |
| `GET` | `/v1/rollinace-api/live/quota` | Query usage quota and open usage |

---

## Recommended integration flow

1. Contact the service provider to register `cliId` and get `secret`;
2. First use `GET .../quota` to confirm your live/replay quota (`maxLive`/`maxReplay`) is sufficient;
3. Call `POST .../tasks` with `gameId` + known `board`/`mode` to create a task, and save the returned `taskId`;
4. Periodically (suggest interval ≥10s) call `GET .../tasks/{taskId}` to check `status`;
5. When the business finishes or needs to be aborted, call `DELETE .../tasks/{taskId}` to close it;
6. With `mode=replay`, **don't** set `loop=true` (it will be rejected); pass `mode=replay` directly for replays.

---

## Repository structure

```
rollinace_live_api/
├── README.md                      # This document (quickstart)
├── doc/
│   ├── API_REFERENCE.md           # Live/Replay task: full API reference (fields/error codes/status table)
│   └── USAGE_EXAMPLES.md          # Multi-language usage examples (curl / Python / Node)
├── examples/
│   ├── bash/
│   │   └── full_lifecycle.sh      # Full lifecycle example (incl. quota pre-check)
│   ├── python/
│   │   └── task_demo.py           # Python example
│   └── node/
│       └── task_demo.mjs          # Node.js example
└── skills/
    └── rollinace-api-client/      # Agent Skill: Live/Replay task API
```

- Full live/replay task API details: [doc/API_REFERENCE.en.md](doc/API_REFERENCE.en.md).
- Multi-language usage examples: [doc/USAGE_EXAMPLES.en.md](doc/USAGE_EXAMPLES.en.md) and [examples/](examples/).
- Skill for other AI agents: Live/Replay task API at [skills/rollinace-api-client/](skills/rollinace-api-client/SKILL.md).

---

## Content & security notes

- This is a **public documentation repo** containing only public contracts (Live/Replay tasks: `https://gateway.yakidev.top/v1/rollinace-api/live/*`), **no** internal paths, origin addresses, or secrets.
- Don't commit any real credentials, keys, or `.env` files to this repo (`.gitignore` catches common cases).
- Auth failure for the live/replay task API uniformly returns `HTTP 401`: `{ "error": "unauthorized" }`.

---

## License

[MIT](LICENSE) — ©2026 yakizkna