# env-first pattern

The `make <env> <action>` interface. Copy this skeleton, then adjust compose
file names and service names to the project.

```make
SHELL := /bin/bash
.DEFAULT_GOAL := help

ENVIRONMENTS := dev prod
ENV := $(filter $(ENVIRONMENTS),$(MAKECMDGOALS))

# Разные compose-файлы для сред; все прочие различия - только в env.
COMPOSE_dev  := docker compose
COMPOSE_prod := docker compose -f docker-compose.prod.yml
COMPOSE := $(COMPOSE_$(ENV))

# Терминал выделяется, только если он есть: из скрипта, cron или ssh без -t
# `docker compose exec` с терминалом падает с "the input device is not a TTY".
EXEC := $(COMPOSE) exec $$([ -t 0 ] || echo -T)

# node напрямую, а не через npm: в выводе нет обёртки npm, и одна и та же строка
# работает и в дев-контейнере, и в прод-образе.
CLI := $(EXEC) server node src/cli/index.js
CLI_STDIN := $(COMPOSE) exec -T server node src/cli/index.js

DEPLOY_USER ?= deploy
TAIL ?= 200
DUMP_DIR ?= backups

ACTIONS := up down restart ps logs build migrate migrate-undo dump \
	cli shell db update

.PHONY: help $(ENVIRONMENTS) $(ACTIONS) env-required

help:
	@echo "Использование: make <среда> <действие> [ПАРАМЕТРЫ]"
	@echo "Среды: dev - docker-compose.yml, prod - docker-compose.prod.yml"

# Среда - отдельная цель без собственной работы: её роль - быть прочитанной в
# ENV выше. `make prod` без действия показывает справку, а не молчит.
$(ENVIRONMENTS):
	@if [ "$(words $(MAKECMDGOALS))" = 1 ]; then $(MAKE) --no-print-directory help; fi

env-required:
ifeq ($(words $(ENV)),0)
	$(error Укажите среду: make dev <действие> или make prod <действие>. Список: make help)
endif
ifneq ($(words $(ENV)),1)
	$(error Укажите одну среду, а не несколько: $(ENV))
endif
```

## Notes

- `ENV := $(filter ...)` reads the env from the command line goals, so env is a
  target with no work of its own.
- Suffix style (`start-dev` / `start-prod`) is acceptable when the project
  already uses it. New Makefiles use env-first.
- Compose file per environment, everything else identical: differences between
  dev and prod belong in compose files and env files, not in Makefile logic.
- Service env file (`server/.env`) is baked into the container at start:
  `make <env> restart` recreates the server so it rereads the file.
