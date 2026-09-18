# Usage Examples

This page provides integration examples in three languages — curl / Python / Node.js — covering the full task lifecycle:

```
Create task → Query status (polling) → Close task
```

Runnable scripts are available in the repository `examples/` directory:

| Language | Script | Description |
|---|---|---|
| bash | `examples/bash/full_lifecycle.sh` | Full lifecycle (including quota pre-check) |
| Python | `examples/python/task_demo.py` | Full lifecycle |
| Node.js | `examples/node/task_demo.mjs` | Full lifecycle |

> `CLI_ID` / `CLI_SECRET` in all examples are placeholders; replace them with the real credentials issued by the service provider.

---

## 1. curl

### 1.1 Create a Task

```bash
BASE=https://gateway.yakidev.top
CLI_ID=<your-cli-id>
CLI_SECRET=<your-cli-secret>

curl -s -X POST "$BASE/v1/rollinace-api/live/tasks" \
  -H "Content-Type: application/json" \
  -H "X-Cli-Id: $CLI_ID" \
  -H "X-Cli-Secret: $CLI_SECRET" \
  -d '{"gameId":"2021039309","board":"npb","mode":"replay"}'
```

### 1.2 Query Task Status

```bash
TASK_ID=8f3a9c2e1b6d

curl -s "$BASE/v1/rollinace-api/live/tasks/$TASK_ID" \
  -H "X-Cli-Id: $CLI_ID" \
  -H "X-Cli-Secret: $CLI_SECRET"
```

### 1.3 Close a Task

```bash
curl -s -X DELETE "$BASE/v1/rollinace-api/live/tasks/$TASK_ID" \
  -H "X-Cli-Id: $CLI_ID" \
  -H "X-Cli-Secret: $CLI_SECRET"
```

### 1.4 Query Your Own Quota (confirm before creating a task)

```bash
curl -s "$BASE/v1/rollinace-api/live/quota" \
  -H "X-Cli-Id: $CLI_ID" \
  -H "X-Cli-Secret: $CLI_SECRET"
```

---

## 2. Python (requests)

```python
import time
import requests

BASE = "https://gateway.yakidev.top"
CLI_ID = "<your-cli-id>"
CLI_SECRET = "<your-cli-secret>"
HEADERS = {
    "X-Cli-Id": CLI_ID,
    "X-Cli-Secret": CLI_SECRET,
    "Content-Type": "application/json",
}

# 1) Create a task (mode=replay; live works the same way, but the match must not have ended)
r = requests.post(
    f"{BASE}/v1/rollinace-api/live/tasks",
    headers=HEADERS,
    json={"gameId": "2021039309", "board": "npb", "mode": "replay"},
)
r.raise_for_status()
task_id = r.json()["taskId"]
print("taskId:", task_id)

# 2) Poll the status until ended/error/stopped (recommended interval >= 10s)
while True:
    st = requests.get(f"{BASE}/v1/rollinace-api/live/tasks/{task_id}", headers=HEADERS).json()
    print("status:", st["status"], "| detail:", st.get("detail"), "| error:", st.get("error"))
    if st["status"] in ("ended", "error", "stopped"):
        break
    time.sleep(15)

# 3) Close the task (idempotent)
print(requests.delete(f"{BASE}/v1/rollinace-api/live/tasks/{task_id}", headers=HEADERS).json())
```

---

## 3. Node.js (built-in fetch)

```js
const BASE = "https://gateway.yakidev.top";
const CLI_ID = "<your-cli-id>";
const CLI_SECRET = "<your-cli-secret>";

const headers = {
  "X-Cli-Id": CLI_ID,
  "X-Cli-Secret": CLI_SECRET,
  "Content-Type": "application/json",
};

const sleep = (ms) => new Promise((r) => setTimeout(r, ms));

async function main() {
  // 1) Create a task
  const created = await fetch(`${BASE}/v1/rollinace-api/live/tasks`, {
    method: "POST",
    headers,
    body: JSON.stringify({ gameId: "2021039309", board: "npb", mode: "replay" }),
  }).then((r) => r.json());
  const taskId = created.taskId;
  console.log("taskId:", taskId);

  // 2) Poll the status
  for (;;) {
    const st = await fetch(`${BASE}/v1/rollinace-api/live/tasks/${taskId}`, { headers }).then((r) => r.json());
    console.log("status:", st.status, "| detail:", st.detail, "| error:", st.error);
    if (["ended", "error", "stopped"].includes(st.status)) break;
    await sleep(15000);
  }

  // 3) Close the task
  const closed = await fetch(`${BASE}/v1/rollinace-api/live/tasks/${taskId}`, {
    method: "DELETE",
    headers,
  }).then((r) => r.json());
  console.log("closed:", closed);
}

main();
```

---

## 4. bash Full Lifecycle Script

The `examples/bash/full_lifecycle.sh` in the repository includes the complete "quota pre-check → create → poll → close" flow:

```bash
# Usage: write your credentials into environment variables and run
export ROLLINACE_CLI_ID=<your-cli-id>
export ROLLINACE_CLI_SECRET=<your-cli-secret>

bash examples/bash/full_lifecycle.sh --game 2021039309 --board npb --mode replay
```

Script behavior:

1. First calls `GET .../quota` to pre-check the quota and exits if it is insufficient;
2. Calls `POST .../tasks` to create the task and saves the `taskId`;
3. Polls `GET .../tasks/{taskId}` every 15s until `ended` / `error` / `stopped`;
4. Finally calls `DELETE .../tasks/{taskId}` to close the task.
