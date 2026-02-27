---
name: agent-teleport-backup
description: Create a portable OpenClaw workspace backup and upload it to TiDB/MySQL as chunked payload parts. Use on source machine before migration.
---

# Agent Teleport Backup (Source OpenClaw)

## EXECUTION CONTRACT (MANDATORY, NON-NEGOTIABLE)
The assistant MUST follow this contract exactly for every backup request.

### 0) Preflight disclosure (REQUIRED)
Before any command, print:
- skill name
- current working directory
- proposed default scope

Then ask for explicit user confirmation.

### 1) Scope selection first (REQUIRED)
Do NOT run backup commands until user selects one scope:
- A: Full workspace
- B: Memory-only (MEMORY.md, memory/, optional USER.md, IDENTITY.md)
- C: Custom paths (user-provided)

Rules:
- MUST ask user to choose A/B/C.
- MUST echo exact include/exclude paths before execution.
- MUST wait for user confirmation.
- MUST NOT auto-switch scope.

### 2) Step-by-step transparent logging (REQUIRED)
For each step, print a chat update in this exact format:

`[STEP n/5] <name>`
`Command: <exact command>`
`Result: <success|failed>`
`Output: <key stdout/stderr lines>`
`Next: <what happens next>`

Required steps:
1. Scope confirmation + path plan
2. Archive creation
3. Archive size check
4. DSN provisioning
5. DB write + final handoff

### 3) Size guardrail and stop policy (REQUIRED)
If archive > 32MB:
- MUST STOP immediately
- MUST ask user to choose:
  1) clean files and retry same scope
  2) switch to memory-only
- MUST NOT continue automatically

### 4) Failure policy (REQUIRED)
On any error:
- print failing step number
- print exact error message
- print 2 concrete recovery options
- ask user which option to execute
- do not silently retry unless user asked for auto-retry

### 5) Final output schema (REQUIRED)
Final response MUST contain all fields below (no omission):
- Backup scope:
- Included paths:
- Excluded paths:
- Archive size (MB):
- DSN provisioned:
- DB write status:
- RESTORE_DSN=
- Restore command (single-line, copy/paste ready)

### 6) Security handling (REQUIRED)
- Treat DSN as secret.
- Do not post DSN until DB write succeeds.
- If user asks to redact, show masked DSN plus a private handoff note.

### 7) Language + verbosity (REQUIRED)
- Match user language.
- If user asks “show every step”, do not summarize; show full step blocks.

## Execution model (important)
Run **one step at a time**. Do not send one giant command block.
After each step, post a short status update in chat.

## Chat Output Contract (MANDATORY)
After backup, send:
1) scope + archive size + part count + status
2) restore handoff with real code

```text
# B: restore
https://github.com/lilyjazz/openclaw-memory-teleport-skill/blob/main/skills/agent_teleport_restore/SKILL.md
Use this skill to restore OpenClaw’s memory.
🔐 Restore Code: <real code>
```

---

## Step 1 — Enter workspace
```bash
cd /home/ubuntu/.openclaw/workspace
```

## Step 2 — Preview size tree (folders)
```bash
du -h --max-depth=1 . 2>/dev/null | sort -hr | head -n 20
```

## Step 3 — Confirm scope in chat (MANDATORY, no default)
User must explicitly choose one scope before any backup run:
- `SCOPE=full`
- `SCOPE=core`
- `SCOPE=custom` with `CUSTOM_PATHS="path1 path2 ..."`

If user does not choose, stop and ask again. Do not continue.

## Step 4 — Build archive
```bash
set -euo pipefail
IGNORE=(.git .venv venv env __pycache__ node_modules "*.log" "*.pyc" ".DS_Store" ".env" "*.pem" "*.key" "id_rsa" "id_dsa")
EX=(); for p in "${IGNORE[@]}"; do EX+=(--exclude="$p"); done

SCOPE="${SCOPE:-}"
CUSTOM_PATHS="${CUSTOM_PATHS:-}"
[ -n "$SCOPE" ] || { echo "ERROR: SCOPE is required. Choose full/core/custom in chat first."; exit 1; }
SOURCES=()
case "$SCOPE" in
  core)
    for p in AGENTS.md SOUL.md USER.md MEMORY.md TOOLS.md IDENTITY.md HEARTBEAT.md memory skills; do
      [ -e "$p" ] && SOURCES+=("$p")
    done
    ;;
  custom)
    [ -n "$CUSTOM_PATHS" ] || { echo "ERROR: SCOPE=custom requires CUSTOM_PATHS"; exit 1; }
    # shellcheck disable=SC2206
    USER_PATHS=( $CUSTOM_PATHS )
    for p in "${USER_PATHS[@]}"; do [ -e "$p" ] || { echo "ERROR: path not found: $p"; exit 1; }; done
    SOURCES=("${USER_PATHS[@]}")
    ;;
  full)
    SOURCES=(.) ;;
  *)
    echo "ERROR: invalid SCOPE '$SCOPE'. Must be full/core/custom"; exit 1 ;;
esac

rm -f workspace.tar.gz
# shellcheck disable=SC2068
tar -czf workspace.tar.gz ${EX[@]} "${SOURCES[@]}"

du -h workspace.tar.gz
```

## Step 5 — Get DSN
```bash
set -euo pipefail
if [ -n "${TIDB_HOST:-}" ] && [ -n "${TIDB_USER:-}" ] && [ -n "${TIDB_PASSWORD:-}" ]; then
  TIDB_PORT="${TIDB_PORT:-4000}"
  DSN="mysql://${TIDB_USER}:${TIDB_PASSWORD}@${TIDB_HOST}:${TIDB_PORT}/test"
else
  DSN=$(curl -sS -X POST https://zero.tidbapi.com/v1alpha1/instances -H 'content-type: application/json' -d '{}' \
    | sed -n 's/.*"connectionString":"\([^"]*\)".*/\1/p')
fi
[ -n "$DSN" ] || { echo "ERROR: failed to get DSN"; exit 1; }
printf '%s\n' "$DSN" > .teleport_dsn.tmp
echo "DSN ready"
```

## Step 6 — Upload (auto chunk at 10MB)
```bash
set -euo pipefail
DSN="$(cat .teleport_dsn.tmp)"
UPLOAD_OUT=$(DSN="$DSN" ARCHIVE_PATH="$(pwd)/workspace.tar.gz" npx -y -p mysql2 node - <<'NODE'
const fs = require('fs');
const mysql = require('mysql2/promise');
(async () => {
  const dsn = process.env.DSN;
  const u = new URL(dsn);
  const db = (u.pathname || '/test').replace(/^\//,'') || 'test';
  const conn = await mysql.createConnection({ host: u.hostname, port: Number(u.port || 4000), user: decodeURIComponent(u.username), password: decodeURIComponent(u.password), database: db, ssl: { minVersion: 'TLSv1.2' } });
  try {
    await conn.query(`CREATE TABLE IF NOT EXISTS teleport_parts (transfer_id VARCHAR(64) NOT NULL, part_no INT NOT NULL, total_parts INT NOT NULL, data LONGBLOB NOT NULL, created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP, PRIMARY KEY (transfer_id, part_no))`);
    const buf = fs.readFileSync(process.env.ARCHIVE_PATH);
    const CHUNK = 10 * 1024 * 1024;
    const total = Math.ceil(buf.length / CHUNK) || 1;
    const transferId = `tp_${Date.now().toString(36)}_${Math.random().toString(36).slice(2,8)}`;
    for (let i = 0; i < total; i++) {
      const part = buf.subarray(i * CHUNK, Math.min((i + 1) * CHUNK, buf.length));
      await conn.query('INSERT INTO teleport_parts (transfer_id, part_no, total_parts, data) VALUES (?, ?, ?, ?)', [transferId, i + 1, total, part]);
      process.stderr.write(`[upload] part ${i + 1}/${total}\n`);
    }
    process.stdout.write(`TRANSFER_ID=${transferId}\nPARTS=${total}\nSIZE_BYTES=${buf.length}\n`);
  } finally { await conn.end(); }
})();
NODE
)
printf '%s\n' "$UPLOAD_OUT" > .teleport_upload.tmp
echo "$UPLOAD_OUT"
```

## Step 7 — Generate restore code
```bash
set -euo pipefail
DSN="$(cat .teleport_dsn.tmp)"
TRANSFER_ID=$(sed -n 's/^TRANSFER_ID=//p' .teleport_upload.tmp)
PARTS=$(sed -n 's/^PARTS=//p' .teleport_upload.tmp)
SIZE_BYTES=$(sed -n 's/^SIZE_BYTES=//p' .teleport_upload.tmp)
RAW="${DSN}|${TRANSFER_ID}"
CODE_PAYLOAD=$(printf '%s' "$RAW" | base64 | tr -d '\n' | tr '+/' '-_' | tr -d '=')
RESTORE_CODE="RESTORE-${CODE_PAYLOAD}"
printf '%s\n' "$RESTORE_CODE" > teleport_restore_code.txt
printf 'RESTORE_CODE=%s\nPARTS=%s\nSIZE_BYTES=%s\n' "$RESTORE_CODE" "$PARTS" "$SIZE_BYTES"
```

## Step 8 — Post chat summary
Use this exact template in chat:
```text
Backup completed.
- Scope: <full|core|custom>
- Archive: <size>
- Parts: <n>
- Status: success

# B: restore
https://github.com/lilyjazz/openclaw-memory-teleport-skill/blob/main/skills/agent_teleport_restore/SKILL.md
Use this skill to restore OpenClaw’s memory.
🔐 Restore Code: <real code>
```

## Failure retry index (backup)
- Fail at **Step 4 (build archive)**: fix scope/path, rerun Step 4.
- Fail at **Step 5 (get DSN)**: check network/TiDB env, rerun Step 5.
- Fail at **Step 6 (upload)**: rerun Step 6 only (uses existing `workspace.tar.gz` and `.teleport_dsn.tmp`).
- Fail at **Step 7 (code generation)**: rerun Step 7 only.
