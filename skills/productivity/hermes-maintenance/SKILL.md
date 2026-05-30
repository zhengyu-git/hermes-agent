---
name: hermes-maintenance
description: Routine maintenance tasks for the Hermes environment, including cache cleanup, skill directory inspection, memory sanitization, and logging of actions.
---
# Overview
This skill defines a standard procedure to keep the Hermes agent tidy and performant. It is intended to be run regularly (e.g., via a cron job) and can be invoked manually.

## Steps
1. **Cache Cleanup**
   - Remove files older than 7 days from `~/.hermes/image_cache/`.
   - Remove files older than 7 days from `~/.hermes/cache/screenshots/`.
2. **Skill Directory Inspection**
   - List all skill categories under `~/.hermes/skills/`.
   - Verify that each skill's `SKILL.md` exists and is parsable.
   - Optionally, run any skill-specific health checks (if defined).
3. **Memory Sanitization**
   - Scan `~/.hermes/memories/` for outdated or invalid entries (e.g., expired tokens, stale configuration notes).
   - Update or remove such entries to keep memories relevant.
4. **Logging**
   - Summarize actions taken: number of files deleted, any memory updates performed, and any anomalies detected.
   - Output the summary to stdout or write to a log file `~/.hermes/maintenance.log`.

## Pitfalls & Tips
- **Never delete files indiscriminately**: Always filter by modification time (`find … -mtime +7`).
- **Token handling**: When removing expired tokens from memory, ensure you do not delete valid credentials.
- **Skill health checks**: If a skill lacks a `SKILL.md`, consider creating a minimal placeholder to avoid future errors.
- **Idempotence**: The script should be safe to run multiple times; missing directories are acceptable and should not cause failure.

## References
- `references/maintenance_log.md` – Example log output from the latest maintenance run.
