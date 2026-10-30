# ToDo Infrastructure

Shared infrastructure for the ToDo services:
[todo-api](https://github.com/Domiff/todo-api) and [todo-bot](https://github.com/Domiff/todo-bot).
The services themselves only contain application containers and connect to this stack over the
external Docker network **`todo-network`**.

## Services

| Service       | Container            | Address inside `todo-network` | Host port (from `.env`)                     | Used by            |
|---------------|----------------------|-------------------------------|---------------------------------------------|--------------------|
| PostgreSQL 16 | `todo-pg`            | `todo-pg:5432`                | `POSTGRES_PORT`                             | todo-api           |
| Redis 8       | `todo-redis`         | `todo-redis:6379`             | — (not published)                           | todo-bot (FSM)     |
| RabbitMQ 4    | `todo-rabbitmq`      | `todo-rabbitmq:5672`          | `RABBITMQ_PORT`, `RABBITMQ_MANAGEMENT_PORT` | todo-api, todo-bot |
| rabbitmq-init | `todo-rabbitmq-init` | —                             | —                                           | one-shot job       |
| Grafana       | `todo-grafana`       | `todo-grafana:3000`           | `GRAFANA_PORT`                              | you                |
| Loki          | `todo-bot-loki`      | `loki:3100`                   | `LOKI_PORT`                                 | Promtail, Grafana  |
| Promtail      | `todo-promtail`      | —                             | `PROMTAIL_PORT`                             | —                  |

Services should reach each other by **container name** (`todo-pg`, `todo-redis`, `todo-rabbitmq`):
container names are unique in the shared network, while service names like `pg` or `redis` may
clash with other compose projects attached to it.

## Getting started

1. Create the shared network once:

   ```bash
   docker network create todo-network
   ```

2. Create `.env` next to `docker-compose.yml` (it is gitignored):

   ```env
   POSTGRES_DB=todo
   POSTGRES_USER=
   POSTGRES_PASSWORD=
   POSTGRES_PORT=5432

   RABBITMQ_USER=
   RABBITMQ_PASSWORD=
   RABBITMQ_HOST=todo-rabbitmq       # used by rabbitmq-init to reach the broker
   RABBITMQ_PORT=5672
   RABBITMQ_MANAGEMENT_PORT=15672

   REDIS_HEALTHCHECK_PASSWORD=       # same as the healthcheck user's password in redis/redis.acl

   GRAFANA_PORT=3000
   GF_SECURITY_ADMIN_USER=
   GF_SECURITY_ADMIN_PASSWORD=
   LOKI_PORT=3100
   PROMTAIL_PORT=9080
   ```

   The services (todo-api, todo-bot) must use the same `POSTGRES_*` and `RABBITMQ_*` credentials
   in their own `.env` files.

3. Create the Redis users file from the template and set real passwords (it is gitignored):

   ```bash
   cp redis/redis.acl.example redis/redis.acl
   ```

   See [Redis](#redis) for what the users are for.

4. Start the stack:

   ```bash
   docker compose up -d
   ```

Start order for the whole system: **todo-infrastructure → todo-api → todo-bot**. The services have
no `depends_on` on this stack (Compose cannot depend on another project), so the infrastructure
must already be healthy when they start.

## RabbitMQ topology

The broker topology is described in [`rabbitmq/definitions.json`](rabbitmq/definitions.json) and owned by this repo,
not by the services: todo-api only publishes to an exchange, todo-bot only consumes from a queue.

```
todo-api ──> todo.tasks (topic) ──tasks.#──> todo-bot.tasks ──> todo-bot
```

| Object           | Type          | Notes                                                    |
|------------------|---------------|----------------------------------------------------------|
| `todo.tasks`     | topic exchange, durable | todo-api publishes task events here            |
| `todo-bot.tasks` | queue, durable          | consumed by todo-bot                           |
| binding          | `tasks.#`               | every task event (`tasks.created`, `tasks.deleted`, ...) |

How it is applied: the broker starts with the user from `RABBITMQ_USER` / `RABBITMQ_PASSWORD`,
and once it is healthy the one-shot `rabbitmq-init` container imports `definitions.json` through
the management API (`rabbitmqadmin definitions import`) and exits.

- Import is idempotent and safe to run on every start.
- Import **adds** objects but never changes or removes existing ones. To change an existing
  object (e.g. a binding's routing key or an exchange type), delete it in the management UI first,
  then re-run the import:

  ```bash
  docker compose up rabbitmq-init
  ```

- Users are intentionally not stored in `definitions.json`, so no password hashes end up in git.

Management UI: `http://localhost:${RABBITMQ_MANAGEMENT_PORT}`.

## Redis

Redis stores the bot's FSM state. It starts with [`redis/redis.conf`](redis/redis.conf) and has no
port published to the host: only containers in `todo-network` can reach it.

Configuration (`redis/redis.conf`):

| Setting                                       | Value                                                   |
|-----------------------------------------------|---------------------------------------------------------|
| `bind 0.0.0.0`, `port 6379`                   | listen inside the container network                     |
| `protected-mode yes`                          | refuse non-local clients if no password is configured  |
| `aclfile /usr/local/etc/redis/redis.acl`      | users and permissions (mounted from `redis/redis.acl`) |
| `appendonly yes`, `appendfsync everysec`      | AOF journal: at most ~1 second of writes lost on a crash |
| `save 3600 1 300 100 60 10000`                | RDB snapshots in addition to AOF                        |
| `maxmemory 128mb`, `maxmemory-policy noeviction` | memory cap; writes fail instead of silently evicting FSM state |
| `logfile ""`                                  | logs go to stdout (`docker logs`, Promtail)             |

Users (`redis/redis.acl`, created from [`redis/redis.acl.example`](redis/redis.acl.example)):

| User          | Permissions                                              | Used by                          |
|---------------|----------------------------------------------------------|----------------------------------|
| `default`     | disabled                                                 | —                                |
| `todo-bot`    | keys `fsm:*` only; read, write and keyspace commands, no dangerous ones | todo-bot (`REDIS_USER`, `REDIS_PASSWORD`) |
| `healthcheck` | `PING` only                                              | the container healthcheck        |

The healthcheck runs `redis-cli --user healthcheck ping` and takes the password from
`REDISCLI_AUTH`, which is set from `REDIS_HEALTHCHECK_PASSWORD`.

`redis/redis.acl` contains passwords and is gitignored. An empty or broken ACL file is a common
cause of problems: with a syntax error Redis does not start, and with an empty file the `default`
user stays enabled without a password, so `protected-mode` rejects the bot.

## Logs

Promtail reads the logs of all Docker containers on the host and ships them to Loki; Grafana has
Loki provisioned as a datasource (`grafana/datasources.yml`).

Grafana: `http://localhost:${GRAFANA_PORT}`, credentials from `GF_SECURITY_ADMIN_*`.

## Data

| Data            | Docker volume                              |
|-----------------|--------------------------------------------|
| PostgreSQL      | `todo-infrastructure_pg_data`              |
| RabbitMQ        | `todo-infrastructure_rabbitmq_data`        |
| Redis (bot FSM) | `todo-infrastructure_redis_data`           |
| Grafana         | `todo-infrastructure_grafana_data`         |

`docker compose down` keeps the data; `docker compose down -v` deletes the volumes.

## Repository layout

```
.
├── docker-compose.yml
├── grafana/                # Grafana Labs stack: Grafana, Loki, Promtail
│   ├── datasources.yml     # Grafana → Loki datasource
│   ├── loki-config.yml
│   └── promtail-config.yml
├── rabbitmq/
│   └── definitions.json    # RabbitMQ topology (exchanges, queues, bindings)
└── redis/
    ├── redis.conf          # Redis server configuration
    ├── redis.acl.example   # template for redis.acl (users and permissions)
    └── redis.acl           # real users with passwords (gitignored, create from the example)
```

Not in git: `.env` and `redis/redis.acl` (secrets).
