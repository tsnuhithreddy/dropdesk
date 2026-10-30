# Phase 1: Environment and Database Foundation

**Project:** DropDesk
**Milestone:** M0 (setup and pilot ask)
**Status:** Complete

## Summary

This phase prepared the development machine and put a working MySQL database under Docker Compose. Nothing user-facing exists yet. The aim was to remove environment risk before any application code is written, and to confirm early that the database version supports the two MySQL features the concurrency design depends on: `CHECK` constraints and `SKIP LOCKED`.

## Goals

- Confirm the required tools are installed and working.
- Create the repository layout for a monorepo (web, api, worker).
- Keep secrets out of version control from the first day.
- Run MySQL 8 in Docker with a healthcheck and persistent storage.
- Verify the exact MySQL version before designing around it.
- Start a decisions log so every choice has a recorded reason.

## What was done

1. **Checked the toolchain.** Docker, Docker Compose, Git and Node were all present (versions below).
2. **Created the repository structure.** One repository holds `apps/web`, `apps/api`, `apps/worker`, `infra`, `loadtest`, `docs` and `.github/workflows`.
3. **Set up secret handling.** A git-ignored `.env` holds real credentials. A committed `.env.example` documents which variables exist, using placeholder values only.
4. **Wrote the Compose file** at `infra/docker-compose.yml` for a single MySQL service.
5. **Started MySQL and verified it**: the container reports `healthy` and the server reports its version.
6. **Started the decisions log** at `docs/decisions.md`.

## Environment

| Component | Version |
|---|---|
| Docker Engine / CLI | 29.7.2 |
| Docker Desktop | 4.89.0 |
| Docker Compose | v5.5.0 |
| Git | 2.50.1 (Windows) |
| Node.js | v22.20.0 |
| WSL | 3.0.3.0 (WSL2, Ubuntu present) |
| MySQL (container) | **8.4.13** |
| Host OS | Windows, PowerShell and the VS Code terminal |

## Repository layout after this phase

```
dropdesk/
  apps/
    web/
    api/
    worker/
  infra/
    docker-compose.yml
  loadtest/
  docs/
    decisions.md
  files/                  phase documentation (this folder)
  .github/workflows/
  .env                    local only, ignored by git
  .env.example
  .gitignore
```

Git does not track empty folders, so `apps/web`, `apps/worker`, `loadtest` and `.github/workflows` will appear in version control once they contain files.

## The Compose configuration

The MySQL service is defined in `infra/docker-compose.yml`. The choices that matter:

| Setting | Reason |
|---|---|
| `image: mysql:8.4` | Everyone, including CI and a reviewer's laptop, gets the same database version. 8.4 is still MySQL 8 and supports both features we need. |
| Credentials from `${...}` variables | Passwords come from `.env`, so none appear in the file. |
| Port mapping `3307:3306` | Avoids a clash with any MySQL already installed on the host at 3306. |
| Named volume `mysql_data` | Data survives the container being removed or recreated. |
| Healthcheck using `mysqladmin ping` | A running container is not the same as a database that is ready to accept queries. Later services will wait for `healthy`. |
| `$$MYSQL_ROOT_PASSWORD` in the healthcheck | The doubled `$` makes the variable resolve inside the container instead of on the host. |

Because the Compose file lives in `infra/` while `.env` lives in the repository root, every command passes `--env-file .env` explicitly.

## Verification

Container status, from `docker compose --env-file .env -f infra/docker-compose.yml ps`:

```
dropdesk-mysql   mysql:8.4   "docker-entrypoint.s..."   mysql   About a minute ago   Up About a minute (healthy)   0.0.0.0:3307->3306/tcp
```

Version check, from `SELECT VERSION();` run inside the container:

```
+-----------+
| VERSION() |
+-----------+
| 8.4.13    |
+-----------+
```

## Why the MySQL version matters

The design relies on two features, each with a minimum version:

- **`CHECK` constraints** are enforced from MySQL 8.0.16. They will act as a safety net so that `held + paid` can never exceed `total` for an item, even if application code has a bug.
- **`SELECT ... FOR UPDATE SKIP LOCKED`** is available from MySQL 8.0.1. The expiry worker will use it so that two workers never pick up the same overdue hold.

Version 8.4.13 is above both thresholds. This confirms support on paper only. Both features still need to be exercised with real queries and tests, which happens in M1 (`CHECK`) and M3 (`SKIP LOCKED`).

## A problem along the way: Docker Desktop and WSL

On first launch, Docker Desktop reported "WSL not installed" and the engine would not start. Every `docker compose` command failed with an error about the `dockerDesktopLinuxEngine` pipe.

I did not reinstall anything straight away. Running `wsl --install` returned `ERROR_ALREADY_EXISTS`, which suggested WSL and Ubuntu were already present. Inspecting the real state showed WSL 3.0.3.0 installed, Ubuntu registered as a WSL2 distribution, and the Virtual Machine Platform feature enabled. A test command inside Ubuntu ran successfully. After confirming this and relaunching Docker Desktop, the engine started and `docker info` returned a `Server` section.

I did not pin down a single root cause. The most likely explanation is a start-up ordering problem, with Docker Desktop launching before WSL was usable. The useful lesson is the order of work: inspect the actual state first, and only then change something.

## Decisions made in this phase

| ID | Decision | Reason |
|---|---|---|
| D-001 | Monorepo (`apps/web`, `apps/api`, `apps/worker`) | One person, one repository, simpler CI and Compose. |
| D-002 | MySQL in Docker, image `mysql:8.4` | Same version on every machine. Verified as 8.4.13. |
| D-003 | Host port 3307 for MySQL | Avoids a clash with a local MySQL on 3306. |
| D-004 | Secrets only in `.env`; `.env.example` committed | No secrets in the repository. |
| D-005 | PowerShell and the VS Code terminal, no Git Bash | Owner preference. Files are created in the editor to avoid encoding problems from shell redirection. |
| D-006 | Keep the existing Ubuntu WSL distribution | WSL was already installed and working. Docker Desktop manages its own internal Linux. |
| D-007 | No commits unless the owner asks | The owner decides what enters git history. |
| D-008 | Phase documentation lives in `files/` | One place for the write-up of each phase. |

The full log is in `docs/decisions.md`.

## What is not done yet

- No schema exists. The database is empty.
- No application code has been written.
- The API, worker and web services are not yet in Compose.
- `CHECK` and `SKIP LOCKED` have not been tested with real queries.
- No metrics have been collected. Nothing in this phase produces a result that needs a real or synthetic label.

## Talking points for interviews

- I confirmed the database version before building on `CHECK` constraints and `SKIP LOCKED`, because the correctness guarantees depend on them. A version mismatch found in M3 would have meant rework.
- The whole stack runs from Docker Compose, so local development, CI and a demo use the same database version.
- Credentials live in a git-ignored `.env`. The repository ships only `.env.example`, so anyone can configure the project without exposing secrets.
- When Docker failed to start, I inspected what was installed before changing anything. Reinstalling blindly would have hidden the actual state of the machine.
- The healthcheck exists because "container started" and "database ready" are different things. Dependent services should wait for the second.

## Reproducing this phase

From the repository root in PowerShell:

```powershell
Copy-Item .env.example .env
# Edit .env and set your own passwords (letters and digits only)

docker compose --env-file .env -f infra/docker-compose.yml up -d
docker compose --env-file .env -f infra/docker-compose.yml ps
docker compose --env-file .env -f infra/docker-compose.yml exec mysql mysql -uroot -p -e "SELECT VERSION();"
```

Wait for `healthy` in the `ps` output before running the version check. MySQL reads the passwords from `.env` only on its first start, when the volume is created. Changing them afterwards does not change the database's passwords.

## Next

Phase 2 adds the API skeleton: an Express server in TypeScript with a `/health` endpoint and one automated test, running locally. It is containerized and added to Compose in Phase 3.
