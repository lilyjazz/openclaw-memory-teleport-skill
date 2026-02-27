---
name: agent-teleport-backup
description: Create a portable OpenClaw workspace backup and upload it to TiDB/MySQL as a restore payload. Use on the source machine before migration.
---

# Title
Agent Teleport Backup (Source OpenClaw)

## Description
Run this on the source OpenClaw. It creates `workspace.tar.gz`, stores it in TiDB/MySQL `teleport.data` (LONGBLOB), and prints a single restore code.

- Zero-input UX.
- Outputs one code: `OCMT1-...` (DSN transformed in-skill).
- This is obfuscation for UX/readability, not cryptographic security.

## Steps (source machine)
```bash
set -euo pipefail

IGNORE=(.git .venv venv env __pycache__ node_modules "*.log" "*.pyc" ".DS_Store" ".env" "*.pem" "*.key" "id_rsa" "id_dsa")
EX=(); for p in "${IGNORE[@]}"; do EX+=(--exclude="$p"); done

rm -f workspace.tar.gz
# shellcheck disable=SC2068
tar -czf workspace.tar.gz ${EX[@]} .

SZ=$(du -m workspace.tar.gz | awk '{print $1}')
if [ "$SZ" -gt 32 ]; then
  echo "ERROR: archive ${SZ}MB > 32MB. Clean cache/build files first."; exit 1
fi

if [ -n "${TIDB_HOST:-}" ] && [ -n "${TIDB_USER:-}" ] && [ -n "${TIDB_PASSWORD:-}" ]; then
  TIDB_PORT="${TIDB_PORT:-4000}"
  DSN="mysql://${TIDB_USER}:${TIDB_PASSWORD}@${TIDB_HOST}:${TIDB_PORT}/test"
else
  DSN=$(curl -sS -X POST https://zero.tidbapi.com/v1alpha1/instances -H 'content-type: application/json' -d '{}' \
    | sed -n 's/.*"connectionString":"\([^"]*\)".*/\1/p')
  [ -z "$DSN" ] && { echo "ERROR: failed to provision DSN"; exit 1; }
fi

# Upload archive to TiDB/MySQL via Node (no mysql CLI required)
DSN="$DSN" ARCHIVE_PATH="$(pwd)/workspace.tar.gz" npx -y -p mysql2 node - <<'NODE'
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
    await conn.query('CREATE TABLE IF NOT EXISTS teleport (id INT PRIMARY KEY, data LONGBLOB, created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP)');
    const blob = fs.readFileSync(archivePath);
    await conn.query('REPLACE INTO teleport (id, data) VALUES (1, ?)', [blob]);
  } finally {
    await conn.end();
  }
})();
NODE

# Convert DSN to one restore code (base64url + prefix)
CODE_PAYLOAD=$(printf '%s' "$DSN" | base64 | tr -d '\n' | tr '+/' '-_' | tr -d '=')
RESTORE_CODE="OCMT1-${CODE_PAYLOAD}"

echo "RESTORE_CODE=$RESTORE_CODE"
printf '%s\n' "$RESTORE_CODE" > teleport_restore_code.txt
echo "Saved restore code: $(pwd)/teleport_restore_code.txt"
```

## Runtime requirements
- `bash`, `tar`, `curl`, `base64`
- `node`, `npx` (uses `mysql2` via `npx -p mysql2`)

## Security
- DSN is restore key. Keep private.
- This mode prioritizes simplicity over cryptographic secrecy; treat DSN as secret.
