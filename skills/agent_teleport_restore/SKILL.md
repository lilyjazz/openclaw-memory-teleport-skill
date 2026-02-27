---
name: agent-teleport-restore
description: Restore full OpenClaw core/workspace files in-place from TiDB/MySQL teleport payload using DSN or encrypted OMT1 restore code. Use on destination machine after migration.
---

# Title
Agent Teleport Restore (Destination OpenClaw, In-Place)

## Description
Run this on destination OpenClaw to restore actual workspace files in-place.

Supported inputs:
- Plain DSN: `mysql://...`
- Encrypted code: `OMT1:<base64>` (will prompt passphrase interactively)

## One-shot restore (recommended)
```bash
set -euo pipefail

# Paste one of:
# 1) DSN:  mysql://USER:PASSWORD@HOST:4000/test
# 2) OMT1: OMT1:BASE64...
RESTORE_CODE_RAW='mysql://USER:PASSWORD@HOST:4000/test'
RESTORE_CODE="$(printf '%s' "$RESTORE_CODE_RAW" | tr -d '[:space:]')"

TARGET_PATH="${TARGET_PATH:-/home/ubuntu/.openclaw/workspace}"

for c in bash mysql tar xxd date mktemp; do
  command -v "$c" >/dev/null 2>&1 || { echo "ERROR: missing command: $c"; exit 1; }
done

# Resolve DSN from restore code
if [[ "$RESTORE_CODE" == OMT1:* ]]; then
  command -v openssl >/dev/null 2>&1 || { echo "ERROR: openssl required for OMT1 code"; exit 1; }
  read -rsp "Enter passphrase for OMT1 restore code: " TELEPORT_KEY
  echo
  [ -n "${TELEPORT_KEY:-}" ] || { echo "ERROR: passphrase is required"; exit 1; }
  PAYLOAD="${RESTORE_CODE#OMT1:}"
  DSN=$(printf '%s' "$PAYLOAD" | openssl enc -d -aes-256-cbc -pbkdf2 -salt -pass pass:"$TELEPORT_KEY" -base64 -A)
else
  DSN="$RESTORE_CODE"
fi

case "$DSN" in
  mysql://* ) ;;
  * ) echo "ERROR: invalid restore code / DSN"; exit 1 ;;
esac

# Parse DSN
TMP="${DSN#mysql://}"; AUTH="${TMP%@*}"; HOSTDB="${TMP#*@}"
USER="${AUTH%%:*}"; PASS="${AUTH#*:}"
HOSTPORT="${HOSTDB%%/*}"; DB="${HOSTDB#*/}"
HOST="${HOSTPORT%%:*}"; PORT="${HOSTPORT#*:}"
[ "$DB" = "$HOSTDB" ] && DB="test"

HEX=$(mysql --host="$HOST" --port="$PORT" --user="$USER" --password="$PASS" --database="$DB" \
  --ssl-mode=REQUIRED -N -B -e "SELECT HEX(data) FROM teleport WHERE id=1")

if [ -z "${HEX:-}" ]; then
  echo "ERROR: teleport payload not found (invalid/expired code, wrong DB, or table empty)"
  exit 1
fi

TMPDIR=$(mktemp -d)
ARCHIVE="$TMPDIR/workspace.tar.gz"
printf '%s' "$HEX" | xxd -r -p > "$ARCHIVE"
tar -tzf "$ARCHIVE" >/dev/null || { echo "ERROR: invalid/corrupted archive"; exit 1; }

STAMP=$(date +%Y%m%d-%H%M%S)
BACKUP_PATH="${TARGET_PATH}_backup_${STAMP}"
if [ -d "$TARGET_PATH" ]; then
  cp -a "$TARGET_PATH" "$BACKUP_PATH"
  echo "Safety backup: $BACKUP_PATH"
else
  mkdir -p "$TARGET_PATH"
fi

tar -xzf "$ARCHIVE" -C "$TARGET_PATH"

echo "Restore completed."
echo "Target path: $TARGET_PATH"
```

## Runtime requirements
- `bash`, `mysql`, `tar`, `xxd`, `date`, `mktemp`
- If using OMT1 code: `openssl`

## Notes
- Restores files in-place (not CSV/JSON table export).
- Use `TARGET_PATH=/your/path` to override destination.
- Keep restore code private.
