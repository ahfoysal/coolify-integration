# In Progress

Active tasks for the current development cycle.

Workflow: `ROADMAP.md` → start a task → move here → finish → move to `HISTORY.md`.

When a PR closes a task, the **same PR** must update both `IN_PROGRESS.md` (remove) and `HISTORY.md` (add). Do not defer either to a follow-up commit on the protected branch after merge.

- [ ] `bug` deploy-mode always resolves to 'unknown' — dead field probe against Coolify's application endpoints `#8` `~1h`
  - Repro: run `atelier-coolify deploy-mode <uuid>` against any real app; `deploy_mode` is always `"unknown"`.
  - Root cause: `GET /applications/{uuid}` never eager-loads the `settings` relation upstream, so `is_auto_deploy_enabled` (and the two other probed field names) never appear in the response — the three-way jq probe in `cmd_deploy_mode` is structurally guaranteed to fail. See #8 for the full trace through Coolify's source.
  - Acceptance: remove the dead probe; `deploy-mode` no longer pretends it can resolve `auto` vs `manual` from these fields. Docs (`SKILL.md`, `README.md`, `CLAUDE.md`) updated to state the flag can't currently be read from the API and the skill always treats apps as manual unless the operator overrides. Existing `.coolify-deploy-mode.json` cache format/behavior degrades gracefully (no crash on old cache entries).
