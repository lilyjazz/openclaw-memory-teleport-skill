# OpenClaw Memory Teleport Skill

Portable backup/restore skills for migrating OpenClaw workspace state across machines.

## Overview
This repo provides a shell-based migration flow:
1. Backup on source machine
2. Upload payload to TiDB/MySQL (`teleport` table)
3. Restore in-place on destination machine

It restores real workspace files (not just DB row export), including memory/config/skills.

## Included Skills
- `skills/agent_teleport_backup/SKILL.md` — source-side backup/upload
- `skills/agent_teleport_restore/SKILL.md` — destination-side in-place restore

## Copy/Paste Quick Start

> Run on **OpenClaw A** (source), then run on **OpenClaw B** (destination).

### A) Source machine — Backup + Upload
```bash
cd /home/ubuntu/.openclaw/workspace
set -euo pipefail

IGNORE=(.git .venv venv env __pycache__ node_modules "*.log" "*.pyc" ".DS_Store" ".env" "*.pem" "*.key" "id_rsa" "id_dsa")
EX=(); for p in "${IGNORE[@]}"; do EX+=(--exclude="$p"); done

rm -f workspace.tar.gz
# shellcheck disable=SC2068
tar -czf workspace.tar.gz ${EX[@]} .

SZ=$(du -m workspace.tar.gz | awk '{print $1}')
if [ "$SZ" -gt 32 ]; then
  echo "ERROR: archive ${SZ}MB > 32MB"; exit 1
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

echo "RESTORE_DSN=$DSN"
printf '%s\n' "$DSN" > teleport_restore_dsn.txt
echo "Saved: $(pwd)/teleport_restore_dsn.txt"
```

### B) Destination machine — Restore in-place
```bash
cd /home/ubuntu/.openclaw/workspace
set -euo pipefail

# Paste DSN from source (no spaces)
DSN_RAW='mysql://USER:PASSWORD@HOST:4000/test'
DSN="$(printf '%s' "$DSN_RAW" | tr -d '[:space:]')"

TARGET_PATH="${TARGET_PATH:-/home/ubuntu/.openclaw/workspace}"

for c in bash mysql tar xxd date mktemp; do
  command -v "$c" >/dev/null 2>&1 || { echo "ERROR: missing command: $c"; exit 1; }
done

TMP="${DSN#mysql://}"; AUTH="${TMP%@*}"; HOSTDB="${TMP#*@}"
USER="${AUTH%%:*}"; PASS="${AUTH#*:}"
HOSTPORT="${HOSTDB%%/*}"; DB="${HOSTDB#*/}"
HOST="${HOSTPORT%%:*}"; PORT="${HOSTPORT#*:}"
[ "$DB" = "$HOSTDB" ] && DB="test"

HEX=$(mysql --host="$HOST" --port="$PORT" --user="$USER" --password="$PASS" --database="$DB" --ssl-mode=REQUIRED -N -B -e "SELECT HEX(data) FROM teleport WHERE id=1")
[ -z "${HEX:-}" ] && { echo "ERROR: payload not found"; exit 1; }

TMPDIR=$(mktemp -d)
ARCHIVE="$TMPDIR/workspace.tar.gz"
printf '%s' "$HEX" | xxd -r -p > "$ARCHIVE"
tar -tzf "$ARCHIVE" >/dev/null

STAMP=$(date +%Y%m%d-%H%M%S)
BACKUP_PATH="${TARGET_PATH}_backup_${STAMP}"
if [ -d "$TARGET_PATH" ]; then
  cp -a "$TARGET_PATH" "$BACKUP_PATH"
  echo "Safety backup: $BACKUP_PATH"
else
  mkdir -p "$TARGET_PATH"
fi

tar -xzf "$ARCHIVE" -C "$TARGET_PATH"
echo "Restore completed: $TARGET_PATH"
```

## Verified Workflow (A -> B)
Validated with real run:
- Source packed and uploaded `workspace.tar.gz`
- Destination restored to `/home/ubuntu/.openclaw/workspace`
- Safety backup created before overwrite
- Core files successfully restored in-place

## Requirements
### Backup side
- `bash`, `tar`, `mysql`, `curl`, `xxd`, `sed`, `awk`

### Restore side
- `bash`, `mysql`, `tar`, `xxd`, `date`, `mktemp`

## Troubleshooting
- **DSN copied with spaces/newlines**: keep `DSN_RAW` and sanitize via `tr -d '[:space:]'` (already in example).
- **`mysql: command not found`**: install mysql client first.
- **`payload not found`**: DSN wrong, DB wrong, or `teleport.id=1` missing.
- **archive too large**: clean caches/build outputs and retry backup.

## Security Notes
- DSN is a restore key (secret).
- Never share DSN in public channels.
- Delete/rotate temporary DB resources after restore for sensitive data.

## Maintainer
GitHub: [@lilyjazz](https://github.com/lilyjazz)
