---
name: agent-teleport-restore
description: Restore full OpenClaw core/workspace files in-place from TiDB/MySQL chunked teleport payload using one restore code (RESTORE).
---

# Agent Teleport Restore (Destination OpenClaw)

## Execution model (important)
Run **one step at a time**. Do not send one giant command block.
After each step, post progress in chat.
For long-running download/verify phases, post heartbeat updates every 20-30s.

Heartbeat format:
`[WORKING] <stage> — <progress> — elapsed <seconds>s`

Examples:
- `[WORKING] Downloading parts — 12/40 (30%) — elapsed 76s`
- `[WORKING] Verifying archive — SHA256 check running — elapsed 19s`

## Chat Output Contract (MANDATORY)
Success:
```text
Restore completed.
- Target path: <path>
- Safety backup: <backup_path>
- Payload parts: <n>
- Expires At: <ISO timestamp or unknown>
- Claim Action: <claim now if near expiry>
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
[[ "$RESTORE_CODE" == RESTORE-* ]] || { echo "ERROR: restore code must start with RESTORE-"; exit 1; }
printf '%s\n' "$RESTORE_CODE" > .teleport_code.tmp
echo "Code loaded"
```

## Step 3 — Decode restore code to DSN + transfer id
```bash
set -euo pipefail
RESTORE_CODE="$(cat .teleport_code.tmp)"
TRANSFER_ID=""
PAYLOAD="${RESTORE_CODE#RESTORE-}"
B64=$(printf '%s' "$PAYLOAD" | tr '-_' '+/')
PAD=$(( (4 - ${#B64} % 4) % 4 ))
[ "$PAD" -eq 2 ] && B64="${B64}=="
[ "$PAD" -eq 1 ] && B64="${B64}="
DEC=$(printf '%s' "$B64" | base64 -d 2>/dev/null || true)
DSN="${DEC%%|*}"
[ "$DEC" != "$DSN" ] && TRANSFER_ID="${DEC#*|}"
case "$DSN" in mysql://*) ;; *) echo "ERROR: invalid RESTORE code payload"; exit 1;; esac
printf '%s\n' "$DSN" > .teleport_dsn.tmp
printf '%s\n' "$TRANSFER_ID" > .teleport_transfer.tmp
echo "Decode OK"
```

## Step 4 — Download payload parts and assemble archive
```bash
set -euo pipefail
umask 077
TMPDIR=$(mktemp -d)
ARCHIVE="$TMPDIR/workspace.tar.gz"
DSN="$(cat .teleport_dsn.tmp)"
TRANSFER_ID="$(cat .teleport_transfer.tmp)"
[ -n "$TRANSFER_ID" ] || { echo "ERROR: restore code missing transfer_id"; exit 1; }

DOWN_OUT=$(DSN="$DSN" TRANSFER_ID="$TRANSFER_ID" ARCHIVE_PATH="$ARCHIVE" npx -y -p mysql2 node - <<'NODE'
const fs = require('fs');
const mysql = require('mysql2/promise');
(async () => {
  const u = new URL(process.env.DSN);
  const db = (u.pathname || '/test').replace(/^\//,'') || 'test';
  const conn = await mysql.createConnection({ host: u.hostname, port: Number(u.port || 4000), user: decodeURIComponent(u.username), password: decodeURIComponent(u.password), database: db, ssl: { minVersion: 'TLSv1.2' } });
  let partCount = 1;
  let expectedSha256 = '';
  let expiresAt = '';
  let claimUrl = '';
  try {
    const transferId = process.env.TRANSFER_ID || '';
    const [tbl] = await conn.query("SHOW TABLES LIKE 'teleport_parts'");
    if (tbl.length) {
      const [rows] = await conn.query('SELECT part_no,total_parts,data FROM teleport_parts WHERE transfer_id=? ORDER BY part_no ASC', [transferId]);
      if (!rows.length) throw new Error('transfer_id not found');
      const expectedParts = Number(rows[0].total_parts || 0);
      if (!expectedParts || expectedParts !== rows.length) throw new Error('parts count mismatch');
      partCount = rows.length;
      const fd = fs.openSync(process.env.ARCHIVE_PATH, 'w');
      try {
        for (let i = 0; i < rows.length; i++) {
          if (rows[i].part_no !== i + 1) throw new Error('part order mismatch');
          fs.writeSync(fd, rows[i].data);
          const done = i + 1;
          const pct = Math.floor((done / rows.length) * 100);
          process.stderr.write(`[download] part ${done}/${rows.length} (${pct}%)\n`);
        }
      } finally {
        fs.closeSync(fd);
      }

      const [metaTbl] = await conn.query("SHOW TABLES LIKE 'teleport_meta'");
      if (metaTbl.length) {
        const [meta] = await conn.query('SELECT sha256, parts, expires_at, claim_url FROM teleport_meta WHERE transfer_id=? LIMIT 1', [transferId]);
        if (meta.length) {
          expectedSha256 = meta[0].sha256 || '';
          if (meta[0].parts && Number(meta[0].parts) !== partCount) throw new Error('meta parts mismatch');
          expiresAt = meta[0].expires_at || '';
          claimUrl = meta[0].claim_url || '';
        }
      }
    } else {
      const [rows] = await conn.query('SELECT data FROM teleport WHERE id=1');
      if (!rows.length || !rows[0].data) throw new Error('payload not found');
      fs.writeFileSync(process.env.ARCHIVE_PATH, rows[0].data);
    }
    process.stdout.write(`PARTS=${partCount}\nARCHIVE=${process.env.ARCHIVE_PATH}\nEXPECTED_SHA256=${expectedSha256}\nEXPIRES_AT=${expiresAt}\nCLAIM_URL=${claimUrl}\n`);
  } finally { await conn.end(); }
})();
NODE
)
printf '%s\n' "$DOWN_OUT" > .teleport_download.tmp
printf '%s\n' "$ARCHIVE" > .teleport_archive.tmp
printf '%s\n' "$DOWN_OUT" | sed -n 's/^EXPECTED_SHA256=//p' > .teleport_expected_sha256.tmp
printf '%s\n' "$DOWN_OUT" | sed -n 's/^EXPIRES_AT=//p' > .teleport_expires_at.tmp
printf '%s\n' "$DOWN_OUT" | sed -n 's/^CLAIM_URL=//p' > .teleport_claim_url.tmp
echo "$DOWN_OUT"
```

## Step 5 — Verify archive
```bash
set -euo pipefail
ARCHIVE="$(cat .teleport_archive.tmp)"
tar -tzf "$ARCHIVE" >/dev/null
EXPECTED_SHA256="$(cat .teleport_expected_sha256.tmp 2>/dev/null || true)"
ACTUAL_SHA256=$(ARCHIVE="$ARCHIVE" node - <<'NODE'
const fs = require('fs');
const crypto = require('crypto');
const p = process.env.ARCHIVE;
const buf = fs.readFileSync(p);
process.stdout.write(crypto.createHash('sha256').update(buf).digest('hex'));
NODE
)
if [ -n "$EXPECTED_SHA256" ] && [ "$EXPECTED_SHA256" != "$ACTUAL_SHA256" ]; then
  echo "ERROR: sha256 mismatch"; exit 1
fi
if [ -n "$EXPECTED_SHA256" ]; then
  echo "Archive OK: $ARCHIVE (sha256 verified)"
else
  echo "Archive OK: $ARCHIVE (sha256 unavailable, tar structure verified)"
fi
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
If `EXPIRES_AT` exists (from `.teleport_expires_at.tmp`) and is near expiry, remind user to claim promptly using `.teleport_claim_url.tmp`.
In success summary, always include `Expires At` and `Claim Action` fields.

## Step 9 — Cleanup sensitive temp files (recommended)
```bash
rm -f .teleport_dsn.tmp .teleport_transfer.tmp .teleport_archive.tmp .teleport_download.tmp .teleport_expected_sha256.tmp .teleport_expires_at.tmp .teleport_claim_url.tmp
echo "Sensitive temp files cleaned"
```

## Failure retry index (restore)
- Fail at **Step 3 (decode)**: fix restore code, rerun Step 2 then Step 3.
- Fail at **Step 4 (download)**: rerun Step 4 only. If `transfer_id` missing, regenerate restore code from backup side.
- Fail at **Step 5 (verify)**: payload damaged; rerun Step 4 then Step 5.
- Fail at **Step 6 (backup target)**: check disk permissions/space, rerun Step 6.
- Fail at **Step 7 (extract)**: rerun Step 7 only after fixing target path/disk.
