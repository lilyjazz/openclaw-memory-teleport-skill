---
name: agent-teleport-restore
description: Restore full OpenClaw core/workspace files in-place from TiDB/MySQL teleport payload using DSN. Use on destination machine after migration; creates safety backup and extracts archive to target path.
---

# Title
Agent Teleport Restore (Destination OpenClaw, In-Place)

## Description
Run this on the **new/destination OpenClaw** to restore the **actual workspace files** (MEMORY.md, memory/, skills/, configs, etc.) to the real path, not a DB export folder.

This version adds:
- preflight dependency checks
- target path control
- automatic safety backup before overwrite
- clearer errors for missing DSN/table/payload
- DSN whitespace sanitization

## One-shot restore (recommended)
```bash
set -euo pipefail

# 1) REQUIRED: paste restore DSN (can include accidental spaces; they are removed)
DSN_RAW='mysql://USER:PASSWORD@HOST:4000/test'
DSN="$(printf '%s' "$DSN_RAW" | tr -d '[:space:]')"

# 2) OPTIONAL: restore target (default = real OpenClaw workspace)
TARGET_PATH="${TARGET_PATH:-/home/ubuntu/.openclaw/workspace}"

# --- Preflight ---
for c in bash mysql tar xxd date mktemp; do
  command -v "$c" >/dev/null 2>&1 || { echo "ERROR: missing command: $c"; exit 1; }
done

case "$DSN" in
  mysql://* ) ;;
  * ) echo "ERROR: DSN must start with mysql://"; exit 1 ;;
esac

# Parse DSN: mysql://user:pass@host:port/db
TMP="${DSN#mysql://}"; AUTH="${TMP%@*}"; HOSTDB="${TMP#*@}"
USER="${AUTH%%:*}"; PASS="${AUTH#*:}"
HOSTPORT="${HOSTDB%%/*}"; DB="${HOSTDB#*/}"
HOST="${HOSTPORT%%:*}"; PORT="${HOSTPORT#*:}"
[ "$DB" = "$HOSTDB" ] && DB="test"

# --- Fetch payload ---
HEX=$(mysql --host="$HOST" --port="$PORT" --user="$USER" --password="$PASS" --database="$DB" \
  --ssl-mode=REQUIRED -N -B -e "SELECT HEX(data) FROM teleport WHERE id=1")

if [ -z "${HEX:-}" ]; then
  echo "ERROR: teleport payload not found (invalid/expired DSN, wrong DB, or table empty)"
  exit 1
fi

# --- Build archive in temp dir ---
TMPDIR=$(mktemp -d)
ARCHIVE="$TMPDIR/workspace.tar.gz"
printf '%s' "$HEX" | xxd -r -p > "$ARCHIVE"
tar -tzf "$ARCHIVE" >/dev/null || { echo "ERROR: invalid/corrupted archive"; exit 1; }

# --- Safety backup before overwrite ---
STAMP=$(date +%Y%m%d-%H%M%S)
BACKUP_PATH="${TARGET_PATH}_backup_${STAMP}"
if [ -d "$TARGET_PATH" ]; then
  cp -a "$TARGET_PATH" "$BACKUP_PATH"
  echo "Safety backup: $BACKUP_PATH"
else
  mkdir -p "$TARGET_PATH"
fi

# --- Restore in-place ---
tar -xzf "$ARCHIVE" -C "$TARGET_PATH"

echo "Restore completed."
echo "Target path: $TARGET_PATH"
```

## Expected result
After success, `$TARGET_PATH` should contain restored core files such as:
- `MEMORY.md`
- `memory/`
- `skills/`
- other workspace root files from snapshot

## Runtime requirements
- `bash`, `mysql`, `tar`, `xxd`, `date`, `mktemp`

## Notes
- This restores **files**, not table-to-CSV export.
- If you want non-default destination, set `TARGET_PATH=/your/path` before running.
- Treat DSN as secret (whoever has DSN can restore payload).
