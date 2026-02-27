---
name: agent-teleport-backup
description: Create a portable OpenClaw workspace backup and upload it to TiDB/MySQL as chunked payload parts. Use on the source machine before migration.
---

# Title
Agent Teleport Backup (Source OpenClaw)

## Description
Run this on source OpenClaw. It previews workspace size, asks user to confirm backup scope, creates `workspace.tar.gz`, uploads to TiDB/MySQL, and prints one restore code.

## Chat Output Contract (important)
After running backup, the agent must send a concise chat summary (not only CLI logs):
- selected scope (`full/core/custom`)
- archive size
- part count
- restore handoff block with real code

Required handoff message format:
```text
# B: restore
https://github.com/lilyjazz/openclaw-memory-teleport-skill/blob/main/skills/agent_teleport_restore/SKILL.md
Use this skill to restore OpenClaw’s memory.
🔐 Restore Code: <real code>
```

- Zero-input for credentials when env is set / auto-provision is available.
- Interactive scope confirmation before backup.
- Outputs one code: `RESTORE-...`.
- Default chunk upload: if archive > 10MB, split into multiple parts and upload all parts.
- Code format is for UX/readability, not cryptographic security.

## Steps (source machine)
```bash
set -euo pipefail

echo "[1/9] Preparing backup..."
cd /home/ubuntu/.openclaw/workspace

IGNORE=(.git .venv venv env __pycache__ node_modules "*.log" "*.pyc" ".DS_Store" ".env" "*.pem" "*.key" "id_rsa" "id_dsa")

echo "[2/9] Scanning workspace size (folders only)..."
TOTAL=$(du -sh . | awk '{print $1}')
echo "Workspace total: $TOTAL"
echo
echo "Top folders by size:"
du -h --max-depth=1 . 2>/dev/null | sort -hr | head -n 20
echo

echo "Choose backup scope:"
echo "  1) full  - entire workspace (default)"
echo "  2) core  - AGENTS.md SOUL.md USER.md MEMORY.md TOOLS.md IDENTITY.md HEARTBEAT.md memory/ skills/"
echo "  3) custom - enter relative paths (space-separated)"
read -rp "Select [1/2/3] (default 1): " SCOPE_CHOICE
SCOPE_CHOICE="${SCOPE_CHOICE:-1}"

SOURCES=()
case "$SCOPE_CHOICE" in
  2)
    for p in AGENTS.md SOUL.md USER.md MEMORY.md TOOLS.md IDENTITY.md HEARTBEAT.md memory skills; do
      [ -e "$p" ] && SOURCES+=("$p")
    done
    [ ${#SOURCES[@]} -eq 0 ] && { echo "ERROR: core scope resolved to empty set"; exit 1; }
    ;;
  3)
    read -rp "Enter relative paths to include: " -a USER_PATHS
    [ ${#USER_PATHS[@]} -eq 0 ] && { echo "ERROR: no custom paths provided"; exit 1; }
    for p in "${USER_PATHS[@]}"; do
      [ -e "$p" ] || { echo "ERROR: path not found: $p"; exit 1; }
      SOURCES+=("$p")
    done
    ;;
  *)
    SOURCES=(.)
    ;;
esac

echo "Selected sources: ${SOURCES[*]}"

# Build tar exclude args
EX=()
for p in "${IGNORE[@]}"; do EX+=(--exclude="$p"); done

echo "[3/9] Compressing selected scope..."
rm -f workspace.tar.gz
# shellcheck disable=SC2068,SC2068
tar -czf workspace.tar.gz ${EX[@]} "${SOURCES[@]}"

SZ=$(du -m workspace.tar.gz | awk '{print $1}')
echo "Archive size: ${SZ}MB"

echo "[4/9] Resolving upload DSN..."
if [ -n "${TIDB_HOST:-}" ] && [ -n "${TIDB_USER:-}" ] && [ -n "${TIDB_PASSWORD:-}" ]; then
  TIDB_PORT="${TIDB_PORT:-4000}"
  DSN="mysql://${TIDB_USER}:${TIDB_PASSWORD}@${TIDB_HOST}:${TIDB_PORT}/test"
else
  DSN=$(curl -sS -X POST https://zero.tidbapi.com/v1alpha1/instances -H 'content-type: application/json' -d '{}' \
    | sed -n 's/.*"connectionString":"\([^"]*\)".*/\1/p')
  [ -z "$DSN" ] && { echo "ERROR: failed to provision DSN"; exit 1; }
fi

echo "[5/9] Uploading archive to TiDB/MySQL (chunked if >10MB)..."
UPLOAD_OUT=$(DSN="$DSN" ARCHIVE_PATH="$(pwd)/workspace.tar.gz" npx -y -p mysql2 node - <<'NODE'
const fs = require('fs');
const mysql = require('mysql2/promise');

(async () => {
  const dsn = process.env.DSN;
  const archivePath = process.env.ARCHIVE_PATH;
  const u = new URL(dsn);
  const db = (u.pathname || '/test').replace(/^\//,'') || 'test';

  const conn = await mysql.createConnection({
    host: u.hostname,
    port: Number(u.port || 4000),
    user: decodeURIComponent(u.username),
    password: decodeURIComponent(u.password),
    database: db,
    ssl: { minVersion: 'TLSv1.2' }
  });

  try {
    await conn.query(`CREATE TABLE IF NOT EXISTS teleport_parts (
      transfer_id VARCHAR(64) NOT NULL,
      part_no INT NOT NULL,
      total_parts INT NOT NULL,
      data LONGBLOB NOT NULL,
      created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
      PRIMARY KEY (transfer_id, part_no)
    )`);

    const buf = fs.readFileSync(archivePath);
    const CHUNK = 10 * 1024 * 1024; // 10MB
    const totalParts = Math.ceil(buf.length / CHUNK) || 1;
    const transferId = `tp_${Date.now().toString(36)}_${Math.random().toString(36).slice(2,8)}`;

    for (let i = 0; i < totalParts; i++) {
      const part = buf.subarray(i * CHUNK, Math.min((i + 1) * CHUNK, buf.length));
      await conn.query(
        'INSERT INTO teleport_parts (transfer_id, part_no, total_parts, data) VALUES (?, ?, ?, ?)',
        [transferId, i + 1, totalParts, part]
      );
      process.stderr.write(`[upload] part ${i + 1}/${totalParts}\n`);
    }

    process.stdout.write(`TRANSFER_ID=${transferId}\nPARTS=${totalParts}\nSIZE_BYTES=${buf.length}\n`);
  } finally {
    await conn.end();
  }
})();
NODE
)

echo "[6/9] Finalizing transfer metadata..."
TRANSFER_ID=$(printf '%s\n' "$UPLOAD_OUT" | sed -n 's/^TRANSFER_ID=//p')
PARTS=$(printf '%s\n' "$UPLOAD_OUT" | sed -n 's/^PARTS=//p')
SIZE_BYTES=$(printf '%s\n' "$UPLOAD_OUT" | sed -n 's/^SIZE_BYTES=//p')
[ -z "$TRANSFER_ID" ] && { echo "ERROR: failed to get transfer_id"; exit 1; }

echo "[7/9] Generating restore code..."
RAW="${DSN}|${TRANSFER_ID}"
CODE_PAYLOAD=$(printf '%s' "$RAW" | base64 | tr -d '\n' | tr '+/' '-_' | tr -d '=')
RESTORE_CODE="RESTORE-${CODE_PAYLOAD}"

echo "[8/9] Writing local outputs..."
echo "RESTORE_CODE=$RESTORE_CODE"
echo "TRANSFER_ID=$TRANSFER_ID"
echo "PARTS=$PARTS"
echo "SIZE_BYTES=$SIZE_BYTES"
printf '%s\n' "$RESTORE_CODE" > teleport_restore_code.txt
echo "Saved restore code: $(pwd)/teleport_restore_code.txt"

echo "[9/9] Backup complete."
echo
echo "# B: restore"
echo "https://github.com/lilyjazz/openclaw-memory-teleport-skill/blob/main/skills/agent_teleport_restore/SKILL.md"
echo "Use this skill to restore OpenClaw’s memory."
echo "🔐 Restore Code: $RESTORE_CODE"
```

## Runtime requirements
- `bash`, `tar`, `curl`, `base64`, `sed`, `du`, `sort`
- `node`, `npx` (uses `mysql2` via `npx -p mysql2`)

## Security
- DSN is restore key. Keep private.
- Code format is not strong encryption.
