# Ansible Role: redash

Деплой [Redash](https://redash.io/) в Docker Swarm (single-node или multi-node).

## Что делает роль

1. Проверяет, что нода является **Swarm manager**
2. (Опционально) логинится в приватный Docker registry
3. Создаёт директории (`/var/redash`, postgres data dir)
4. Рендерит `docker-compose.yml` из шаблона
5. Деплоит стек через `docker stack deploy`
6. Ждёт, пока сервис `server` поднимется
7. (Опционально) `redash_db_init`: `manage.py database create_tables` + `db upgrade` на первом деплое
8. (Опционально) `redash_db_migrate`: только `manage.py db upgrade` — для обновления версии с миграциями

## Сервисы в стеке

| Сервис             | Образ                   | Описание                                         |
|--------------------|-------------------------|--------------------------------------------------|
| `postgres`         | `postgres:17-alpine`    | Метаданные Redash                                |
| `redis`            | `redis:7-alpine`        | Очереди RQ, без persistence                      |
| `server`           | `redash/redash:26.9.0`  | Web-приложение (gunicorn)                        |
| `scheduler`        | то же                   | RQ scheduler                                     |
| `worker`           | то же                   | RQ worker: `periodic`, `emails`, `default`       |
| `adhoc_worker`     | то же                   | RQ worker: `queries` (запросы из UI)             |
| `scheduled_worker` | то же                   | RQ worker: `scheduled_queries`, `schemas`        |

## Переменные

### Обязательные (нужно переопределить)

| Переменная                  | Описание                                   |
|-----------------------------|--------------------------------------------|
| `redash_postgres_password`  | Пароль к PostgreSQL                        |
| `redash_secret_key`         | SECRET_KEY для Redash (генерируется 1 раз) |
| `redash_cookie_secret`      | COOKIE_SECRET для Redash                   |

### Основные

| Переменная                  | По умолчанию                                | Описание                          |
|-----------------------------|---------------------------------------------|-----------------------------------|
| `redash_image`              | `redash/redash`                             | Образ Redash                      |
| `redash_version`            | `26.9.0`                                    | Тег образа                        |
| `redash_postgres_version`   | `17-alpine`                                 | Тег образа `postgres`             |
| `redash_redis_version`      | `7-alpine`                                  | Тег образа `redis`, нужен Redis ≥ 4 |
| `redash_stack_name`         | `redash`                                    | Имя Swarm-стека                   |
| `redash_listen_port`        | `5000`                                      | Публичный порт web-приложения     |
| `redash_postgres_data_dir`  | `/opt/redash/postgres`                      | Путь к данным PostgreSQL          |
| `redash_compose_dir`        | `/var/redash`                               | Директория для compose-файла      |
| `redash_network_name`       | `redash`                                    | Имя overlay-сети                  |

### Логи

| Переменная             | По умолчанию | Описание                                                        |
|------------------------|--------------|-----------------------------------------------------------------|
| `redash_log_driver`    | `json-file`  | Logging driver всех сервисов стека                              |
| `redash_log_max_size`  | `50m`        | `max-size`, только для `json-file` и `local`                    |
| `redash_log_max_file`  | `3`          | `max-file`, только для `json-file` и `local`                    |

Для остальных драйверов (например, `fluentd`) роль задаёт только имя драйвера. Опции (`fluentd-address`, `tag`) Docker берёт из `log-opts` в `daemon.json`, если там тот же драйвер.

### Registry

| Переменная                  | По умолчанию  | Описание                               |
|-----------------------------|---------------|----------------------------------------|
| `redash_registry_auth`      | `false`       | Включить логин в registry              |
| `redash_registry_url`       | `registry.ru` | URL registry                           |
| `redash_registry_username`  | `""`          | Логин                                  |
| `redash_registry_password`  | `""`          | Пароль (хранить в vault!)              |

### Traefik

| Переменная               | По умолчанию | Описание                       |
|--------------------------|--------------|--------------------------------|
| `redash_traefik_enabled` | `false`      | Добавить labels для Traefik    |
| `redash_traefik_host`    | `""`         | Hostname для Traefik router    |

### База данных

| Переменная          | По умолчанию | Описание                                                          |
|---------------------|--------------|-------------------------------------------------------------------|
| `redash_db_init`    | `false`      | `create_tables` + `db upgrade` (только 1-й раз)                   |
| `redash_db_migrate` | `false`      | Только `db upgrade`. Игнорируется, если включён `redash_db_init`  |

## Использование

### Первый деплой

```bash
# Добавить секреты в vault
ansible-vault edit inventory/prod/group_vars/redash/vault.yml

# Задеплоить и инициализировать БД
ansible-playbook playbooks/redash.yml -i inventory/prod -e redash_db_init=true
```

### Обновление версии

```bash
# Поменять redash_version в vars, затем:
ansible-playbook playbooks/redash.yml -i inventory/prod
# Если в релизе есть миграции БД:
ansible-playbook playbooks/redash.yml -i inventory/prod -e redash_db_migrate=true
```

Перед обновлением снимите бэкап метаданных (`pg_dump -Fc`) и сверьтесь с release notes Redash: там указано, есть ли новые миграции.

### Смена мажорной версии PostgreSQL

Дефолт `redash_postgres_version` — `17-alpine`. Если в `redash_postgres_data_dir` уже лежат данные другой мажорной версии (до этой роли дефолтом был `12-alpine`), новый контейнер их не откроет и не стартует. Два варианта:

- оставить старую версию явно, например `redash_postgres_version: "12-alpine"`;
- перенести данные dump/restore:
  1. остановить сервисы приложения (`docker service rm <stack>_server <stack>_scheduler <stack>_worker <stack>_adhoc_worker <stack>_scheduled_worker`);
  2. снять дамп клиентом новой версии: `pg_dump -Fc` одной БД `redash` (не `pg_dumpall`: в новых образах `scram-sha-256`, а `pg_dumpall` перенесёт md5-хеш пароля);
  3. остановить `postgres`, переименовать каталог данных и создать пустой с владельцем `70:70`, правами `0700` — по тому же пути, потому что опции volume `postgres_data` в Docker неизменяемы;
  4. поднять временный контейнер новой версии на этом каталоге, сделать `pg_restore`, затем `vacuumdb --analyze-in-stages`, остановить контейнер;
  5. обновить `redash_postgres_version` и прогнать роль.

`postgres:18` и новее хранят данные в `/var/lib/postgresql/18/docker`, а шаблон монтирует `/var/lib/postgresql/data`. Для 18 шаблон нужно доработать.

### Redis

Начиная с Redash 25.1.0 статус `refresh_queries` пишется через `hset("redash:status", mapping=…)`, то есть многопольным `HSET`. Его нет в Redis 3 (до 25.1.0 был `hmset`). На `redis:3-alpine` задача падает каждые 30 секунд. Данных в Redis нет, только очереди RQ, поэтому версию можно менять без переноса.

### Только пересоздать стек

```bash
ansible-playbook playbooks/redash.yml -i inventory/prod --tags deploy
```

## Структура файлов на сервере

```
/var/redash/                 # redash_compose_dir
└── docker-compose.yml       # рендерится Ansible
/opt/redash/postgres/        # redash_postgres_data_dir, bind mount в postgres
```

## Vault-пример

```yaml
# inventory/prod/group_vars/redash/vault.yml
vault_registry_username: "deployer"
vault_registry_password: "s3cr3t"
vault_postgres_password: "pg_s3cr3t"
vault_redash_secret_key: "$(python -c 'import secrets; print(secrets.token_hex(32))')"
vault_redash_cookie_secret: "$(python -c 'import secrets; print(secrets.token_hex(32))')"
```
