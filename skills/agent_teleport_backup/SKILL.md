---
name: agent-teleport-backup
description: Create a portable OpenClaw workspace backup and upload it to TiDB/MySQL as a restore payload. Use on the source machine before migration.
---

# Title
Agent Teleport Backup (Source OpenClaw)

## Description
Run this on the source OpenClaw. It creates `workspace.tar.gz`, stores it in TiDB/MySQL `teleport.data` (LONGBLOB), and prints a restore code.

- If `TELEPORT_KEY` is set, outputs encrypted code: `OMT1:<base64>`
- Otherwise outputs plain DSN

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

TMP="${DSN#mysql://}"; AUTH="${TMP%@*}"; HOSTDB="${TMP#*@}"
USER="${AUTH%%:*}"; PASS="${AUTH#*:}"
HOSTPORT="${HOSTDB%%/*}"; DB="${HOSTDB#*/}"
HOST="${HOSTPORT%%:*}"; PORT="${HOSTPORT#*:}"
[ "$DB" = "$HOSTDB" ] && DB="test"

HEX=$(xxd -p workspace.tar.gz | tr -d '\n')

mysql --host="$HOST" --port="$PORT" --user="$USER" --password="$PASS" --database="$DB" --ssl-mode=REQUIRED -e \
"CREATE TABLE IF NOT EXISTS teleport (id INT PRIMARY KEY, data LONGBLOB, created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP);
 REPLACE INTO teleport (id,data) VALUES (1, UNHEX('$HEX'));"

if [ -n "${TELEPORT_KEY:-}" ]; then
  ENCRYPTED=$(printf '%s' "$DSN" | openssl enc -aes-256-cbc -pbkdf2 -salt -pass env:TELEPORT_KEY -base64 -A)
  CODE="OMT1:${ENCRYPTED}"
  echo "RESTORE_CODE=$CODE"
  printf '%s\n' "$CODE" > teleport_restore_code.txt
  echo "Saved encrypted code: $(pwd)/teleport_restore_code.txt"
else
  echo "RESTORE_DSN=$DSN"
  printf '%s\n' "$DSN" > teleport_restore_dsn.txt
  echo "Saved plain DSN: $(pwd)/teleport_restore_dsn.txt"
  echo "TIP: set TELEPORT_KEY to output encrypted OMT1 code."
fi
```

## Runtime requirements
- `bash`, `tar`, `mysql`, `curl`, `xxd`, `sed`, `awk`
- Optional for encrypted code: `openssl`

## Security
- DSN is restore key. Keep private.
- Prefer encrypted code output (`TELEPORT_KEY`) for sharing between operators.
