# GTM — Agent Teleport Skill (Open Source)

## 1) Positioning
**Agent Teleport** helps OpenClaw users migrate/backup agent memory + workspace between machines in minutes using DSN-based restore.

**One-liner:**
> Move your OpenClaw brain to a new machine with one restore code.

## 2) ICP (Ideal Customer Profile)
- Solo builders running OpenClaw on laptop + cloud VM
- AI operators with multiple servers/regions
- Devs frequently rebuilding environments
- Teams wanting repeatable assistant state migration

## 3) Core Value Props
- Shell-only workflow, no Python script dependency
- Fast backup/restore for real workspace files
- In-place restore with safety backup
- Cross-machine, cross-instance portability

## 4) Packaging & Offers
- Free OSS repo with 2 skills:
  - `agent_teleport_backup`
  - `agent_teleport_restore`
- Optional future paid offer:
  - managed encrypted relay
  - bigger payload handling
  - org audit logs

## 5) Launch Channels
1. GitHub README + demo GIF
2. OpenClaw Discord showcase post
3. X/Twitter thread with migration pain-point framing
4. ClawHub listing (if published there)
5. Short YouTube walkthrough (2–3 min)

## 6) Content Plan (first 2 weeks)
- Day 1: Launch post + repo + quickstart
- Day 2: “from old VM to new VM in 60s” demo clip
- Day 4: troubleshooting post (missing mysql/curl, DSN mistakes)
- Day 7: case study (real migration story)
- Day 10: security deep-dive (DSN handling & backup strategy)
- Day 14: roadmap + collect feature requests

## 7) Activation Funnel
1. Visit repo
2. Copy backup skill command
3. Generate DSN
4. Run restore skill on target machine
5. Verify core files restored

North-star activation metric:
- **First successful restore within 15 minutes**

## 8) Success Metrics
- GitHub stars/watchers
- Unique cloners/week
- Successful restores (self-reported)
- Time-to-first-restore
- Issue-to-fix turnaround time

## 9) Objections & Responses
- “Is DSN safe?” → DSN is restore key; keep private; rotate/delete temporary DB after restore
- “What if restore breaks?” → automatic safety backup before in-place extraction
- “Do I need Python?” → no `.py` requirement for the published skills

## 10) Next Product Iterations
- Add optional core-only backup mode (memory/config/skills)
- Add large-file diagnostics before pack
- Add checksum verification output
- Add auto-cleanup script for temporary DB

## 11) Launch Assets Checklist
- [ ] README quickstart polished
- [ ] Demo terminal recording
- [ ] Security note section prominent
- [ ] FAQ section (common failure cases)
- [ ] One-liner copy for social posts

## 12) Suggested Announcement Copy
**Short post:**
> Just open-sourced Agent Teleport Skill for OpenClaw 🚀
> Backup on machine A, restore on machine B with one DSN code.
> Shell-only, no Python script requirement.
> Repo: https://github.com/lilyjazz/openclaw-memory-teleport-skill

**Long post:**
> Migrating OpenClaw agents used to be messy (missing paths, script drift, partial restores).
> I packaged Agent Teleport into two clean skills:
> - backup (source)
> - restore (destination)
> It restores real workspace files in-place and creates a safety backup first.
> Public repo: https://github.com/lilyjazz/openclaw-memory-teleport-skill
