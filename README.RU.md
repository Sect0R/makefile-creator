# makefile-creator

[![License: MIT](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![No dependencies](https://img.shields.io/badge/dependencies-none-brightgreen)]()
[![Agent skill](https://img.shields.io/badge/agent-skill-8b5cf6)]()
[![OpenCode](https://img.shields.io/badge/OpenCode-supported-6b8fff)]()
[![Claude Code](https://img.shields.io/badge/Claude%20Code-skill-d97757)]()
[![Codex](https://img.shields.io/badge/Codex-supported-74aa9c)]()
[![Gemini CLI](https://img.shields.io/badge/Gemini%20CLI-supported-4285f4)]()
[![Cursor](https://img.shields.io/badge/Cursor-supported-000000)]()

Портативный `SKILL.md` (agentskills.io); проверен на OpenCode.

[English](README.md) | Русский

Скил для агентных CLI, который пишет и дополняет Makefile для docker-compose проектов так, как ты уже работаешь: единый интерфейс `make <среда> <действие>` для `dev` и `prod`, env-файлы, деплой и правила безопасности для секретов и дампов.

## Пример

> **Ты:** добавь цель для дампа базы

Агент находит compose-файлы (`docker-compose.yml`, `docker-compose.prod.yml`), сервис `mysql` и CLI приложения и пишет цель `dump`, которая не светит креды и не может оставить обрезанный бэкап:

```make
dump: env-required
	@umask 077; mkdir -p "$(DUMP_DIR)"; \
	file="$(DUMP_DIR)/app-$(ENV)-$$(date +%Y%m%d-%H%M%S).sql.gz"; \
	set -o pipefail; \
	$(COMPOSE) exec -T mysql sh -c 'MYSQL_PWD="$$MYSQL_PASSWORD" exec mysqldump \
		-u"$$MYSQL_USER" --single-transaction --no-tablespaces \
		--default-character-set=utf8mb4 "$$MYSQL_DATABASE"' | gzip > "$$file.part" \
		|| { rm -f "$$file.part"; echo "дамп не снят, файл не создан"; exit 1; }; \
	mv "$$file.part" "$$file"; \
	echo "Дамп: $$file"
```

Почему каждая строка такая: `umask 077` - дамп не читается всеми подряд; запись в `.part` и rename только после успеха; `set -o pipefail` - чтобы сломанный `mysqldump` не спрятался за довольным `gzip`; `MYSQL_PWD` внутри контейнера вместо видимого `-p`.

## Установка

OpenCode, симлинком, чтобы правки репозитория были видны сразу:

```bash
git clone git@github.com:Sect0R/makefile-creator.git ~/Projects/makefile-creator
ln -s ~/Projects/makefile-creator ~/.config/opencode/skills/makefile-creator
```

Или через установщик скиллов (копия в общее хранилище агента):

```bash
npx -y skills add Sect0R/makefile-creator
```

## Что зашито

- `make <среда> <действие>`: среда читается из целей, `help` - дефолтная цель, без среды - понятная ошибка по-русски;
- типовые цели: `up down restart ps logs [SERVICE= TAIL=] build migrate migrate-undo [CONFIRM=yes] dump cli ARGS= shell db update [FORCE=1]` плюс проектные команды через CLI приложения;
- env-файлы: `.env.example` с именами и комментариями, значения не печатаются, правка сервисного env -> `make <env> restart`;
- безопасность: пароли через stdin, а не argv; `MYSQL_PWD` вместо `-p`; дамп с `umask 077`, `.part` -> rename и `pipefail`; `CONFIRM=yes` для разрушительного на проде; TTY-безопасный `exec`;
- `update` - только prod: git pull от deploy-пользователя, пересборка, миграции, проверка.

## Структура

```
SKILL.md               принципы, env-файлы, чеклист
reference/env-first.md паттерн make <env> <action> целиком
reference/actions.md   таблица целей + guard-примеры
reference/safety.md    секреты, дампы, prod-дисциплина
```

## Триггеры

«сделай Makefile», «добавь цель», «допиши цели», стандартизация команд проекта под dev/prod, любая правка Makefile.

## Лицензия

MIT. См. [LICENSE](LICENSE).
