# Standard targets

Every target depends on `env-required`. All are `.PHONY`. Parameters use `?=`
defaults where it makes sense.

| Target | Signature | Semantics |
|---|---|---|
| `up` | `make <env> up` | `$(COMPOSE) up -d --build` - raise the stack, rebuild changed images |
| `down` | `make <env> down` | stop the stack. Data and volumes survive |
| `restart` | `make <env> restart` | `$(COMPOSE) up -d --force-recreate server` - reread a changed service env file right now |
| `ps` | `make <env> ps` | container state, `-a` included |
| `logs` | `make <env> logs [SERVICE=server] [TAIL=200]` | `logs -f --tail=$(TAIL) $(SERVICE)`; default all services |
| `build` | `make <env> build` | rebuild images |
| `migrate` | `make <env> migrate` | `$(COMPOSE) run --rm migrate` (or the project migration entrypoint) |
| `migrate-undo` | `make <env> migrate-undo [CONFIRM=yes]` | undo the last migration. On prod requires `CONFIRM=yes` and warns to dump first |
| `dump` | `make <env> dump [DUMP_DIR=backups]` | gzipped DB dump, see safety.md |
| `cli` | `make <env> cli ARGS="rotate-keys"` | any app console command through the CLI entrypoint |
| `shell` | `make <env> shell` | shell inside the `server` container |
| `db` | `make <env> db` | MySQL console as the application user |
| `update` | `make <env> update [FORCE=1] [DEPLOY_USER=deploy]` | **prod only**: git pull from the deploy user, rebuild changed images, restart, migrate, verify |

## Guard examples

Target with required parameters fails with a usage line, not with a stack trace:

```make
create-user: env-required
	@if [ -z "$(USERNAME)" ] || [ -z "$(ROLE)" ] || [ -z "$(FULLNAME)" ]; then \
		echo 'Нужно: make $(ENV) create-user USERNAME=ivanov ROLE=user FULLNAME="John doe" [ASK_PASSWORD=1]'; \
		exit 1; \
	fi
```

Action restricted to prod:

```make
update: env-required
	@if [ "$(ENV)" != prod ]; then \
		echo "update - только для prod. В разработке: git pull && make dev up"; \
		exit 1; \
	fi
	@DEPLOY_USER="$(DEPLOY_USER)" FORCE="$(FORCE)" bash deploy/update.sh
```

Destructive action with confirmation:

```make
migrate-undo: env-required
	@if [ "$(ENV)" = prod ] && [ "$(CONFIRM)" != yes ]; then \
		echo "Откат миграции на проде может удалить колонку вместе с данными."; \
		echo "Сначала make prod dump, потом: make prod migrate-undo CONFIRM=yes"; \
		exit 1; \
	fi
	$(COMPOSE) run --rm migrate npx sequelize-cli db:migrate:undo
```

## Project-specific commands

User management, stats, key rotation and friends go through the app CLI:

```make
get-users: env-required
	$(CLI) get-users

get-stats: env-required
	$(CLI) get-stats

cli: env-required
	@if [ -z "$(ARGS)" ]; then echo 'Нужно: make $(ENV) cli ARGS="rotate-keys"'; exit 1; fi
	$(CLI) $(ARGS)
```

Read-only commands are safe to run on prod; note that in the help line.
