---
name: agent-teleport-backup
description: Create a portable OpenClaw workspace backup and upload it to TiDB/MySQL as chunked payload parts. Use on the source machine before migration.
---

# Title
Agent Teleport Backup (Source OpenClaw)

## Description
Run this on source OpenClaw. It creates `workspace.tar.gz`, uploads it to TiDB/MySQL, and prints one restore code.

- Zero-input UX.
- Outputs one restore code: `RESTORE-...`.
- Default chunk upload: if archive > 10MB, split into multiple parts and upload all parts.
- This code format is for UX/readability, not cryptographic security.

## Steps (source machine)
```bash
set -euo pipefail

IGNORE=(.git .venv venv env __pycache__ node_modules "*.log" "*.pyc" ".DS_Store" ".env" "*.pem" "*.key" "id_rsa" "id_dsa")
EX=(); for p in "${IGNORE[@]}"; do EX+=(--exclude="$p"); done

rm -f workspace.tar.gz
# shellcheck disable=SC2068
tar -czf workspace.tar.gz ${EX[@]} .

if [ -n "${TIDB_HOST:-}" ] && [ -n "${TIDB_USER:-}" ] && [ -n "${TIDB_PASSWORD:-}" ]; then
  TIDB_PORT="${TIDB_PORT:-4000}"
  DSN="mysql://${TIDB_USER}:${TIDB_PASSWORD}@${TIDB_HOST}:${TIDB_PORT}/test"
else
  DSN=$(curl -sS -X POST https://zero.tidbapi.com/v1alpha1/instances -H 'content-type: application/json' -d '{}' \
    | sed -n 's/.*"connectionString":"\([^"]*\)".*/\1/p')
  [ -z "$DSN" ] && { echo "ERROR: failed to provision DSN"; exit 1; }
fi

# Upload archive to TiDB/MySQL via Node (no mysql CLI required)
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
    }

    process.stdout.write(`TRANSFER_ID=${transferId}\nPARTS=${totalParts}\nSIZE_BYTES=${buf.length}\n`);
  } finally {
    await conn.end();
  }
})();
NODE
)

TRANSFER_ID=$(printf '%s\n' "$UPLOAD_OUT" | sed -n 's/^TRANSFER_ID=//p')
PARTS=$(printf '%s\n' "$UPLOAD_OUT" | sed -n 's/^PARTS=//p')
SIZE_BYTES=$(printf '%s\n' "$UPLOAD_OUT" | sed -n 's/^SIZE_BYTES=//p')
[ -z "$TRANSFER_ID" ] && { echo "ERROR: failed to get transfer_id"; exit 1; }

# Convert DSN+transfer to one restore code (base64url + prefix)
RAW="${DSN}|${TRANSFER_ID}"
CODE_PAYLOAD=$(printf '%s' "$RAW" | base64 | tr -d '\n' | tr '+/' '-_' | tr -d '=')
RESTORE_CODE="RESTORE-${CODE_PAYLOAD}"

echo "RESTORE_CODE=$RESTORE_CODE"
echo "TRANSFER_ID=$TRANSFER_ID"
echo "PARTS=$PARTS"
echo "SIZE_BYTES=$SIZE_BYTES"
printf '%s\n' "$RESTORE_CODE" > teleport_restore_code.txt
echo "Saved restore code: $(pwd)/teleport_restore_code.txt"

echo
echo "# B: restore"
echo "https://github.com/lilyjazz/openclaw-memory-teleport-skill/blob/main/skills/agent_teleport_restore/SKILL.md"
echo "Use this skill to restore OpenClaw’s memory."
echo "🔐 Restore Code: $RESTORE_CODE"
```

## Runtime requirements
- `bash`, `tar`, `curl`, `base64`, `sed`
- `node`, `npx` (uses `mysql2` via `npx -p mysql2`)

## Security
- DSN is restore key. Keep private.
- Code format is not strong encryption.
