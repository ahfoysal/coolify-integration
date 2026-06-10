---
name: coolify
description: >-
  Validate, deploy, and manage applications on a Coolify VPS via the
  atelier-coolify CLI. Use when the operator asks to deploy a project, check a
  deployment's status, read deploy logs, diagnose or fix a failed deployment,
  set environment variables, or create/delete an app on the VPS.
---

# Coolify

`atelier-coolify` is a thin client for the Coolify v1 API. It reads
`COOLIFY_API_TOKEN` and `COOLIFY_BASE_URL` from the current project's `.env`
(kept gitignored by atelier's `.env*` guardrail) or from the environment, so one
operator can deploy different projects to different instances. Run
`atelier-coolify --help` for the full reference.

## Commands

| Command | Purpose |
| --- | --- |
| `atelier-coolify list` | List applications and their UUIDs |
| `atelier-coolify status <uuid>` | Application status |
| `atelier-coolify logs <uuid> [lines]` | Recent logs |
| `atelier-coolify deployments` | Running deployments |
| `atelier-coolify validate <uuid>` | Status + logs if unhealthy (exit 2 on failure) |
| `atelier-coolify deploy-mode <uuid>` | Report deploy mode: `auto` (push deploys) vs `manual`; cached per project |
| `atelier-coolify deploy <uuid> [--force]` | Trigger a deployment |
| `atelier-coolify set-env <uuid> KEY=VALUE` | Upsert an env var |
| `atelier-coolify create-app-public …` | Create an app (operator confirmation) |
| `atelier-coolify delete-app <uuid>` | Delete an app (operator confirmation) |

## Know how the app deploys before triggering anything

Many apps are wired through Coolify's GitHub App so that a push to the watched
branch **auto-deploys**. For those, calling `deploy` manually is redundant and
can double-trigger. Resolve the deploy mode once with `deploy-mode <uuid>`:

- `deploy_mode: "auto"` — a `git push`/merge to `watched_branch` **is** the
  deploy. Do not run `deploy` for code changes; just push and then `validate`.
- `deploy_mode: "manual"` — Coolify will not deploy on push; run `deploy <uuid>`
  yourself after the code lands.
- `deploy_mode: "unknown"` — the API did not expose the flag. Treat as manual
  and tell the operator the mode could not be confirmed.

`deploy-mode` caches the result per project (`.coolify-deploy-mode.json`), so it
is resolved once, not re-assumed every run. Pass `--refresh` after you change
the app's git/auto-deploy wiring in Coolify.

## Validate-and-fix flow

1. `list` → find the app UUID. `deploy-mode <uuid>` → learn auto vs manual.
2. `validate <uuid>` — if it exits non-zero, read the logs it prints.
3. Diagnose from the logs (missing env var, build failure, wrong branch).
4. Apply the fix:
   - **Code fix:** if `auto`, push/merge to the watched branch — that deploys;
     do **not** call `deploy`. If `manual`, land the code then `deploy <uuid>`.
   - **Config fix (`set-env`):** always `deploy <uuid>` afterwards, even on
     `auto` apps — `set-env` writes through the API, not git, so auto-deploy
     does not fire for it.
5. `validate <uuid>` again to confirm the deployment is healthy.

## Rules

- Resolve the target app with `list` first; never guess a UUID.
- Check `deploy-mode` before triggering a deploy on a code change; on `auto`
  apps a push already deploys — do not call `deploy`.
- `create-app-public` and `delete-app` are not in the allowlist — confirm with
  the operator before running them.
- After `set-env`, a `deploy` is required for the change to take effect (this
  holds even on auto-deploy apps).
