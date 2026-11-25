# makefile-creator

[![License: MIT](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![No dependencies](https://img.shields.io/badge/dependencies-none-brightgreen)]()
[![Agent skill](https://img.shields.io/badge/agent-skill-8b5cf6)]()
[![OpenCode](https://img.shields.io/badge/OpenCode-supported-6b8fff)]()
[![Claude Code](https://img.shields.io/badge/Claude%20Code-skill-d97757)]()
[![Codex](https://img.shields.io/badge/Codex-supported-74aa9c)]()
[![Gemini CLI](https://img.shields.io/badge/Gemini%20CLI-supported-4285f4)]()
[![Cursor](https://img.shields.io/badge/Cursor-supported-000000)]()

Portable `SKILL.md` (agentskills.io); verified on OpenCode.

English | [Русский](README.RU.md)

An agent skill that writes and extends Makefiles for docker-compose projects the way you already work: one `make <env> <action>` interface for `dev` and `prod`, env files, deploy, and safety rails for secrets and dumps.

## Example

> **You:** add a target to dump the database

The agent detects the compose files (`docker-compose.yml`, `docker-compose.prod.yml`), the `mysql` service and the app CLI, and writes a `dump` target that does not leak credentials and cannot leave a truncated backup:

```make
dump: env-required
	@umask 077; mkdir -p "$(DUMP_DIR)"; \
	file="$(DUMP_DIR)/app-$(ENV)-$$(date +%Y%m%d-%H%M%S).sql.gz"; \
	set -o pipefail; \
	$(COMPOSE) exec -T mysql sh -c 'MYSQL_PWD="$$MYSQL_PASSWORD" exec mysqldump \
		-u"$$MYSQL_USER" --single-transaction --no-tablespaces \
		--default-character-set=utf8mb4 "$$MYSQL_DATABASE"' | gzip > "$$file.part" \
		|| { rm -f "$$file.part"; echo "dump failed, no file left behind"; exit 1; }; \
	mv "$$file.part" "$$file"; \
	echo "Dump: $$file"
```

Why each line is there - `umask 077` so the dump is not world-readable, write to `.part` and rename only on success, `set -o pipefail` so a broken `mysqldump` is not masked by a happy `gzip`, `MYSQL_PWD` inside the container instead of a visible `-p`.

## Install

OpenCode, as a symlink so edits to this repo show up immediately:

```bash
git clone git@github.com:Sect0R/makefile-creator.git ~/Projects/makefile-creator
ln -s ~/Projects/makefile-creator ~/.config/opencode/skills/makefile-creator
```

Or through the skills installer (copies into the agent's shared store):

```bash
npx -y skills add Sect0R/makefile-creator
```

## What it encodes

- `make <env> <action>` with `dev`/`prod` read from the goals, `help` as the default target, and a clear Russian error when the env is missing;
- standard targets: `up down restart ps logs [SERVICE= TAIL=] build migrate migrate-undo [CONFIRM=yes] dump cli ARGS= shell db update [FORCE=1]`, plus project commands through the app CLI;
- env files: `.env.example` carries names and comments, values are never printed, and editing a service env file means `make <env> restart`;
- safety: passwords via stdin, not argv; `MYSQL_PWD` instead of `-p`; dumps with `umask 077`, `.part` -> rename and `pipefail`; `CONFIRM=yes` for destructive prod actions; TTY-safe `exec`;
- `update` is prod-only: git pull from the deploy user, rebuild, migrate, verify.

## Structure

```
SKILL.md               principles, env files, checklist
reference/env-first.md the make <env> <action> pattern, complete
reference/actions.md   target table + guard examples
reference/safety.md    secrets, dumps, prod discipline
```

## Triggers

"сделай Makefile", "добавь цель", "допиши цели", standardizing a project's commands for dev/prod, any Makefile edit.

## License

MIT. See [LICENSE](LICENSE).
