# OpenClaw Memory Teleport Skill

Portable backup/restore skills for migrating OpenClaw workspace state across machines.

## Overview
This repo provides a shell-based migration flow:
1. Backup on source machine
2. Upload payload to TiDB/MySQL (`teleport_parts` table)
3. Restore in-place on destination machine

It restores real workspace files (not just DB row export), including memory/config/skills.

## Included Skills
- `skills/agent_teleport_backup/SKILL.md` — source-side backup/upload
- `skills/agent_teleport_restore/SKILL.md` — destination-side in-place restore

## Copy/Paste Quick Start

> Run on **OpenClaw A** (source), then run on **OpenClaw B** (destination).

### Recommended (extreme simplicity)
No extra setup, no passphrase input, just URL prompts.
Backup returns one restore code (`RESTORE-...`) and restore accepts it directly.
Before backup starts, it shows folder-size tree and asks user to select scope (`full` / `core` / `custom`).
If archive is larger than 10MB, backup auto-splits into multiple DB parts; restore auto-downloads all parts and reassembles.

```text
# A: backup
https://github.com/lilyjazz/openclaw-memory-teleport-skill/blob/main/skills/agent_teleport_backup/SKILL.md
Use this skill to back up all of OpenClaw’s memory.
After backup, output the restore handoff with the real code for OpenClaw B.
```

```text
# B: restore
https://github.com/lilyjazz/openclaw-memory-teleport-skill/blob/main/skills/agent_teleport_restore/SKILL.md
Use this skill to restore OpenClaw’s memory.
🔐 Restore Code: RESTORE-...
```

## Verified Workflow (A -> B)
Validated with real run:
- Source packed and uploaded `workspace.tar.gz`
- Destination restored to `/home/ubuntu/.openclaw/workspace`
- Safety backup created before overwrite
- Core files successfully restored in-place

## Requirements
### Backup side
- `bash`, `tar`, `curl`, `sed`, `awk`, `base64`
- `node`, `npx` (uses `mysql2` package automatically)

### Restore side
- `bash`, `tar`, `date`, `mktemp`, `base64`
- `node`, `npx` (uses `mysql2` package automatically)

## Troubleshooting
- **DSN copied with spaces/newlines**: keep `DSN_RAW` and sanitize via `tr -d '[:space:]'` (already in example).
- **`npx` install blocked**: ensure outbound npm access or preinstall `mysql2` in your environment.
- **`payload not found`**: DSN wrong, DB wrong, or `teleport.id=1` missing.
- **archive too large**: clean caches/build outputs and retry backup.

## Security Notes
- DSN is a restore key (secret).
- Never share DSN in public channels.
- Delete/rotate temporary DB resources after restore for sensitive data.

## Maintainer
GitHub: [@lilyjazz](https://github.com/lilyjazz)
