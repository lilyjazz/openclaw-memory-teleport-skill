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

### Recommended (encrypted restore code)
Set the same key on both sides:

```bash
export TELEPORT_KEY='your-strong-secret'
```

Then use URL prompts:

```text
# A: backup
https://github.com/lilyjazz/openclaw-memory-teleport-skill/blob/main/skills/agent_teleport_backup/SKILL.md
Use this skill to back up all of OpenClaw’s memory.
```

```text
# B: restore
https://github.com/lilyjazz/openclaw-memory-teleport-skill/blob/main/skills/agent_teleport_restore/SKILL.md
Use this skill to restore OpenClaw’s memory.
🔐 Restore Code: OMT1:...
```

### Legacy (plain DSN)
If `TELEPORT_KEY` is not set, backup returns plain DSN and restore still works with:

```text
🔐 Restore Code: mysql://USER:PASSWORD@HOST:4000/test
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
