# OpenClaw Memory Teleport Skill

Portable backup/restore skills for migrating OpenClaw workspace state across machines.

## Overview
This repository provides a practical, shell-based migration flow:
- backup on source machine
- upload archive payload to TiDB/MySQL
- restore in-place on destination machine

It is designed for reliability and operational clarity, with explicit preflight checks and safety backup before restore.

## Included Skills
- `skills/agent_teleport_backup/SKILL.md`  
  Source-side workflow for packing and uploading workspace data.

- `skills/agent_teleport_restore/SKILL.md`  
  Destination-side workflow for restoring workspace files in-place.

## Typical Use Case
Use this when moving OpenClaw to a new host and preserving:
- `MEMORY.md`
- `memory/`
- `skills/`
- workspace-level configuration and project files

## Quick Start
1. On source machine, run backup flow from:
   - `skills/agent_teleport_backup/SKILL.md`
2. Save the returned `RESTORE_DSN`.
3. On destination machine, run restore flow from:
   - `skills/agent_teleport_restore/SKILL.md`

Default restore target is:
- `/home/ubuntu/.openclaw/workspace`

Restore flow creates an automatic safety backup before extraction.

## Requirements
### Backup side
- `bash`, `tar`, `mysql`, `curl`, `xxd`, `sed`, `awk`

### Restore side
- `bash`, `mysql`, `tar`, `xxd`, `date`, `mktemp`

## Security Notes
- DSN should be treated as a secret restore key.
- Avoid sharing DSN in public channels.
- Remove temporary DB resources after restore when handling sensitive data.

## Maintainer
GitHub: [@lilyjazz](https://github.com/lilyjazz)
