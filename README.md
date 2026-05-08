# Ansible Role: redash

Деплой [Redash](https://redash.io/) в Docker Swarm (single-node или multi-node).

## Что делает роль

1. Проверяет, что нода является **Swarm manager**
2. (Опционально) логинится в приватный Docker registry
3. Создаёт директории (`/var/redash`, postgres data dir)
4. Рендерит `docker-compose.yml` из шаблона
5. Деплоит стек через `docker stack deploy`
6. Ждёт, пока сервис `redash` поднимется
7. (Опционально) запускает `manage.py db upgrade` для первичной инициализации БД

## Сервисы в стеке

| Сервис      | Образ                                  | Описание                    |
|-------------|----------------------------------------|-----------------------------|
| `postgres`  | `postgres:12-alpine`                   | База данных                 |
| `redis`     | `redis:3-alpine`                       | Очередь задач / кэш         |
| `redash`    | `registry.unitpay.ru/unitpay/redash`   | Web-приложение              |
| `worker`    | то же                                  | Celery worker               |
| `scheduler` | то же                                  | Celery beat scheduler       |

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
| `redash_image`              | `registry.unitpay.ru/unitpay/redash`        | Образ Redash                      |
| `redash_version`            | `0.0.12`                                    | Тег образа                        |
| `redash_stack_name`         | `redash`                                    | Имя Swarm-стека                   |
| `redash_listen_port`        | `5000`                                      | Публичный порт web-приложения     |
| `redash_postgres_data_dir`  | `/opt/redash/postgres`                      | Путь к данным PostgreSQL          |
| `redash_compose_dir`        | `/opt/redash`                               | Директория для compose-файла      |
| `redash_network_name`       | `redash`                                    | Имя overlay-сети                  |

### Registry

| Переменная                  | По умолчанию  | Описание                               |
|-----------------------------|---------------|----------------------------------------|
| `redash_registry_auth`      | `false`       | Включить логин в registry              |
| `redash_registry_url`       | `""`          | URL registry                           |
| `redash_registry_username`  | `""`          | Логин                                  |
| `redash_registry_password`  | `""`          | Пароль (хранить в vault!)              |

### Traefik

| Переменная               | По умолчанию | Описание                       |
|--------------------------|--------------|--------------------------------|
| `redash_traefik_enabled` | `false`      | Добавить labels для Traefik    |
| `redash_traefik_host`    | `""`         | Hostname для Traefik router    |

### Первичный деплой

| Переменная        | По умолчанию | Описание                                            |
|-------------------|--------------|-----------------------------------------------------|
| `redash_db_init`  | `false`      | Запустить `manage.py db upgrade` (только 1-й раз)  |

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
```

### Только пересоздать стек

```bash
ansible-playbook playbooks/redash.yml -i inventory/prod --tags deploy
```

## Структура файлов на сервере

```
/var/redash/
├── docker-compose.yml     # рендерится Ansible
└── postgres/              # данные PostgreSQL (bind mount)
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
