# API Reference

Complete reference for integrators using the "Rollin Ace Live/Replay Task" API. All requests go through the gateway.

| Item | Value |
|---|---|
| Base URL | `https://gateway.yakidev.top` |
| Protocol | HTTPS (HTTP requests are automatically redirected via 301) |
| Content type | `application/json` |
| API prefix | `/v1/rollinace-api/live` |

---

## 1. Authentication

Attach one of the following credentials to each request:

**Method A (registration-based, recommended)**

```
X-Cli-Id: <cliId>
X-Cli-Secret: <secret>
```

**Method B (JWT)**

```
Authorization: Bearer <token>
```

> Credentials are issued by the service provider. Store them securely and never hardcode them into frontend code. If credentials are exposed in plaintext, contact the service provider immediately to rotate them.

A failed authentication returns uniformly:

```json
// HTTP 401
{ "error": "unauthorized" }
```

If `cliId` is not registered or has been disabled, HTTP 403 (`code=1`) is returned; see §4 Error Codes.

---

## 2. Unified Response Convention

Responses are returned as plain JSON objects without an extra wrapping layer, and every response contains a business code `code`:

| code | Meaning |
|---|---|
| 0 | Success |
| 1 | Invalid parameter / business rejection (HTTP 400/403/409) |
| 2 | Internal error (HTTP 500) |
| 3 | Operation failure (start/stop failed, HTTP 500) |
| 4 | Match status check failure (HTTP 502) |
| 5 | Match has already ended; live task cannot be created (HTTP 409) |

> **To judge success, rely on `code == 0`, not just the HTTP status code.**

---

## 3. Endpoints

### 3.1 Create Task

```
POST /v1/rollinace-api/live/tasks
```

Creates a task and **starts it automatically**; no separate start call is needed.

**Request body** (JSON):

| Field | Type | Required | Description |
|---|---|---|---|
| `gameId` | string | Yes | Yahoo match ID (e.g. `2021039309`) |
| `board` | string | Yes | Tournament board: `npb` (Nippon Professional Baseball) / `hsb_summer` (Summer Koshien) / `u18` (U-18 Samurai Japan and other U18 tournaments) |
| `mode` | string | Yes | Task type: `live` / `replay` |
| `env` | string | No | Reporting environment: `pro` (production, default) / `tst` (test) |
| `loop` | bool | No | **Always pass `false`**; `true` is rejected (`code=1`) |
| `shortName` | string | No | Short name displayed in the live room; if left empty, defaults to the full "Home Team vs Away Team" name |
| `homeShortName` | string | No | Home team short name |
| `awayShortName` | string | No | Away team short name |

**Success response** (HTTP 200):

```json
{ "code": 0, "taskId": "8f3a9c2e1b6d", "matchId": "L76HNWWV" }
```

| Field | Type | Description |
|---|---|---|
| `code` | int | Business code; `0` means success |
| `taskId` | string | Task ID, used for subsequent status queries and task shutdown |
| `matchId` | string | Live room ID (**may be empty**, see note below) |

> **`matchId` is backfilled asynchronously**: the live room is created asynchronously by the server after the task is created, and `matchId` is backfilled once the rollin-ace room creation returns. By the time the create request returns, the room may not be ready yet, in which case `matchId` is an empty string; it can be retrieved later via the query task status endpoint (§3.2).

**Error response examples**:

```json
// HTTP 400 invalid parameter
{ "code": 1, "error": "mode must be live or replay" }

// HTTP 409 match already ended
{ "code": 5, "error": "比赛已结束，无法创建直播任务，请使用 mode=replay" }

// HTTP 409 insufficient quota
{ "code": 1, "error": "<额度提示>" }

// HTTP 500 creation failed
{ "code": 2, "error": "..." }
```

> **Pre-creation check for live tasks**: for `mode=live`, the server checks in real time whether the match has ended. If it has, the request is rejected with `code=5` (use `mode=replay` instead), avoiding the accidental creation of empty live rooms.
>
> **Open-quota limits**: the number of simultaneously open live/replay tasks for each registered `cliId` is limited by quota (`maxLive` / `maxReplay`). Creating is rejected (`code=1`, HTTP 409) when the limit is exceeded or the quota is 0. Quota is counted by "open" tasks (tasks that have ended, stopped, errored, or been closed do not consume quota).
>
> **Duplicate creation**: multiple tasks can be created for the same match (each with its own independent live room). To avoid duplicates, maintain your own `gameId → taskId` mapping.

### 3.2 Query Task Status

```
GET /v1/rollinace-api/live/tasks/{taskId}
```

**Success response** (HTTP 200):

```json
{
  "code": 0,
  "taskId": "8f3a9c2e1b6d",
  "matchId": "L76HNWWV",
  "status": "running",
  "detail": "live",
  "error": ""
}
```

| Field | Type | Description |
|---|---|---|
| `code` | int | Business code; `0` means success |
| `taskId` | string | Task ID |
| `matchId` | string | Live room ID (empty when the room has not been created) |
| `status` | string | Task status (see table below) |
| `detail` | string | Internal raw status (for debugging) |
| `error` | string | Error message (only populated when `status=error`) |

**`status` values**:

| status | Meaning | Recommended action |
|---|---|---|
| `waiting` | Created; waiting for first pitch / match not started | Keep polling |
| `running` | Playing (live reporting in progress / replay playing) | Keep polling |
| `paused` | Reporting paused (task still running) | Poll and wait for resume |
| `ended` | Ended (match over, room closed) | Close the task |
| `stopped` | Stopped | Close the task |
| `error` | Error occurred (check the `error` field) | Troubleshoot or close |
| `deleted` | Task does not exist or has been deleted (idempotent) | No action needed |

> When a task does not exist, **HTTP 404 is NOT returned**; instead HTTP 200 is returned with `status=deleted`.

### 3.3 Close Task

```
DELETE /v1/rollinace-api/live/tasks/{taskId}
```

Permanently closes the task: stop reporting → close the live room → delete the task record → clean up temporary data.

**Success response** (HTTP 200, idempotent — returns success even if the task does not exist):

```json
{ "code": 0, "taskId": "8f3a9c2e1b6d" }
```

### 3.4 Query Quota (limits/usage)

```
GET /v1/rollinace-api/live/quota
```

Queries the live/replay quota and open usage of the current caller (or a specified caller). Useful for confirming that you have enough quota before creating a task.

Optional query parameter:

| Parameter | Type | Description |
|---|---|---|
| `clientId` | string | The caller's `cliId` to query; when omitted, queries the caller corresponding to the current credentials |

**Success response** (HTTP 200):

```json
{
  "code": 0,
  "cliId": "cli_user",
  "maxLive": 0,
  "maxReplay": 1,
  "openLive": 0,
  "openReplay": 1
}
```

| Field | Type | Description |
|---|---|---|
| `code` | int | Business code; `0` means success |
| `cliId` | string | The queried caller's `cliId` |
| `maxLive` | int | Live quota limit (`0` means creating live tasks is prohibited) |
| `maxReplay` | int | Replay quota limit |
| `openLive` | int | Current number of open live tasks |
| `openReplay` | int | Current number of open replay tasks |

> Security constraint: you can only query yourself; querying another `cliId` is allowed only with an admin credential, otherwise HTTP 403 (`code=3`) is returned.

---

## 4. Common Error Quick Reference

| HTTP | code | Scenario |
|---|---|---|
| 401 | — | Missing/invalid credentials (`{"error":"unauthorized"}`) |
| 403 | 1 | `cliId` not registered or has been disabled |
| 400 | 1 | Missing required fields, invalid `mode`/`board`, `loop=true` |
| 502 | 4 | Match status check failed when creating a live task |
| 409 | 5 | Match has already ended; live task cannot be created |
| 409 | 1 | Open quota exceeded |
| 403 | 3 | Querying another caller's quota without admin privileges (only when quota is queried for others via `clientId`) |
| 500 | 2/3 | Internal server error |

---

## 5. Additional Notes

- **Replay data source**: `mode=replay` requires the match to have ended and the server to have cached the complete play-by-play data for that match; if the data is not ready, the task may enter the `error` state.
- **Environment distinction**: `env=tst` is for integration testing and reports to a test live room; use the default `pro` for production operations.
- **Polling interval**: for status queries, an interval of ≥10s is recommended to avoid putting pressure on the service.
