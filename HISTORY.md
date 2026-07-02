# History

Completed work log. Newest first. Each entry references the PR(s) that delivered the work.

## 2026-07

### Security hardening: token off argv, in-CLI guards for destructive commands, query-string encoding — 2026-07-01
**PR:** [#5](https://github.com/AkaLab-Tech/coolify-integration/pull/5) — branch `fix/token-and-confirm-guards`

Hardening pass on `scripts/atelier-coolify` from a security audit: the API
token was visible in `ps` for the lifetime of every request, the destructive
commands had no in-CLI confirmation (the allowlist omission was the only
guard), and UUIDs were interpolated into query strings unencoded.

**Delivered:**
- `api()` now passes the `Authorization: Bearer` header to curl via a
  process-substitution fd (`-H @<(printf …)`, curl >= 7.55) instead of argv,
  so the token no longer appears in the process list (verified on macOS
  bash 3.2 under `set -euo pipefail`).
- `delete-app` and `create-app-public` require `--yes` or an interactive
  confirmation before any network call: `delete-app` makes the operator type
  the app uuid back; `create-app-public` asks y/N. Non-TTY without `--yes`
  dies with a message explaining the flag. `usage()`, `skills/coolify/SKILL.md`
  and `README.md` updated.
- New `_uri()` helper (`jq -rn --arg v … '$v|@uri'`) percent-encodes values
  interpolated into query strings (`deploy?uuid=…`, `logs?lines=…`).

**Tests:** `bash -n` and `shellcheck -S warning` clean; `--help` exits 0;
fake-project run against `https://127.0.0.1:1` fails at the connection stage
(header mechanism builds a valid curl call); `set -x` trace of a request shows
no token in curl's argv; non-TTY `delete-app` without `--yes` dies before any
network I/O.
