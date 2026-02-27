---
name: agent-teleport-restore
description: Restore full OpenClaw core/workspace files in-place from TiDB/MySQL chunked teleport payload using one restore code (RESTORE) or plain DSN.
---

# Title
Agent Teleport Restore (Destination OpenClaw, In-Place)

## Description
Run this on destination OpenClaw to restore actual workspace files in-place.

## Chat Output Contract (MANDATORY)
Do not stop at CLI output. Always post restore result back to chat.

Required success format:
```text
Restore completed.
- Target path: <path>
- Safety backup: <backup_path>
- Payload parts: <n>
- Status: success
```

Required failure format:
```text
Restore failed.
- Stage: <decode|download|verify|extract>
- Error: <message>
- Status: failed
```

Supported input:
- Preferred one-code format: `RESTORE-...` (contains DSN + transfer_id)
- Backward compatible plain DSN: `mysql://...` (restores latest transfer)

## One-shot restore (recommended)
```bash
set -euo pipefail

echo "[1/7] Preparing restore..."
# Paste restore code (preferred): RESTORE-...
# Or plain DSN for compatibility: mysql://USER:PASSWORD@HOST:4000/test
RESTORE_CODE_RAW='RESTORE-PASTE_CODE_HERE'
RESTORE_CODE="$(printf '%s' "$RESTORE_CODE_RAW" | tr -d '[:space:]')"

TARGET_PATH="${TARGET_PATH:-/home/ubuntu/.openclaw/workspace}"

for c in bash tar date mktemp base64 node npx; do
  command -v "$c" >/dev/null 2>&1 || { echo "ERROR: missing command: $c"; exit 1; }
done

echo "[2/7] Decoding restore code..."
TRANSFER_ID=""
if [[ "$RESTORE_CODE" == RESTORE-* ]]; then
  PAYLOAD="${RESTORE_CODE#RESTORE-}"
  B64=$(printf '%s' "$PAYLOAD" | tr '-_' '+/')
  PAD=$(( (4 - ${#B64} % 4) % 4 ))
  [ "$PAD" -eq 2 ] && B64="${B64}=="
  [ "$PAD" -eq 1 ] && B64="${B64}="
  DEC=$(printf '%s' "$B64" | base64 -d 2>/dev/null || true)
  DSN="${DEC%%|*}"
  [ "$DEC" != "$DSN" ] && TRANSFER_ID="${DEC#*|}"
else
  DSN="$RESTORE_CODE"
fi

case "$DSN" in
  mysql://* ) ;;
  * ) echo "ERROR: invalid restore code / DSN"; exit 1 ;;
esac

echo "[3/7] Preparing temp workspace..."
TMPDIR=$(mktemp -d)
ARCHIVE="$TMPDIR/workspace.tar.gz"

echo "[4/7] Downloading payload parts from TiDB/MySQL..."
# Fetch payload via Node (no mysql CLI required)
DSN="$DSN" TRANSFER_ID="$TRANSFER_ID" ARCHIVE_PATH="$ARCHIVE" npx -y -p mysql2 node - <<'NODE'
const fs = require('fs');
const mysql = require('mysql2/promise');

(async () => {
  const dsn = process.env.DSN;
  let transferId = process.env.TRANSFER_ID || '';
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
    // Prefer new chunked table
    const [tableRows] = await conn.query("SHOW TABLES LIKE 'teleport_parts'");
    if (tableRows.length) {
      if (!transferId) {
        const [latest] = await conn.query('SELECT transfer_id FROM teleport_parts ORDER BY created_at DESC LIMIT 1');
        if (!latest.length) {
          console.error('ERROR: no transfer found in teleport_parts');
          process.exit(1);
        }
        transferId = latest[0].transfer_id;
      }

      const [rows] = await conn.query(
        'SELECT part_no, total_parts, data FROM teleport_parts WHERE transfer_id=? ORDER BY part_no ASC',
        [transferId]
      );

      if (!rows.length) {
        console.error('ERROR: transfer_id not found in teleport_parts');
        process.exit(1);
      }

      const total = rows[0].total_parts;
      if (rows.length !== total) {
        console.error(`ERROR: missing parts (got ${rows.length}, expected ${total})`);
        process.exit(1);
      }

      const bufs = [];
      for (let i = 0; i < rows.length; i++) {
        if (rows[i].part_no !== i + 1) {
          console.error(`ERROR: part order mismatch at index ${i}`);
          process.exit(1);
        }
        bufs.push(rows[i].data);
        process.stderr.write(`[download] part ${i + 1}/${rows.length}\n`);
      }
      fs.writeFileSync(archivePath, Buffer.concat(bufs));
      return;
    }

    // Backward compatibility: old single-row table
    const [rows] = await conn.query('SELECT data FROM teleport WHERE id=1');
    if (!rows.length || !rows[0].data) {
      console.error('ERROR: payload not found (invalid/expired code, wrong DB, or table empty)');
      process.exit(1);
    }
    fs.writeFileSync(archivePath, rows[0].data);
  } finally {
    await conn.end();
  }
})();
NODE

echo "[5/7] Verifying archive integrity..."
tar -tzf "$ARCHIVE" >/dev/null || { echo "ERROR: invalid/corrupted archive"; exit 1; }

echo "[6/7] Creating safety backup..."
STAMP=$(date +%Y%m%d-%H%M%S)
BACKUP_PATH="${TARGET_PATH}_backup_${STAMP}"
if [ -d "$TARGET_PATH" ]; then
  cp -a "$TARGET_PATH" "$BACKUP_PATH"
  echo "Safety backup: $BACKUP_PATH"
else
  mkdir -p "$TARGET_PATH"
fi

echo "[7/7] Extracting archive to target path..."
tar -xzf "$ARCHIVE" -C "$TARGET_PATH"

echo "Restore completed."
echo "Target path: $TARGET_PATH"
```

## Runtime requirements
- `bash`, `tar`, `date`, `mktemp`, `base64`
- `node`, `npx` (uses `mysql2` via `npx -p mysql2`)

## Notes
- Restores files in-place (not CSV/JSON export).
- Supports multi-part archives automatically.
- Use `TARGET_PATH=/your/path` to override destination.
