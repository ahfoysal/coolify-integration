# coolify-integration

Optional, standalone [Claude Code](https://docs.anthropic.com/claude/docs/claude-code)
plugin that lets [atelier](https://github.com/AkaLab-Tech/atelier) agents operate
a [Coolify](https://coolify.io) VPS: validate deployments, read logs, trigger
deploys, set environment variables, and create or delete applications — all via
Coolify's v1 REST API.

It is independent of atelier; install it only if you deploy with Coolify.

## Install

```
/plugin marketplace add AkaLab-Tech/claude-plugins
/plugin install coolify-integration@akalab-tech
```

## Setup

In Coolify, create an API token (Settings → API / Keys & Tokens) with the
`read`, `deploy`, and `write` scopes (avoid `root` unless you need it).

Setup has two parts: a **machine-wide** part (link the CLI onto `PATH`, grant
the [allowlist](#how-permissions-work)) and a **per-project** part (the token +
URL). Auth is per project so one operator can deploy different projects to
different Coolify instances.

**With atelier** — pick it during `install.sh`, or run any time:

```
/atelier:setup-coolify
```

**Standalone** — `configure` does the machine-wide part, then (run from inside a
project) wires that project's `.env`:

```sh
atelier-coolify configure
# or, inside a Claude Code session:
/coolify-integration:setup
```

### Per-project auth

Each project carries its own credentials in its `.env` (kept gitignored by
atelier's `.env*` guardrail):

```sh
COOLIFY_BASE_URL=https://coolify.example.com
COOLIFY_API_TOKEN=<token>
```

`atelier-coolify` reads these from the project's `.env` (or `$COOLIFY_ENV_FILE`,
or the environment) at call time.

## Usage

Run `atelier-coolify --help`, or let the `coolify` skill drive it. Common flow:

```sh
atelier-coolify list                         # find the app UUID
atelier-coolify deploy-mode <uuid>           # cached/known deploy mode
atelier-coolify validate <uuid>              # status + logs if unhealthy
atelier-coolify set-env <uuid> API_URL=https://api.example.com
atelier-coolify deploy <uuid>                # apply the change
```

`deploy-mode` records the app's git context, but Coolify's current application
API does not expose the auto-deploy flag. Live reads therefore return
`deploy_mode: "unknown"`, which the `coolify` skill should treat as manual:
run an explicit `deploy` for code changes unless an operator has confirmed the
app auto-deploys. That operator-confirmed mode can be cached per project in
`.coolify-deploy-mode.json`; add that file to the project's `.gitignore`, and
use `--refresh` after changing the app's git settings in Coolify.

## How permissions work

This plugin stays fully decoupled from atelier — it never edits atelier's
shipped `settings.template.json`. Instead, `/coolify-integration:setup` (or
`atelier-coolify enable-permissions`) merges the routine allowlist into your
**user-level** `settings.json` (`$ATELIER_CONFIG_DIR/settings.json`), which
persists across tasks and applies on top of atelier's per-task settings:

```json
"Bash(atelier-coolify list:*)",
"Bash(atelier-coolify status:*)",
"Bash(atelier-coolify logs:*)",
"Bash(atelier-coolify deployments:*)",
"Bash(atelier-coolify validate:*)",
"Bash(atelier-coolify deploy-mode:*)",
"Bash(atelier-coolify deploy:*)",
"Bash(atelier-coolify set-env:*)",
"Bash(atelier-coolify health:*)",
"Bash(atelier-coolify version:*)",
"Skill(coolify-integration:*)"
```

The merge is idempotent and order-preserving, and touches nothing else in the
file. The gated commands (`create-app-public`, `delete-app`) are deliberately
omitted so they fall back to operator confirmation. As defense-in-depth they
also confirm in-CLI: without `--yes`, `create-app-public` asks y/N and
`delete-app` requires typing the app uuid back before anything is sent to the
API; non-interactive runs (no TTY) refuse to proceed unless `--yes` is passed.
Undo the allowlist with `atelier-coolify disable-permissions`.

## API compatibility

Targets Coolify v4's `/api/v1` endpoints. If your instance differs, adjust the
paths in [`scripts/atelier-coolify`](scripts/atelier-coolify).

## License

MIT — see [LICENSE](LICENSE).
