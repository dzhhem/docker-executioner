# Docker eXecutioner

`dx` is a project-agnostic Docker eXecutioner CLI — a Compose wrapper that simplifies container management for both development and production environments. It provides smart command routing between the host machine and running containers, making it ideal for monorepo setups — with or without Turborepo.

## Features

- **Smart Proxy:** Detects whether commands should run on the host or be forwarded to the runner container automatically.
- **Dev & Prod Stacks:** Separate lifecycle commands for development (`compose.dev.yaml`) and production (`compose.yaml`) — prefix any command with `p` for production.
- **Task Runner Detection:** Automatically detects `npm`, `pnpm`, `yarn`, `bun` or `make` based on project files.
- **First-time Setup:** Copies all `*.example` env files to their real counterparts with a single command.
- **Shell Shortcuts:** Open an interactive shell or run a one-off command in any service container (dev stack).
- **Runner Container:** Proxies `npm run <script>` calls through a dedicated runner container for consistent environments.
- **Safety Guards:** Destructive commands (`downv`, `pdownv`) require interactive confirmation.
- **Configurable:** All defaults are overridable via environment variables — no need to edit the script.
- **Docker Detection:** Automatically detects when running inside a container and executes commands locally instead of re-entering Docker.
- **Named Arguments:** Consistent short and long flag support across all commands.

## Requirements

- `bash`
- `docker` with the Compose plugin (`docker compose`)

### Runner container requirements

The runner container used by `dx run` and `dx exec` must include:

- `bash`
- `git` *(optional, for `codebase-dump.sh`)*
- `postgresql-client` *(optional, for `db-dump.sh` / `db-restore.sh`)*
- `perl` *(optional, for binary detection in `codebase-dump.sh`)*
- `docker-cli` *(optional, for Docker-in-Docker commands)*

A minimal `Dockerfile.runner` example:

```dockerfile
FROM node:22-alpine

RUN apk add --no-cache bash git postgresql-client perl docker-cli

WORKDIR /app

CMD ["bash"]
```

## Companion Scripts

To keep `dx` lightweight and focused, specialized tasks like database backups and codebase exports are maintained in separate repositories. You should place these scripts in the `scripts/` directory (alongside `Dockerfile.runner`):

- [**db-dump**](https://github.com/dzhhem/db-dump): Smart PostgreSQL backup script (Docker-aware).
- [**db-restore**](https://github.com/dzhhem/db-restore): Database restoration script with schema reset support.
- [**codebase-dump**](https://github.com/dzhhem/codebase-dump): AI-ready project codebase exporter (respects `.gitignore`).


## Installation

Copy `dx` into your project root and make it executable:

```bash
chmod +x dx
```

That's it. No global install required.

## Usage

```bash
bash dx <command> [args...]
```

Or, after making it executable:

```bash
./dx <command> [args...]
```

## Setup

```bash
bash dx setup
```

Copies all `*.example` files in the project to their real counterparts (e.g. `.env.example` → `.env`). Skips files that already exist. Run once after cloning, then fill in your values.

## Dev lifecycle

| Command | Description |
|---|---|
| `bash dx dev [args]` | Start all containers in detached mode |
| `bash dx devb [args]` | Start dev containers in detached mode with --build |
| `bash dx stop [svc]` | Stop running containers (keeps volumes) |
| `bash dx down [args]` | Stop and remove containers |
| `bash dx downv [args]` | Stop and remove containers + volumes *(requires confirmation)* |
| `bash dx restart [svc]` | Restart one or all services |
| `bash dx build [args]` | Build images |
| `bash dx rebuild [args]` | Build images without cache |
| `bash dx logs [svc]` | Follow log output (Ctrl-C to stop) |
| `bash dx shell <svc> [cmd...]` | Open an interactive shell or run a command in a service |

If the target container is not running, `shell` falls back to `docker compose run --rm` automatically.

## Production lifecycle

Prefix any lifecycle command with `p` to target the production stack (`compose.yaml`):

| Command | Description |
|---|---|
| `bash dx prod [args]` | Start the production stack in detached mode |
| `bash dx prodb [args]` | Start the production stack in detached mode with --builds |
| `bash dx pstop [svc]` | Stop production containers |
| `bash dx pdown [args]` | Stop and remove production containers *(requires confirmation)* |
| `bash dx pdownv [args]` | Stop and remove production containers + volumes *(requires confirmation)* |
| `bash dx prestart [svc]` | Restart one or all production services |
| `bash dx pbuild [args]` | Build production images |
| `bash dx prebuild [args]` | Build production images without cache |
| `bash dx plogs [svc]` | Follow production log output |
| `bash dx pshell <svc> [cmd...]` | Open an interactive shell in a production container |

## Status

| Command | Description |
|---|---|
| `bash dx ps` | List containers with status and health |

## Runner commands

These commands are routed through the runner container (or executed locally if already inside a container):

| Command | Description |
|---|---|
| `bash dx run <script> [args]` | Run an npm script or make target at the root |
| `bash dx exec <cmd> [args]` | Run any arbitrary command in the runner |
| `bash dx sh` | Open an interactive shell in the runner |

### Smart routing for `dx run`

If the script definition in `package.json` contains `dx ` (i.e. it itself manages Docker), `dx run` executes it on the host. Otherwise, it forwards to the runner container. This prevents unnecessary nesting.

**Example `package.json` scripts:**

```json
{
  "scripts": {
    "test:api": "npm run test -w project-api",
    "test:api:dx": "bash dx shell api npm run test",
    "restore:db": "bash scripts/db-restore.sh -e .env",
    "restore:db:dx": "bash scripts/db-restore.sh -e .env.docker",
  }
}
```

- `bash dx run test:api` → forwarded to runner container
- `bash dx run test:api:dx` → executed on host (contains `dx `)

## Service shortcuts

Any unrecognized command is treated as a service name and opens a shell in the **development** stack:

```bash
bash dx api       # equivalent to: bash dx shell api
bash dx web       # equivalent to: bash dx shell web
bash dx postgres  # equivalent to: bash dx shell postgres
```

## Configuration

All defaults can be overridden via environment variables — no need to edit the script:

| Variable | Default | Description |
|---|---|---|
| `DX_DEV_COMPOSE` | `compose.dev.yaml` | Path to the dev Compose file |
| `DX_PROD_COMPOSE` | `compose.yaml` | Path to the prod Compose file |
| `DX_RUNNER` | `runner` | Runner service name |
| `DX_RUNNER_PROFILE` | `tools` | Compose profile for the runner service |

Example:

```bash
DX_DEV_COMPOSE=docker/dev.yaml bash dx dev
```

## Compose file setup

### Development (`compose.dev.yaml`)

The runner service must be defined with the correct profile so it does not start automatically with `dx dev`:

```yaml
services:
  runner:
    build:
      context: .
      dockerfile: scripts/Dockerfile.runner
    working_dir: /app
    profiles: ["tools"]
    volumes:
      - .:/app
      - node_modules:/app/node_modules
    env_file:
      - apps/api/.env.docker
    command: bash
```

### Production (`compose.yaml`)

Standard Compose file — no special requirements. Docker eXecutioner uses it as-is for all `p`-prefixed commands.

## Monorepo usage (with or without Turborepo)

Docker eXecutioner works independently of Turborepo. It relies only on your Compose files and the task runner it detects.

**With Turborepo**, your `package.json` scripts might look like:

```json
{
  "scripts": {
    "dev": "turbo run dev",
    "build": "turbo run build",
    "test": "npm run test:api && npm run test:web",
    "test:dx": "bash dx shell api npm run test && bash dx shell web npm run test"
  }
}
```

**Without Turborepo**, they work the same way — Docker eXecutioner only cares about the task runner (`npm`, `pnpm`, `yarn`, `bun`) and your Compose files.

## .gitignore recommendation

```
# Env
.env
.env.local
.env.*.local
.env.docker
!.env.example
!**/.env.example
!**/.env.docker.example

# Dumps (e.g. database dumps, codebase dumps, etc.)
dumps/
```

## Full command reference

```bash
bash dx help
```