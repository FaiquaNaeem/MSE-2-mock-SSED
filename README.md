# CampusHub

CampusHub is a small campus-events app: students can browse upcoming events, register for one, and upload an event photo. Registering queues a confirmation email, and photos are stored in an object store, not on disk.

## Architecture

```
                         ┌───────────────────────────────┐
                         │           browser              │
                         └────────────────┬────────────────┘
                                           │  http://localhost:8080
                                           ▼
                         ┌───────────────────────────────┐
                         │        gateway (nginx)         │  :8080  <- only public port
                         └──┬────────────┬────────────┬───┘
                            │            │            │
                      / -> frontend  /api/ -> api  /storage/ -> minio
                            │            │            │
   ┌────────────────────────────────────────────────────────────────┐
   │ network: frontend        network: backend         network: data│
   │  gateway, frontend         gateway, api, minio      api, worker,│
   │                                                      db, redis, │
   │                                                      rabbitmq,  │
   │                                                      minio,     │
   │                                                      mailhog    │
   └────────────────────────────────────────────────────────────────┘

   worker  --(reads queue)-->  rabbitmq  --(fed by)-->  api
   worker  --(sends mail)-->   mailhog
   api     --(reads/writes)--> db (Postgres), redis (cache), minio (images)
```

Three Docker networks keep things separated:
- **frontend** — gateway, frontend. Just serves the static web page.
- **backend** — gateway, api, minio. The gateway needs to reach both the API and MinIO directly (the `/storage/` route skips the API and talks to MinIO itself).
- **data** — api, worker, db, redis, rabbitmq, minio, mailhog. The actual storage/queue/cache tier.

Only the **gateway** is published to the outside world, on port 8080. Every other service only talks to other containers.

## Service contract

| Service | Image or build | Port | Notes |
|---|---|---|---|
| gateway | nginx-unprivileged | 8080 | only port published to the host |
| frontend | `app/frontend` (multi-stage) | 8080 | static files from `npm run build` (`dist/`) |
| api | `app/api` | 3000 | `npm start` runs migration + seed, then the server |
| worker | same image as api | none | command `node src/worker.js` |
| db | postgres | 5432 | user, password, db name from env |
| redis | redis | 6379 | password required (`REDIS_PASSWORD`) |
| rabbitmq | rabbitmq | 5672 | user and password from env |
| minio | minio | 9000 | bucket `campushub` is created by the API |
| mailhog | mailhog | 1025 SMTP, 8025 UI | |

Gateway routes: `/` to frontend, `/api/` to api, `/storage/` to `minio:9000/campushub/`.
Environment variable names are in `.env.example`. Copy it to `.env` and keep `.env` out of git.

## Run it locally

```bash
cp .env.example .env
# open .env and replace every change-me placeholder with your own password
docker compose up -d --build --wait
```

Open http://localhost:8080 in a browser — you should see the CampusHub page with a list of events.

To check everything automatically:

```bash
DC="docker compose" ENV_FILE=.env bash ./grade.sh
```

This tears the stack down, rebuilds it, and runs 18 checks (health, caching, registration, email, image upload, non-root users, etc.).

## Deploy to an AWS EC2 Ubuntu instance

This uses the images already built and pushed to Docker Hub, so the server never needs to build anything itself.

1. Launch an Ubuntu EC2 instance. In its **security group**, only allow:
   - port **22** (SSH, so you can manage the box)
   - port **8080** (the gateway — this is the only thing the app needs public)
   Do not open any other port (5432, 6379, 5672, 9000, 1025, etc. should stay closed to the internet).
2. SSH in and install Docker + the Compose plugin (e.g. via Docker's official `get-docker.sh` convenience script, or your distro's packages).
3. Clone this repo:
   ```bash
   git clone <this-repo-url>
   cd MSE-2-mock-SSED
   ```
4. Create the `.env` file with real, strong passwords (never commit this file):
   ```bash
   cp .env.example .env
   nano .env   # fill in real passwords
   ```
5. Pull the pre-built images instead of building them on the server:
   ```bash
   docker compose pull
   docker compose up -d --no-build --wait
   ```
6. Visit `http://<your-ec2-public-ip>:8080`.

## Docker Hub images

The `frontend` and `api` images (the worker reuses the api image) are published publicly:

```bash
docker pull faiqua6/campushub-api:1.0
docker pull faiqua6/campushub-frontend:1.0
```

`docker-compose.yml` already points at these tags, so `docker compose pull` + `docker compose up -d --no-build --wait` is all a fresh machine needs — no source code or Dockerfiles required on the server.

## About docker-stack.yml

`docker-stack.yml` is a second orchestration file written for **Docker Swarm** instead of plain Docker Compose. It describes the same 9 services, but with Swarm-specific ideas added: how many copies (`replicas`) of each service should run, a `restart_policy` (since Swarm has no `depends_on` healthy-wait), and a `configs:` entry to hand nginx its config file across a multi-node cluster instead of a host bind-mount.

It has only been **validated**, never deployed — we ran `docker stack config -c docker-stack.yml` to confirm it parses and resolves correctly, but never ran `docker swarm init` or started a swarm. If you did want to actually run it, the commands would be:

```bash
docker swarm init
docker stack deploy -c docker-stack.yml campushub
```

## Design decisions

1. **Multi-stage build for the frontend.** `app/frontend/Dockerfile` uses Node.js only to compile the React app into static files, then throws that whole build stage away and copies just the compiled `dist/` folder into a clean nginx image. The final image that actually runs in production has no Node.js, no `node_modules`, and no source code in it — smaller, and a smaller attack surface, since there's no JavaScript runtime or package manager left to exploit.
2. **Three separate networks, only the gateway public.** Services are grouped into `frontend`, `backend`, and `data` networks so that, for example, Postgres or Redis are never reachable from outside the Docker network at all — only the gateway is published on `0.0.0.0:8080`. If somebody compromised the frontend container, they still couldn't reach the database directly over the network; they'd have to go through the gateway/API like anyone else.
3. **Healthchecks plus `depends_on: condition: service_healthy`.** Every stateful service (Postgres, Redis, RabbitMQ, MinIO) has a real healthcheck that actually tests the service (e.g. `pg_isready`, `redis-cli ping`), and the API/gateway wait on those healthchecks before starting. This avoids the classic "container is running but the app crashes because Postgres isn't ready to accept connections yet" race condition — the stack starts in the right order automatically instead of relying on luck or sleep timers.

**Note on the MinIO image**: Docker Hub removed the official `minio/minio` repository (anonymous pulls now fail), so this project uses `cgr.dev/chainguard/minio` instead — a MinIO build from Chainguard's registry that is still freely pullable and also runs as a non-root user by default.

## Security notes

- The frontend, API, worker, and gateway containers all run as **non-root users** (confirmed with `docker compose exec <service> id -u`, which returns a non-zero uid for each).
- **No secrets are committed to Git.** `.env` is listed in `.gitignore` and `.dockerignore`, and only `.env.example` (with placeholder values) is tracked.
- `docker-compose.yml` never hardcodes a password, key, or hostname — everything comes from `.env` through `env_file` or `${VARIABLE}` substitution.
- The RabbitMQ management UI (15672), MailHog UI (8025), and MinIO console (9001) are only published on `127.0.0.1`, so they're reachable for local debugging but never exposed to the public internet — only the gateway (8080) is bound to all interfaces.
