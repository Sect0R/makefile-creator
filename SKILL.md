---
name: makefile-creator
description: Use when creating or editing a Makefile - standard targets, dev/prod environments, docker compose, env files, deploy, safe commands for secrets and dumps. Triggers - Makefile, make target, build automation, «сделай Makefile», «добавь цель», «допиши цели».
---

# Makefile creator

House conventions for Makefiles that drive docker-compose projects with dev/prod environments.

## Before writing

1. Read the project's existing Makefile if there is one. Keep its style; add targets instead of restructuring unless asked.
2. Detect the real setup, do not assume it:
   - compose files: `docker-compose.yml`, `docker-compose.prod.yml`;
   - env files: `.env`, `.env.example`, `<service>/.env`;
   - CLI entrypoint (for example `node src/cli/index.js`), migration tool, deploy user.
3. Only one environment per invocation. `make prod up` - yes, `make up` - error, not a guess.

## Core interface

`make <env> <action> [ПАРАМЕТРЫ]`, where env is `dev` or `prod`.

- `.DEFAULT_GOAL := help`; `make` with no args prints help.
- `make dev` (env without action) prints help too, never silence.
- Missing env fails with a clear Russian error via the `env-required` guard.
- Environments differ only by compose file: `COMPOSE_dev` / `COMPOSE_prod`.

Full pattern with guard and TTY-safe exec: [env-first.md](reference/env-first.md).

## Standard targets

Signatures and semantics: [actions.md](reference/actions.md). Short list:

```
up  down  restart  ps  logs [SERVICE= TAIL=]  build
migrate  migrate-undo [CONFIRM=yes]  dump [DUMP_DIR=]
cli ARGS="..."  shell  db  update [FORCE=1 DEPLOY_USER=...]
```

Project-specific commands (users, stats, key rotation) go through the app CLI, not ad-hoc SQL.

## Env files

- `.env`, `<service>/.env` are secrets: mention variable names only, never values.
- After editing a service env file the container must be recreated: `make <env> restart`.
- Ship `.env.example` with names and comments, not values.
- `up -d` alone does not reread a changed env file on a running stack.

## Safety rails - mandatory

- Passwords and tokens: via stdin, never as arguments (argv is visible in `ps` and shell history).
- DB password: `MYSQL_PWD` inside the container, never the `-p` flag.
- Dumps: `umask 077`, write to `.part` and rename after success, `set -o pipefail`.
- Destructive prod actions (`migrate-undo`) require `CONFIRM=yes` and a fresh `dump` first.
- `update` is prod-only; `down` never deletes data.
- `docker compose exec` must detect a TTY: `$$([ -t 0 ] || echo -T)`, or it breaks from cron/ssh scripts.

Details and copy-paste blocks: [safety.md](reference/safety.md).

## Style

- help output in Russian, one line per target with parameters.
- Makefile comments explain why, not what.
- Variables that callers may override use `?=`.
- Run the app CLI with `node` directly, not through the npm wrapper.

## Checklist for a new Makefile

- [ ] `.DEFAULT_GOAL := help`, `SHELL := /bin/bash`
- [ ] `env-required` guard with Russian errors
- [ ] `COMPOSE_dev` / `COMPOSE_prod`, TTY-safe `EXEC`
- [ ] `logs` with `SERVICE=` and `TAIL=`
- [ ] `migrate`, `migrate-undo` with prod confirm
- [ ] `dump` with the safe pattern
- [ ] `update` guarded to prod
- [ ] every target listed in `help` and in `.PHONY`
