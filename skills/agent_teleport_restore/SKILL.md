---
name: agent-teleport-restore
description: Restore full OpenClaw core/workspace files in-place from TiDB/MySQL chunked teleport payload using one restore code (RESTORE) or plain DSN.
---

# Agent Teleport Restore (Destination OpenClaw)

## Execution model (important)
Run **one step at a time**. Do not send one giant command block.
After each step, post progress in chat.

## Chat Output Contract (MANDATORY)
Success:
```text
Restore completed.
- Target path: <path>
- Safety backup: <backup_path>
- Payload parts: <n>
- Status: success
```

Failure:
```text
Restore failed.
- Stage: <decode|download|verify|extract>
- Error: <message>
- Status: failed
```

---

## Step 1 — Set target path
```bash
TARGET_PATH="${TARGET_PATH:-/home/ubuntu/.openclaw/workspace}"
printf '%s\n' "$TARGET_PATH" > .teleport_target.tmp
echo "$TARGET_PATH"
```

## Step 2 — Paste restore code
```bash
RESTORE_CODE_RAW='RESTORE-PASTE_CODE_HERE'
RESTORE_CODE="$(printf '%s' "$RESTORE_CODE_RAW" | tr -d '[:space:]')"
printf '%s\n' "$RESTORE_CODE" > .teleport_code.tmp
echo "Code loaded"
```

## Step 3 — Decode restore code to DSN + transfer id
```bash
set -euo pipefail
RESTORE_CODE="$(cat .teleport_code.tmp)"
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
case "$DSN" in mysql://*) ;; *) echo "ERROR: invalid restore code / DSN"; exit 1;; esac
printf '%s\n' "$DSN" > .teleport_dsn.tmp
printf '%s\n' "$TRANSFER_ID" > .teleport_transfer.tmp
echo "Decode OK"
```

## Step 4 — Download payload parts and assemble archive
```bash
set -euo pipefail
TMPDIR=$(mktemp -d)
ARCHIVE="$TMPDIR/workspace.tar.gz"
DSN="$(cat .teleport_dsn.tmp)"
TRANSFER_ID="$(cat .teleport_transfer.tmp)"

DSN="$DSN" TRANSFER_ID="$TRANSFER_ID" ARCHIVE_PATH="$ARCHIVE" npx -y -p mysql2 node - <<'NODE'
const fs = require('fs');
const mysql = require('mysql2/promise');
(async () => {
  const u = new URL(process.env.DSN);
  const db = (u.pathname || '/test').replace(/^\//,'') || 'test';
  const conn = await mysql.createConnection({ host: u.hostname, port: Number(u.port || 4000), user: decodeURIComponent(u.username), password: decodeURIComponent(u.password), database: db, ssl: { minVersion: 'TLSv1.2' } });
  let partCount = 1;
  try {
    let transferId = process.env.TRANSFER_ID || '';
    const [tbl] = await conn.query("SHOW TABLES LIKE 'teleport_parts'");
    if (tbl.length) {
      if (!transferId) {
        const [latest] = await conn.query('SELECT transfer_id FROM teleport_parts ORDER BY created_at DESC LIMIT 1');
        if (!latest.length) throw new Error('no transfer found');
        transferId = latest[0].transfer_id;
      }
      const [rows] = await conn.query('SELECT part_no,total_parts,data FROM teleport_parts WHERE transfer_id=? ORDER BY part_no ASC', [transferId]);
      if (!rows.length) throw new Error('transfer_id not found');
      partCount = rows.length;
      const bufs = [];
      for (let i = 0; i < rows.length; i++) {
        if (rows[i].part_no !== i + 1) throw new Error('part order mismatch');
        bufs.push(rows[i].data);
        process.stderr.write(`[download] part ${i + 1}/${rows.length}\n`);
      }
      fs.writeFileSync(process.env.ARCHIVE_PATH, Buffer.concat(bufs));
    } else {
      const [rows] = await conn.query('SELECT data FROM teleport WHERE id=1');
      if (!rows.length || !rows[0].data) throw new Error('payload not found');
      fs.writeFileSync(process.env.ARCHIVE_PATH, rows[0].data);
    }
    process.stdout.write(`PARTS=${partCount}\nARCHIVE=${process.env.ARCHIVE_PATH}\n`);
  } finally { await conn.end(); }
})();
NODE
printf '%s\n' "$ARCHIVE" > .teleport_archive.tmp
```

## Step 5 — Verify archive
```bash
ARCHIVE="$(cat .teleport_archive.tmp)"
tar -tzf "$ARCHIVE" >/dev/null
echo "Archive OK: $ARCHIVE"
```

## Step 6 — Safety backup target path
```bash
set -euo pipefail
TARGET_PATH="$(cat .teleport_target.tmp)"
STAMP=$(date +%Y%m%d-%H%M%S)
BACKUP_PATH="${TARGET_PATH}_backup_${STAMP}"
if [ -d "$TARGET_PATH" ]; then
  cp -a "$TARGET_PATH" "$BACKUP_PATH"
else
  mkdir -p "$TARGET_PATH"
fi
echo "Safety backup: $BACKUP_PATH"
```

## Step 7 — Extract to target
```bash
TARGET_PATH="$(cat .teleport_target.tmp)"
ARCHIVE="$(cat .teleport_archive.tmp)"
tar -xzf "$ARCHIVE" -C "$TARGET_PATH"
echo "Restore completed: $TARGET_PATH"
```

## Step 8 — Post chat summary
Use the success/failure template from this file.

## Failure retry index (restore)
- Fail at **Step 3 (decode)**: fix restore code, rerun Step 2 then Step 3.
- Fail at **Step 4 (download)**: rerun Step 4 only.
- Fail at **Step 5 (verify)**: payload damaged; rerun Step 4 then Step 5.
- Fail at **Step 6 (backup target)**: check disk permissions/space, rerun Step 6.
- Fail at **Step 7 (extract)**: rerun Step 7 only after fixing target path/disk.
