---
name: agent-teleport-restore
description: Restore full OpenClaw core/workspace files in-place from TiDB/MySQL teleport payload using one restore code (OCMT1) or plain DSN. Use on destination machine after migration.
---

# Title
Agent Teleport Restore (Destination OpenClaw, In-Place)

## Description
Run this on destination OpenClaw to restore actual workspace files in-place.

Supported input:
- Preferred one-code format: `OCMT1-...`
- Backward compatible plain DSN: `mysql://...`

## One-shot restore (recommended)
```bash
set -euo pipefail

# Paste restore code (preferred):
# OCMT1-...
# or plain DSN for compatibility: mysql://USER:PASSWORD@HOST:4000/test
RESTORE_CODE_RAW='OCMT1-PASTE_CODE_HERE'
RESTORE_CODE="$(printf '%s' "$RESTORE_CODE_RAW" | tr -d '[:space:]')"

TARGET_PATH="${TARGET_PATH:-/home/ubuntu/.openclaw/workspace}"

for c in bash mysql tar xxd date mktemp; do
  command -v "$c" >/dev/null 2>&1 || { echo "ERROR: missing command: $c"; exit 1; }
done

if [[ "$RESTORE_CODE" == OCMT1-* ]]; then
  command -v base64 >/dev/null 2>&1 || { echo "ERROR: missing command: base64"; exit 1; }
  PAYLOAD="${RESTORE_CODE#OCMT1-}"
  B64=$(printf '%s' "$PAYLOAD" | tr '-_' '+/')
  PAD=$(( (4 - ${#B64} % 4) % 4 ))
  [ "$PAD" -eq 2 ] && B64="${B64}=="
  [ "$PAD" -eq 1 ] && B64="${B64}="
  DSN=$(printf '%s' "$B64" | base64 -d 2>/dev/null || true)
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
- `bash`, `mysql`, `tar`, `xxd`, `date`, `mktemp`, `base64`

## Notes
- Restores files in-place (not CSV/JSON table export).
- Use `TARGET_PATH=/your/path` to override destination.
- Keep restore code private.
