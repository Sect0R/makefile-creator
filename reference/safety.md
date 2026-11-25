# Safety rails

Mandatory for every target that touches secrets, the database, or prod.

## Passwords and tokens

Arguments are visible in `ps` output and shell history. Passwords travel via
stdin only:

```make
set-password: env-required
	@if [ -z "$(USERNAME)" ]; then echo "Нужно: make $(ENV) set-password USERNAME=ivanov"; exit 1; fi
	@read -rsp "Новый пароль для $(USERNAME): " p1; echo; \
	read -rsp "Ещё раз: " p2; echo; \
	[ "$$p1" = "$$p2" ] || { echo "Пароли не совпали"; exit 1; }; \
	printf '%s' "$$p1" | $(CLI_STDIN) set-password --username "$(USERNAME)"
```

`CLI_STDIN` is the `-T` (no TTY) variant of exec: `printf` pipes into it
without a terminal.

## Database console

The DB password goes through `MYSQL_PWD` inside the container, never the `-p`
flag:

```make
db: env-required
	$(EXEC) mysql sh -c 'MYSQL_PWD="$$MYSQL_PASSWORD" exec mysql -u"$$MYSQL_USER" "$$MYSQL_DATABASE"'
```

## Dump

Why every line exists:

- `umask 077` in the same statement as `mkdir`: otherwise the dump lands world-readable (644). It holds password hashes and session ids even when payloads are encrypted.
- write to `file.part`, `mv` after success: an interrupted dump must never look like a finished backup.
- `set -o pipefail`: without it the exit code comes from `gzip`, which happily compresses an empty stream.
- `--single-transaction`: consistent snapshot without table locks, prod keeps writing.
- `--no-tablespaces`: mysqldump otherwise needs the PROCESS privilege the app user must not have.

```make
dump: env-required
	@umask 077; mkdir -p "$(DUMP_DIR)"; \
	file="$(DUMP_DIR)/<project>-$(ENV)-$$(date +%Y%m%d-%H%M%S).sql.gz"; \
	set -o pipefail; \
	$(COMPOSE) exec -T mysql sh -c 'MYSQL_PWD="$$MYSQL_PASSWORD" exec mysqldump \
		-u"$$MYSQL_USER" --single-transaction --no-tablespaces \
		--default-character-set=utf8mb4 "$$MYSQL_DATABASE"' | gzip > "$$file.part" \
		|| { rm -f "$$file.part"; echo "Дамп не снят, файл не создан"; exit 1; }; \
	mv "$$file.part" "$$file"; \
	echo "Дамп: $$file ($$(du -h "$$file" | cut -f1))"
```

Say in the final echo when the dump contains wrapped keys or hashes: it must be
stored apart from the master key.

## Env files

- Never print values from `.env`, `<service>/.env` - variable names only.
- `.env.example` carries names and comments; real `.env` never enters git.
- A running container does not reread its env file: `make <env> restart`.

## Prod discipline

- `down` stops, never deletes data.
- `migrate-undo` on prod: `CONFIRM=yes` plus a fresh `dump` (see actions.md).
- `update` is prod-only; in dev the answer is `git pull && make dev up`.
- Read-only targets (`get-users`, `get-stats`, `ps`, `logs`) are always safe:
  say so in help so nobody hesitates on prod.
