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
Do NOT run backup commands until user confirms memory-only scope.

Allowed scope:
- B: Memory-only (MEMORY.md, memory/, SOUL.md, USER.md, IDENTITY.md, TOOLS.md)

Rules:
- MUST ask user to confirm memory-only backup.
- MUST echo exact include/exclude paths before execution.
- MUST wait for user confirmation.
- MUST NOT switch to full/custom scope.

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
- DSN provisioned: <yes/no, masked>
- DB write status:
- RESTORE_CODE=
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

## Step 3 — Confirm scope in chat (MANDATORY)
Only one scope is allowed:
- `SCOPE=core` (memory/persona docs only)

Included paths are fixed:
- `MEMORY.md`
- `memory/`
- `SOUL.md`
- `USER.md`
- `IDENTITY.md`
- `TOOLS.md`

If user asks for full/custom workspace backup, refuse and recommend GitHub sync for code/projects.

## Step 4 — Build archive
```bash
set -euo pipefail
IGNORE=(.git .venv venv env __pycache__ node_modules "*.log" "*.pyc" ".DS_Store" ".env" "*.pem" "*.key" "id_rsa" "id_dsa")
EX=(); for p in "${IGNORE[@]}"; do EX+=(--exclude="$p"); done

SCOPE="${SCOPE:-core}"
[ "$SCOPE" = "core" ] || { echo "ERROR: only SCOPE=core is supported. Use GitHub sync for large code/project directories."; exit 1; }
SOURCES=()
for p in MEMORY.md memory SOUL.md USER.md IDENTITY.md TOOLS.md; do
  [ -e "$p" ] && SOURCES+=("$p")
done
[ ${#SOURCES[@]} -gt 0 ] || { echo "ERROR: no core files found to backup"; exit 1; }

rm -f workspace.tar.gz
# shellcheck disable=SC2068
tar -czf workspace.tar.gz ${EX[@]} "${SOURCES[@]}"

du -h workspace.tar.gz
```

## Step 5 — Get DSN
```bash
set -euo pipefail
umask 077
if [ -n "${TIDB_HOST:-}" ] && [ -n "${TIDB_USER:-}" ] && [ -n "${TIDB_PASSWORD:-}" ]; then
  TIDB_PORT="${TIDB_PORT:-4000}"
  DSN="mysql://${TIDB_USER}:${TIDB_PASSWORD}@${TIDB_HOST}:${TIDB_PORT}/test"
  CLAIM_URL=""
  EXPIRES_AT=""
else
  IDEMPOTENCY_KEY="teleport-$(date +%s)-$(od -An -N4 -tx1 /dev/urandom | tr -d ' \n')"
  ZERO_JSON=""
  for attempt in 1 2 3 4 5; do
    HDR="$(mktemp)"
    BODY="$(mktemp)"
    HTTP_CODE=$(curl -sS -D "$HDR" -o "$BODY" -w '%{http_code}' -X POST https://zero.tidbapi.com/v1alpha1/instances \
      -H 'content-type: application/json' \
      -H "Idempotency-Key: ${IDEMPOTENCY_KEY}" \
      -d '{"tag":"openclaw-memory-teleport"}' || printf '000')
    if [ "$HTTP_CODE" = "200" ]; then
      ZERO_JSON="$(cat "$BODY")"
      rm -f "$HDR" "$BODY"
      break
    fi
    if [ "$HTTP_CODE" = "429" ] || [ "$HTTP_CODE" = "503" ]; then
      RETRY_AFTER=$(sed -n 's/^[Rr]etry-[Aa]fter:[[:space:]]*\([0-9][0-9]*\).*/\1/p' "$HDR" | tail -n 1)
      rm -f "$HDR" "$BODY"
      sleep "${RETRY_AFTER:-$((attempt*2))}"
      continue
    fi
    echo "ERROR: Zero API request failed (HTTP $HTTP_CODE)"
    sed -n '1,120p' "$BODY"
    rm -f "$HDR" "$BODY"
    exit 1
  done
  [ -n "$ZERO_JSON" ] || { echo "ERROR: failed to provision TiDB Zero instance after retries"; exit 1; }
  PARSED=$(printf '%s' "$ZERO_JSON" | node - <<'NODE'
const fs = require('fs');
const raw = fs.readFileSync(0, 'utf8');
let obj;
try {
  obj = JSON.parse(raw);
} catch (e) {
  console.error('ERROR: failed to parse Zero API response as JSON');
  process.exit(1);
}
const inst = obj.instance || obj;
const dsn = inst.connectionString || obj.connectionString || '';
if (!dsn) {
  console.error('ERROR: Zero API response missing connectionString');
  process.exit(1);
}
const claimUrl = (inst.claimInfo && inst.claimInfo.claimUrl) || '';
const expiresAt = inst.expiresAt || '';
process.stdout.write(`DSN=${dsn}\nCLAIM_URL=${claimUrl}\nEXPIRES_AT=${expiresAt}\n`);
NODE
)
  DSN=$(printf '%s\n' "$PARSED" | sed -n 's/^DSN=//p')
  CLAIM_URL=$(printf '%s\n' "$PARSED" | sed -n 's/^CLAIM_URL=//p')
  EXPIRES_AT=$(printf '%s\n' "$PARSED" | sed -n 's/^EXPIRES_AT=//p')
fi
[ -n "$DSN" ] || { echo "ERROR: failed to get DSN"; exit 1; }
printf '%s\n' "$DSN" > .teleport_dsn.tmp
printf '%s\n' "${CLAIM_URL:-}" > .teleport_claim_url.tmp
printf '%s\n' "${EXPIRES_AT:-}" > .teleport_expires_at.tmp
if [ -n "${EXPIRES_AT:-}" ]; then
  echo "DSN ready (expiresAt=$EXPIRES_AT)"
else
  echo "DSN ready"
fi
```

## Step 6 — Upload (auto chunk at 10MB)
```bash
set -euo pipefail
DSN="$(cat .teleport_dsn.tmp)"
CLAIM_URL="$(cat .teleport_claim_url.tmp 2>/dev/null || true)"
EXPIRES_AT="$(cat .teleport_expires_at.tmp 2>/dev/null || true)"
TRANSFER_ID_FILE=.teleport_transfer_id.tmp
if [ -f "$TRANSFER_ID_FILE" ]; then
  TRANSFER_ID="$(cat "$TRANSFER_ID_FILE")"
else
  TRANSFER_ID="tp_$(node -e "console.log(require('crypto').randomUUID().replace(/-/g,''))")"
  printf '%s\n' "$TRANSFER_ID" > "$TRANSFER_ID_FILE"
  chmod 600 "$TRANSFER_ID_FILE" 2>/dev/null || true
fi

UPLOAD_OUT=$(DSN="$DSN" ARCHIVE_PATH="$(pwd)/workspace.tar.gz" CLAIM_URL="$CLAIM_URL" EXPIRES_AT="$EXPIRES_AT" TRANSFER_ID="$TRANSFER_ID" npx -y -p mysql2 node - <<'NODE'
const fs = require('fs');
const crypto = require('crypto');
const mysql = require('mysql2/promise');
(async () => {
  const dsn = process.env.DSN;
  const archivePath = process.env.ARCHIVE_PATH;
  const claimUrl = process.env.CLAIM_URL || null;
  const expiresAt = process.env.EXPIRES_AT || null;
  const transferId = process.env.TRANSFER_ID || '';
  if (!transferId) throw new Error('missing TRANSFER_ID');
  const u = new URL(dsn);
  const db = (u.pathname || '/test').replace(/^\//,'') || 'test';
  const conn = await mysql.createConnection({ host: u.hostname, port: Number(u.port || 4000), user: decodeURIComponent(u.username), password: decodeURIComponent(u.password), database: db, ssl: { minVersion: 'TLSv1.2' } });
  try {
    await conn.query(`CREATE TABLE IF NOT EXISTS teleport_parts (transfer_id VARCHAR(64) NOT NULL, part_no INT NOT NULL, total_parts INT NOT NULL, data LONGBLOB NOT NULL, created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP, PRIMARY KEY (transfer_id, part_no))`);
    await conn.query(`CREATE TABLE IF NOT EXISTS teleport_meta (transfer_id VARCHAR(64) NOT NULL PRIMARY KEY, sha256 CHAR(64) NOT NULL, size_bytes BIGINT NOT NULL, parts INT NOT NULL, claim_url TEXT NULL, expires_at VARCHAR(64) NULL, created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP)`);

    const stat = fs.statSync(archivePath);
    const CHUNK = 10 * 1024 * 1024; // 10MB
    const hasher = crypto.createHash('sha256');

    let partNo = 0;
    const stream = fs.createReadStream(archivePath, { highWaterMark: CHUNK });
    for await (const chunk of stream) {
      partNo += 1;
      hasher.update(chunk);
      await conn.query(
        'INSERT INTO teleport_parts (transfer_id, part_no, total_parts, data) VALUES (?, ?, ?, ?) ON DUPLICATE KEY UPDATE total_parts=VALUES(total_parts), data=VALUES(data)',
        [transferId, partNo, 0, chunk]
      );
      process.stderr.write(`[upload] part ${partNo}\n`);
    }

    await conn.query('UPDATE teleport_parts SET total_parts=? WHERE transfer_id=?', [partNo, transferId]);
    const sha256 = hasher.digest('hex');
    await conn.query(
      'INSERT INTO teleport_meta (transfer_id, sha256, size_bytes, parts, claim_url, expires_at) VALUES (?, ?, ?, ?, ?, ?)',
      [transferId, sha256, stat.size, partNo, claimUrl, expiresAt]
    );
    process.stdout.write(`TRANSFER_ID=${transferId}\nPARTS=${partNo}\nSIZE_BYTES=${stat.size}\nSHA256=${sha256}\n`);
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
SHA256=$(sed -n 's/^SHA256=//p' .teleport_upload.tmp)
RAW="${DSN}|${TRANSFER_ID}"
CODE_PAYLOAD=$(printf '%s' "$RAW" | base64 | tr -d '\n' | tr '+/' '-_' | tr -d '=')
RESTORE_CODE="RESTORE-${CODE_PAYLOAD}"
printf '%s\n' "$RESTORE_CODE" > teleport_restore_code.txt
printf 'RESTORE_CODE=%s\nPARTS=%s\nSIZE_BYTES=%s\nSHA256=%s\n' "$RESTORE_CODE" "$PARTS" "$SIZE_BYTES" "$SHA256"
```

## Step 8 — Post chat summary
Use this exact template in chat:
```text
Backup completed.
- Scope: core
- Included paths: <...>
- Archive: <size>
- Parts: <n>
- SHA256: <sha256>
- Expires at: <expiresAt or unknown>
- Status: success

# B: restore
https://github.com/lilyjazz/openclaw-memory-teleport-skill/blob/main/skills/agent_teleport_restore/SKILL.md
Use this skill to restore OpenClaw’s memory.
🔐 Restore Code: <real code>

Recommendation:
Use GitHub to sync large project/code directories (repo clone/pull), and use Teleport for memory/persona state only.
```

## Step 9 — Cleanup sensitive temp files (recommended)
```bash
rm -f .teleport_dsn.tmp .teleport_claim_url.tmp .teleport_expires_at.tmp .teleport_upload.tmp .teleport_transfer_id.tmp
echo "Sensitive temp files cleaned"
```

## Failure retry index (backup)
- Fail at **Step 4 (build archive)**: fix scope/path, rerun Step 4.
- Fail at **Step 5 (get DSN)**: check network/TiDB env, rerun Step 5.
- Fail at **Step 6 (upload)**: rerun Step 6 only (reuses `.teleport_transfer_id.tmp` and UPSERT write for retry safety).
- Fail at **Step 7 (code generation)**: rerun Step 7 only.
