# ToDo Infrastructure

Shared infrastructure for the ToDo services:
[todo-api](https://github.com/Domiff/todo-api), todo-auth, [todo-bot](https://github.com/Domiff/todo-bot) and
[todo-ai](https://github.com/Domiff/todo-ai).
The services themselves only contain application containers and connect to this stack over the
external Docker network **`todo-network`**.

## Services

| Service       | Container            | Address inside `todo-network` | Host port (from `.env`)                     | Used by            |
|---------------|----------------------|-------------------------------|---------------------------------------------|--------------------|
| PostgreSQL 16 | `todo-pg`            | `todo-pg:5432`                | `POSTGRES_PORT`                             | todo-api, todo-auth (own databases, see [PostgreSQL](#postgresql)) |
| Redis 8       | `todo-redis`         | `todo-redis:6379`             | — (not published)                           | todo-bot (FSM), todo-auth (token blocklist) |
| RabbitMQ 4    | `todo-rabbitmq`      | `todo-rabbitmq:5672`          | `RABBITMQ_PORT`, `RABBITMQ_MANAGEMENT_PORT` | todo-api, todo-bot, todo-ai |
| rabbitmq-init | `todo-rabbitmq-init` | —                             | —                                           | one-shot job       |
| Grafana       | `todo-grafana`       | `todo-grafana:3000`           | `GRAFANA_PORT`                              | you                |
| Loki          | `todo-loki`          | `todo-loki:3100`              | `LOKI_PORT`                                 | Alloy, Grafana     |
| Alloy         | `todo-alloy`         | —                             | `ALLOY_PORT`                                | —                  |
| Caddy         | `todo-caddy`         | —                             | `CADDY_HTTP_PORT`, `CADDY_HTTPS_PORT`, `CADDY_PORT` | browsers → todo-api |
| Qdrant 1.20   | `todo-qdrant`        | `todo-qdrant:6333` (REST), `:6334` (gRPC) | `127.0.0.1:6333`, `127.0.0.1:6334` (fixed) | todo-ai (task search) |

Services should reach each other by **container name** (`todo-pg`, `todo-redis`, `todo-rabbitmq`, `todo-qdrant`):
container names are unique in the shared network, while service names like `pg` or `redis` may
clash with other compose projects attached to it.

## Getting started

1. Create the shared network and the static files volume once:

   ```bash
   docker network create todo-network
   docker volume create todo-static
   ```

   `todo-static` is shared with todo-api, see [Reverse proxy](#reverse-proxy).

2. Create `.env` next to `docker-compose.yml` (it is gitignored):

   ```env
   POSTGRES_DB=postgres              # superuser's database; services have their own, see PostgreSQL
   POSTGRES_USER=                    # superuser: administration only, never used by services
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
   ALLOY_PORT=12347

   SITE_ADDRESS=:8080                # Caddy site address: `:8080` locally, a domain in production
   CADDY_PORT=8080                   # host port for the local site (container port 8080)
   CADDY_HTTP_PORT=80                # host port for HTTP (container port 80)
   CADDY_HTTPS_PORT=443              # host port for HTTPS (container port 443)
   ```

   The services (todo-api, todo-bot, todo-ai) must use the same `RABBITMQ_*` credentials in their
   own `.env` files. For PostgreSQL each service has **its own user and database** instead of the
   superuser above, see [PostgreSQL](#postgresql).

3. Create the Redis users file from the template and set real passwords (it is gitignored):

   ```bash
   cp redis/redis.acl.example redis/redis.acl
   ```

   See [Redis](#redis) for what the users are for.

4. Start the stack:

   ```bash
   docker compose up -d
   ```

5. On the first start, create the service databases and users, see
   [Creating a service database](#creating-a-service-database).

Start order for the whole system: **todo-infrastructure → todo-api, todo-auth → todo-bot, todo-ai**. The services have
no `depends_on` on this stack (Compose cannot depend on another project), so the infrastructure
must already be healthy when they start.

## Reverse proxy

[Caddy](https://caddyserver.com/docs/) is the entry point for todo-api. It serves the Django static
files itself and forwards everything else to the backend:

```
                    ┌─────────────────── todo-caddy ───────────────────┐
browser ──HTTP(S)──>│  /static/*  → /srv/static (volume todo-static)   │
                    │  everything else → todo-backend:8000 (Daphne)    │
                    └──────────────────────────────────────────────────┘
```

The config is [`caddy/Caddyfile`](caddy/Caddyfile). The site address comes from `SITE_ADDRESS` in
`.env` (`{$SITE_ADDRESS}` is substituted when Caddy loads the config, so restart Caddy after
changing it).

| `SITE_ADDRESS`     | Caddy listens on   | Published as                          | Open in the browser                  | HTTPS                                    |
|--------------------|--------------------|---------------------------------------|--------------------------------------|------------------------------------------|
| `:8080` (local)    | `8080`             | `CADDY_PORT`                          | `http://localhost:${CADDY_PORT}`     | no                                       |
| `api.example.com`  | `80` and `443`     | `CADDY_HTTP_PORT`, `CADDY_HTTPS_PORT` | `https://api.example.com`            | automatic certificate from Let's Encrypt |

The container ports are fixed: the local site listens on `8080` inside the container, so a
different local port is set with `CADDY_PORT`, not in `SITE_ADDRESS`. In production
`CADDY_HTTP_PORT` and `CADDY_HTTPS_PORT` must stay `80` and `443`: browsers open a domain on these
ports, and Let's Encrypt checks only them.

Always open todo-api through Caddy. Daphne does not serve static files, so with `DEBUG=False` the
backend opened directly shows the admin and Swagger UI without styles. The backend port is not
published to the host for this reason.

### Static files

Static files travel through the external Docker volume `todo-static`:

1. todo-api mounts it at `/backend/static` (its `STATIC_ROOT`) and runs `collectstatic` on every
   start, so the volume always holds the files of the running version.
2. Caddy mounts the same volume read-only at `/srv/static` and answers `/static/*` from it
   (`handle_path` strips the `/static` prefix: `/static/admin/css/base.css` →
   `/srv/static/admin/css/base.css`).

The volume is created by hand (`docker volume create todo-static`) and is empty until todo-api has
started once. Check that static files are served:

```bash
curl -I http://localhost:8080/static/admin/css/base.css
```

`200` means everything works; `404` means the volume is empty (todo-api has not run `collectstatic`).

### Production

To serve a real domain over HTTPS:

1. Point an A record of the domain (e.g. `api.example.com`) to the server's public IP.
2. Set `SITE_ADDRESS=api.example.com` and restart Caddy.
3. Keep `CADDY_HTTP_PORT=80` and `CADDY_HTTPS_PORT=443` and make these ports reachable from the
   internet (firewall, cloud security groups).
   Let's Encrypt checks domain ownership over them (ACME HTTP-01 on `80`, TLS-ALPN-01 on `443`),
   and `80` also redirects `http` to `https`.
4. Configure todo-api for the domain: `ALLOWED_HOSTS`, `CSRF_TRUSTED_ORIGINS` with `https://`,
   `SECURE_SSL_REDIRECT=True` (see the todo-api README).

Certificates are stored in the `caddy_data` volume and renewed automatically about 30 days before
they expire (they are valid for 90 days). Keep the volume: without it Caddy requests a new
certificate on every recreate and quickly hits Let's Encrypt rate limits.

Caddy starts without todo-api and returns `502` for everything except `/static/*` until
`todo-backend` is up.

## RabbitMQ topology

The broker topology is described in [`rabbitmq/definitions.json`](rabbitmq/definitions.json) and owned by this repo,
not by the services: todo-api only publishes to an exchange, todo-bot and todo-ai only consume from
their queues.

```
todo-api ──> todo.tasks (topic) ──tasks.#──> todo-bot.tasks ──> todo-bot
                    │                              │
                    │                           rejected
                    │                              ▼
                    │  todo.tasks.dlx (direct) ──todo-bot.tasks──> todo-bot.tasks.dlq
                    │
                    └──────────────tasks.#──> todo-ai.tasks ──> todo-ai
```

Each consumer has its own queue bound to the same exchange, so every task event is copied to both:
todo-bot sends notifications, todo-ai indexes the task for search. A new consumer only needs a new
queue and binding here; todo-api does not change.

| Object               | Type                     | Notes                                                    |
|----------------------|--------------------------|----------------------------------------------------------|
| `todo.tasks`         | topic exchange, durable  | todo-api publishes task events here                      |
| `todo-bot.tasks`     | queue, durable           | consumed by todo-bot                                     |
| binding              | `tasks.#`                | every task event (`tasks.created`, `tasks.deleted`, ...) |
| `todo-ai.tasks`      | queue, durable           | consumed by todo-ai; no dead letter queue yet            |
| binding              | `tasks.#`                | from `todo.tasks` to `todo-ai.tasks`, every task event   |
| `todo.tasks.dlx`     | direct exchange, durable | receives dead messages from `todo-bot.tasks`             |
| `todo-bot.tasks.dlq` | queue, durable           | stores dead messages; nothing consumes it                |
| binding              | `todo-bot.tasks`         | from `todo.tasks.dlx` to `todo-bot.tasks.dlq`            |

### Dead letter queue

A message that todo-bot rejects without requeue is moved to `todo-bot.tasks.dlq` instead of being
deleted. RabbitMQ adds an `x-death` header with the source queue, the reason and the time.

`todo-ai.tasks` has no dead letter queue yet: a message that todo-ai rejects is deleted.

This is configured with policies, not queue arguments, so it applies to existing queues:

| Policy                  | Queue                | Settings                                                     |
|-------------------------|----------------------|--------------------------------------------------------------|
| `todo-tasks-dlx`        | `todo-bot.tasks`     | `dead-letter-exchange`, `dead-letter-routing-key`            |
| `todo-tasks-dlq-limits` | `todo-bot.tasks.dlq` | `message-ttl` 14 days, `max-length` 10000 (oldest removed)  |

Messages leaving the DLQ by TTL or length limit are deleted permanently. To inspect or move
messages, use the queue's page in the management UI.

### How it is applied

The broker starts with the user from `RABBITMQ_USER` / `RABBITMQ_PASSWORD`,
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

## PostgreSQL

One PostgreSQL server (`todo-pg`) hosts a separate database for every service that needs one
([database per service](https://microservices.io/patterns/data/database-per-service)): each
service owns its data, connects with its own user and cannot reach other services' databases.

| Database    | Owner (user) | Used by   | Service `.env`                                                  |
|-------------|--------------|-----------|-----------------------------------------------------------------|
| `todo_api`  | `todo_api`   | todo-api  | `POSTGRES_DB=todo_api`, `POSTGRES_USER=todo_api`                |
| `todo_auth` | `todo_auth`  | todo-auth | `POSTGRES_DB=todo_auth`, `POSTGRES_USER=todo_auth`              |
| `postgres`  | superuser    | —         | maintenance database for the superuser from `POSTGRES_USER`     |

Services connect to `todo-pg:5432` from `todo-network`, or to `localhost:${POSTGRES_PORT}` when
running on the host.

How the isolation works:

- **Service users are not superusers** and cannot create databases or roles.
- **Each service user owns its database.** In PostgreSQL 15+ the `public` schema belongs to
  `pg_database_owner`, so only the database owner can create tables there; migrations (Alembic,
  Django) run as the service user.
- **`CONNECT` is revoked from `PUBLIC`** on every service database, so other users cannot even
  connect to it. Only the owner and the superuser can.
- Data of different services is never joined or linked with foreign keys; services exchange data
  through their APIs and RabbitMQ events.

### Creating a service database

The image only creates the superuser and `POSTGRES_DB` on an empty volume, so service databases are
created by hand once. Connect as the superuser:

```bash
docker exec -it todo-pg psql -U <POSTGRES_USER> -d postgres
```

Then, for a service named `todo_x`:

```sql
CREATE ROLE todo_x LOGIN;
\password todo_x
CREATE DATABASE todo_x OWNER todo_x;
REVOKE ALL ON DATABASE todo_x FROM PUBLIC;
```

`\password` asks for the password interactively, so it does not end up in the shell history or
logs. Put the same password into the service's `.env` (`POSTGRES_PASSWORD`).

Check the result:

```sql
SELECT datname, pg_get_userbyid(datdba) AS owner, datacl
FROM pg_database WHERE NOT datistemplate;
```

Each service database must be owned by its user and have only `{todo_x=CTc/todo_x}` in `datacl`.

Things to keep in mind:

- Create the database with `OWNER todo_x`. A database created by the superuser without an owner
  (e.g. from a GUI client) belongs to the superuser, and migrations fail with
  `permission denied for schema public`. Fix it with `ALTER DATABASE todo_x OWNER TO todo_x;`.
- Do not change the owner of the `public` schema by hand; it must stay `pg_database_owner`.
- Do not use `REASSIGN OWNED BY <superuser>`: the superuser also owns system objects, and the
  command fails. Transfer ownership with `ALTER DATABASE/TABLE/SEQUENCE ... OWNER TO` instead.
- `psql -U <POSTGRES_USER>` without `-d` connects to a database named like the user, which may not
  exist; pass `-d postgres`.

## Qdrant

[Qdrant](https://qdrant.tech/documentation/) is the vector database of todo-ai: it stores task
embeddings and finds tasks by meaning. todo-ai creates its collection (`tasks`) itself on startup;
nothing is configured here.

- **Image**: `qdrant/qdrant:v1.20.1`, pinned. Keep its minor version within one of the
  `qdrant-client` version in todo-ai; the client warns on startup otherwise.
- **Ports**: `6333` (REST) and `6334` (gRPC) are published on `127.0.0.1` only, so the database is
  reachable from the host (todo-ai running locally) but not from outside. From containers in
  `todo-network` use `http://todo-qdrant:6333`.
- **Storage**: bind mounts on the host, not Docker volumes:
  `/opt/qdrant/storage` (collections) and `/opt/qdrant/snapshots` (snapshots).
  Docker creates the directories on the first start.

Web UI with collections and points: `http://localhost:6333/dashboard`. Check that it is up:

```bash
curl http://127.0.0.1:6333/readyz
```

The index can always be rebuilt from todo-api's data, so losing Qdrant data loses no tasks, only
the time to re-index them.

## Redis

Redis stores the bot's FSM state and the revoked tokens of todo-auth. It starts with
[`redis/redis.conf`](redis/redis.conf) and has no port published to the host: only containers in
`todo-network` can reach it, as `todo-redis:6379`.

Configuration (`redis/redis.conf`):

| Setting                                       | Value                                                   |
|-----------------------------------------------|---------------------------------------------------------|
| `bind 0.0.0.0`, `port 6379`                   | listen inside the container network                     |
| `protected-mode yes`                          | refuse non-local clients if no password is configured  |
| `aclfile /usr/local/etc/redis/redis.acl`      | users and permissions (mounted from `redis/redis.acl`) |
| `appendonly yes`, `appendfsync everysec`      | AOF journal: at most ~1 second of writes lost on a crash |
| `save 3600 1 300 100 60 10000`                | RDB snapshots in addition to AOF                        |
| `maxmemory 128mb`, `maxmemory-policy noeviction` | memory cap; writes fail instead of silently evicting FSM state |
| `logfile ""`                                  | logs go to stdout (`docker logs`, Alloy)                |

Users (`redis/redis.acl`, created from [`redis/redis.acl.example`](redis/redis.acl.example)):

| User          | Permissions                                              | Used by                          |
|---------------|----------------------------------------------------------|----------------------------------|
| `default`     | disabled                                                 | —                                |
| `todo-bot`    | keys `fsm:*` only; read, write and keyspace commands, no dangerous ones | todo-bot (`REDIS_USER`, `REDIS_PASSWORD`) |
| `todo-auth`   | keys `auth:*` only; read, write and keyspace commands, no dangerous ones (no `PING`) | todo-auth (`REDIS_USER`, `REDIS_PASSWORD`) |
| `healthcheck` | `PING` only                                              | the container healthcheck        |

The healthcheck runs `redis-cli --user healthcheck ping` and takes the password from
`REDISCLI_AUTH`, which is set from `REDIS_HEALTHCHECK_PASSWORD`.

`redis/redis.acl` contains passwords and is gitignored. An empty or broken ACL file is a common
cause of problems: with a syntax error Redis does not start, and with an empty file the `default`
user stays enabled without a password, so `protected-mode` rejects the bot.

Redis reads the ACL file only on start. After adding a user to `redis/redis.acl`, restart the
container (`docker compose restart redis`); until then the new user gets `WRONGPASS`.

## Logs

Logs are collected by [Grafana Alloy](https://grafana.com/docs/alloy/latest/) and stored in Loki:

```
Docker containers ──> Alloy ──> Loki ──> Grafana
```

- **Alloy** ([`alloy/config.alloy`](alloy/config.alloy)) discovers all containers on the host
  through the Docker socket (`discovery.docker`), reads their logs (`loki.source.docker`) and
  pushes them to `http://loki:3100/loki/api/v1/push` (`loki.write`). Every stream gets the labels
  `service_name` (container name without the leading `/`, e.g. `todo-api`) and `platform="docker"`.
- **Loki** ([`loki/config.yml`](loki/config.yml)) is a single-node instance with filesystem storage
  and the `tsdb` / `v13` schema. Retention is 7 days (`retention_period: 168h`, applied by the
  compactor).
- **Grafana** has Loki provisioned as a datasource ([`grafana/datasources.yml`](grafana/datasources.yml)).

Example query in Grafana → Explore:

```logql
{service_name="todo-bot"} |= "error"
```

Alloy UI (pipeline graph and component health): `http://localhost:${ALLOY_PORT}`.

Grafana: `http://localhost:${GRAFANA_PORT}`, credentials from `GF_SECURITY_ADMIN_*`.

## Data

| Data            | Docker volume                              |
|-----------------|--------------------------------------------|
| PostgreSQL (`todo_api`, `todo_auth`) | `todo-infrastructure_pg_data` |
| RabbitMQ        | `todo-infrastructure_rabbitmq_data`        |
| Redis (bot FSM, auth blocklist) | `todo-infrastructure_redis_data` |
| Grafana         | `todo-infrastructure_grafana_data`         |
| Loki (logs)     | `todo-infrastructure_loki_data`            |
| Caddy (TLS certificates) | `todo-infrastructure_caddy_data`  |
| Django static files | `todo-static` (external, shared with todo-api, filled by `collectstatic`) |
| Qdrant (task embeddings) | host directories `/opt/qdrant/storage`, `/opt/qdrant/snapshots` (bind mounts) |

`docker compose down` keeps the data; `docker compose down -v` deletes the volumes except the
external `todo-static` (remove it with `docker volume rm todo-static`). Qdrant data is not in a
volume, so `down -v` keeps it too; delete `/opt/qdrant` by hand to reset it.

## Repository layout

```
.
├── docker-compose.yml
├── alloy/
│   └── config.alloy        # log collection: Docker containers → Loki
├── caddy/
│   └── Caddyfile           # reverse proxy for todo-api, serves /static/*
├── grafana/
│   └── datasources.yml     # Grafana → Loki datasource
├── loki/
│   └── config.yml          # Loki storage, schema and retention
├── rabbitmq/
│   └── definitions.json    # RabbitMQ topology (exchanges, queues, bindings)
└── redis/
    ├── redis.conf          # Redis server configuration
    ├── redis.acl.example   # template for redis.acl (users and permissions)
    └── redis.acl           # real users with passwords (gitignored, create from the example)
```

Not in git: `.env` and `redis/redis.acl` (secrets).
