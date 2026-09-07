# Docker & Docker Compose Mastery — Write From Memory, Not From Copy-Paste

> Goal: given any application, you should be able to look at it and derive the Dockerfile and Compose file yourself — not recall one you saw once.

---

## PART 1 — THE MASTER MENTAL MODEL

### 1.1 The universal containerization pipeline

Memorize this chain. Every Dockerfile you will ever write is an answer to these questions, in order.

```
Application
  ↓ what language/runtime?
Base image
  ↓ what OS packages does the runtime need?
System deps
  ↓ what does the app declare (package.json, requirements.txt, go.mod, pom.xml)?
App deps
  ↓ copy source
Source
  ↓ compile / bundle / transpile if needed
Build
  ↓ who runs the process?
Non-root user
  ↓ what port does it listen on?
Port
  ↓ what env does it need at runtime?
Config
  ↓ how do we know it's alive?
Healthcheck
  ↓ what starts the process?
Start command
```

### 1.2 The Dockerfile skeleton (memorize this shape, not files)

```
BASE       → FROM
SYSTEM     → RUN apt/apk install
DEPS       → COPY manifest + RUN install
FILES      → COPY source
BUILD      → RUN build step (if compiled/bundled language)
CONFIG     → ENV / ARG / WORKDIR / USER
RUNTIME    → EXPOSE / VOLUME / HEALTHCHECK
START      → ENTRYPOINT / CMD
```

Say it out loud as: **"Base, System, Deps, Files, Build, Config, Runtime, Start."** This order also happens to be the *correct cache order* — that's not a coincidence, it's why the order exists.

---

## PART 2 — DOCKERFILE KEYWORDS, ONE BY ONE

For each instruction: what/why/where/syntax/example/mistake/memory trick.

### `FROM` — the starting point
- **What/why**: picks the base filesystem + OS your image builds on top of.
- **Where**: always the first non-comment line (except `ARG` before `FROM` for base-image parameterization).
- **Syntax**: `FROM <image>[:<tag>] [AS <name>]`
- **Example**: `FROM node:20-alpine`
- **Production**: `FROM node:20.11.1-alpine3.19` — pin exact versions, never `latest`.
- **Mistake**: using `latest` → builds become non-reproducible.
- **Security**: smaller/official/verified base images = smaller attack surface.
- **Memory trick**: FROM = "what OS do I start standing on?"

### `ARG` — build-time variable
- **What/why**: a variable that exists only during `docker build`, not in the running container.
- **Where**: before `FROM` (to parameterize the base image) or inside a stage.
- **Syntax**: `ARG NODE_VERSION=20`
- **Example**: `ARG NODE_VERSION=20` / `FROM node:${NODE_VERSION}`
- **Mistake**: putting secrets in `ARG` — they persist in image history/layer cache. Use `--mount=type=secret` (BuildKit) instead.
- **Memory trick**: ARG = "Available during build, Reset after" (gone at runtime).

### `ENV` — runtime environment variable
- **What/why**: sets an environment variable baked into the image, visible to the running container and to later build steps.
- **Where**: after `WORKDIR`, wherever config is needed.
- **Syntax**: `ENV PORT=8000`
- **Mistake**: `ENV PASSWORD=secret123` — never bake secrets into an image; anyone with `docker history` sees it.
- **Memory trick**: ENV = "Every process from here oN sees this Value."

### `WORKDIR` — set the working directory
- **What/why**: like `cd`, but persists across all following instructions and at container start.
- **Syntax**: `WORKDIR /app`
- **Mistake**: using `RUN cd /app && ...` instead — doesn't persist between layers.
- **Memory trick**: WORKDIR = "cd, but it sticks."

### `COPY` — bring files into the image
- **What/why**: copies files from build context into the image filesystem.
- **Syntax**: `COPY package.json ./` then later `COPY . .`
- **Production pattern**: copy dependency manifests *before* source code, so dependency installation is cached:
  ```
  COPY package*.json ./
  RUN npm ci
  COPY . .
  ```
- **Mistake**: `COPY . .` first, then `RUN npm install` → cache busts on every source change.
- **Memory trick**: COPY = "bring only what changed last."

### `ADD` — COPY's overpowered cousin
- **What/why**: like COPY, but also auto-extracts local `.tar` archives and can fetch URLs.
- **Recommendation**: prefer `COPY` always, unless you specifically need tar auto-extraction. `ADD` with URLs is discouraged (no cache validation, no cleanup of temp downloads, security risk of unverified remote content).
- **Memory trick**: ADD = "Auto-extract, Dangerous with URLs — avoid."

### `RUN` — execute during build
- **What/why**: runs a command while building the image; result is committed as a new layer.
- **Syntax**: `RUN apt-get update && apt-get install -y curl && rm -rf /var/lib/apt/lists/*`
- **Mistake**: separate `RUN apt-get update` and `RUN apt-get install` in different layers → stale cache installs old package lists.
- **Performance**: chain related commands with `&&` and clean caches in the *same* `RUN` to keep layers small.
- **Memory trick**: RUN = "happens once, at build time, baked in forever."

### `USER` — who runs the process
- **What/why**: switches from root to a non-privileged user for security.
- **Syntax**:
  ```
  RUN addgroup -S app && adduser -S app -G app
  USER app
  ```
- **Mistake**: never setting `USER` → container runs as root (major security issue if compromised).
- **Memory trick**: USER = "who's driving? Never let root drive in prod."

### `EXPOSE` — documentation, not publishing
- **What/why**: documents which port the app listens on. It does **not** publish the port to the host — that's `-p` at `docker run` time.
- **Syntax**: `EXPOSE 8000`
- **Mistake**: thinking `EXPOSE` makes the port reachable from the host. It doesn't.
- **Memory trick**: EXPOSE = "a label on the box," `-p` = "the door to the outside."

### `VOLUME` — declare a persistent mount point
- **What/why**: marks a directory as external, unmanaged data (survives container removal).
- **Syntax**: `VOLUME /var/lib/postgresql/data`
- **Mistake**: declaring VOLUME for directories you also `COPY` into — writes to that path afterward are ignored because a volume is mounted over it.
- **Memory trick**: VOLUME = "this data outlives the container."

### `ENTRYPOINT` — the fixed executable
- **What/why**: the "main program" of the container; not meant to be overridden casually.
- **Syntax (exec form, preferred)**: `ENTRYPOINT ["uvicorn"]`
- **Memory trick**: ENTRYPOINT = "what this image IS."

### `CMD` — default arguments / default command
- **What/why**: default command (or default *arguments* to ENTRYPOINT), easily overridden at `docker run`.
- **Syntax**: `CMD ["app.main:app", "--host", "0.0.0.0", "--port", "8000"]`
- **Memory trick**: CMD = "what it does by default — swap me out."

### `HEALTHCHECK` — is it actually alive?
- **What/why**: defines how Docker checks container health beyond "is the process running."
- **Syntax**:
  ```
  HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 \
    CMD curl -f http://localhost:8000/health || exit 1
  ```
- **Memory trick**: HEALTHCHECK = "running ≠ ready."

### `SHELL`, `STOPSIGNAL`, `ONBUILD`, `LABEL` — rarely used, know them by name
- `SHELL` changes the default shell for shell-form RUN/CMD (rare; used mainly on Windows containers).
- `STOPSIGNAL` changes the signal sent on `docker stop` (rare; useful when your process needs `SIGINT` instead of `SIGTERM`).
- `ONBUILD` triggers instructions when *another* image builds `FROM` this one (rare; used for base-image templates).
- `LABEL` attaches metadata (`LABEL maintainer="you"`), common for CI/registry tooling.

### Full keyword memory table

| Keyword | Purpose | Mental hook |
|---|---|---|
| FROM | base image | "starting point" |
| ARG | build-time var | "gone after build" |
| ENV | runtime var | "baked into the image" |
| WORKDIR | working dir | "sticky cd" |
| COPY | bring files | "bring only what changed" |
| ADD | copy + extract/URL | "avoid unless extracting tar" |
| RUN | execute at build | "baked in forever" |
| USER | who runs it | "never let root drive" |
| EXPOSE | doc port | "a label, not a door" |
| VOLUME | persistent mount | "outlives the container" |
| ENTRYPOINT | fixed program | "what it IS" |
| CMD | default args | "swap me out" |
| HEALTHCHECK | liveness/readiness | "running ≠ ready" |

---

## PART 3 — THE THREE THAT CONFUSE EVERYONE

### RUN vs CMD vs ENTRYPOINT

```
RUN         → executes NOW, during image BUILD. Result committed to a layer.
CMD         → default command run when container STARTS. Overridable.
ENTRYPOINT  → the fixed executable. CMD becomes its default arguments.
```

- `RUN` never runs at container start. `CMD`/`ENTRYPOINT` never run at build time.
- **ENTRYPOINT + CMD together** (the production pattern):
  ```
  ENTRYPOINT ["python"]
  CMD ["manage.py", "runserver", "0.0.0.0:8000"]
  ```
  Running `docker run image` → `python manage.py runserver 0.0.0.0:8000`
  Running `docker run image shell` → `python shell` (CMD is replaced, ENTRYPOINT stays fixed).

- **Exec form `["cmd", "arg"]`** vs **shell form `cmd arg`**: exec form runs the process directly as PID 1 (correctly receives `SIGTERM` on `docker stop`); shell form wraps it in `/bin/sh -c`, which can swallow signals and leave zombie processes. **Always prefer exec form in production.**

### COPY vs ADD
- Use `COPY` by default. Use `ADD` only for local tar auto-extraction. Never use `ADD` with a remote URL — no caching, no integrity check; use `RUN curl` instead.

### ARG vs ENV
- `ARG`: exists only during the build (e.g., choosing a version, a build target).
- `ENV`: exists in the built image and at runtime.
- A build secret (API key, npm token) should be **neither** — use BuildKit secret mounts (`RUN --mount=type=secret,id=npm_token`) so it never lands in a layer.

---

## PART 4 — SINGLE-STAGE DOCKERFILES (BY REQUIREMENT, NOT MEMORIZATION)

Before writing any Dockerfile, answer these:
1. Language/runtime?
2. Package manager / manifest file?
3. Build step needed (compile/bundle) or interpreted straight from source?
4. What port does it bind?
5. What starts the process?
6. Any OS-level deps (e.g., `libpq` for Postgres drivers)?

### Example: Python (FastAPI), single-stage
```dockerfile
FROM python:3.12-slim

WORKDIR /app

# deps first (cache layer)
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# then source
COPY . .

RUN addgroup --system app && adduser --system --ingroup app app
USER app

ENV PORT=8000
EXPOSE 8000

HEALTHCHECK --interval=30s --timeout=3s CMD curl -f http://localhost:8000/health || exit 1

CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Example: Node.js (Express), single-stage
```dockerfile
FROM node:20-alpine

WORKDIR /app

COPY package*.json ./
RUN npm ci --omit=dev

COPY . .

USER node
ENV PORT=3000
EXPOSE 3000

CMD ["node", "server.js"]
```

### Example: static site via Nginx
```dockerfile
FROM nginx:1.27-alpine
COPY dist/ /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
```

---

## PART 5 — MULTI-STAGE BUILDS

### Why they exist
Build tools (compilers, `node_modules` dev deps, `.git`) shouldn't ship in the final image. Multi-stage lets you use a fat "builder" stage, then copy only the finished artifact into a slim "runtime" stage.

### The universal shape
```
BUILDER  → has compilers/toolchain, does the heavy lifting
   ↓
RUNTIME  → minimal base, only the compiled artifact + runtime deps
   ↓ COPY --from=builder
START
```

### React / Vite (build → static Nginx)
```dockerfile
# ---- builder ----
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# ---- runtime ----
FROM nginx:1.27-alpine
COPY --from=builder /app/dist /usr/share/nginx/html
EXPOSE 80
```

### Go (compiled binary)
```dockerfile
# ---- builder ----
FROM golang:1.22 AS builder
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -o /app/server .

# ---- runtime ----
FROM gcr.io/distroless/static-debian12
COPY --from=builder /app/server /server
USER nonroot:nonroot
EXPOSE 8080
ENTRYPOINT ["/server"]
```

### What goes where (rule of thumb)
| Builder stage | Runtime stage |
|---|---|
| Compilers, SDKs | Only the final binary/bundle |
| Dev dependencies | Production dependencies only |
| Source code + build tools | Static assets / compiled output |
| Large base image (e.g. `node`, `golang`) | Minimal base (`alpine`, `distroless`, `slim`) |

Same pattern applies to Java/Spring Boot (`maven` builder → `eclipse-temurin-jre` runtime), .NET (`sdk` builder → `aspnet` runtime), and Next.js (`node` builder → `node:alpine` runtime with `.next/standalone`).

---

## PART 6 — .dockerignore

Excludes files from the build context (faster builds, smaller context upload, no accidental secret leakage).

```
.git
node_modules
__pycache__
*.pyc
.venv
venv
.env
*.log
dist
build
coverage
.DS_Store
```

Why it matters: without it, `.env` or `.git` can end up inside a layer via `COPY . .`, and every unrelated file bloats the context sent to the Docker daemon, slowing every build.

---

## PART 7 — IMAGE OPTIMIZATION

- Prefer `alpine`/`slim`/`distroless` bases — but **not** always Alpine: if your app needs glibc-specific native bindings (some Python C extensions, some Node native modules) Alpine's `musl` libc can cause subtle breakage or slower `pip`/`npm` installs. In that case use `-slim` (Debian-based) instead.
- Order layers from least → most frequently changing (deps before source).
- Multi-stage to drop build tools from the final image.
- Clean package manager caches in the same `RUN` that installed them.
- Run as non-root.
- Pin versions (base image and dependencies) for reproducibility.

---

## PART 8 — NETWORKING, ONE SENTENCE YOU WON'T FORGET

```
-p 8080:8000
     ↑     ↑
  HOST   CONTAINER
```
**"The number on the left is where YOU knock (your laptop's browser); the number on the right is where the APP is listening inside the container."**

- `EXPOSE` = documentation only.
- `-p HOST:CONTAINER` = actually publishes the port to the host.
- Containers on the same **user-defined bridge network** can reach each other by **service/container name** as a DNS hostname — you never need the container's IP.
- Default `bridge` network = isolated NAT network; `host` = container shares the host's network stack directly (no port mapping needed, but no isolation); `none` = no networking at all.

**Mental chain**: `HOST → Docker network (bridge) → container's own network namespace → app's listening port`.

---

## PART 9 — VOLUMES

| Type | Use case |
|---|---|
| Named volume | Docker-managed persistent data (databases) — portable, easy to back up |
| Bind mount | Mount a host path into the container (source code for hot reload) |
| tmpfs | In-memory, non-persistent (secrets, scratch space) |

```yaml
volumes:
  postgres_data:

services:
  db:
    image: postgres:16
    volumes:
      - postgres_data:/var/lib/postgresql/data
```

Chain: **Volume (lives on host, managed by Docker) → mounted into Container → Application writes to that path → data survives container removal.**

---

## PART 10 — DOCKER COMPOSE MENTAL MODEL

```
COMPOSE = SERVICES + NETWORKS + VOLUMES + CONFIGS + SECRETS
```

Top-level shape:
```yaml
services:
  <name>:
    image: / build:
    ports:
    environment: / env_file:
    volumes:
    networks:
    depends_on:
    restart:
    healthcheck:

networks:
volumes:
```

### YAML nesting — derive it, don't memorize it
Read it as a tree, top to bottom:
```
services
  └─ backend
       ├─ build
       │    ├─ context: .
       │    └─ dockerfile: Dockerfile
       ├─ ports: ["8000:8000"]
       ├─ environment: {...}
       ├─ volumes: [...]
       └─ depends_on:
            db:
              condition: service_healthy
```
Each indent level = "belongs to the thing above it." You never need to memorize the whole file — just ask "what does this key belong to?"

### Compose keyword quick table

| Keyword | Meaning | Memory trick |
|---|---|---|
| `build` | build image from a Dockerfile | "I build my own" |
| `image` | use a prebuilt image | "I pull, don't build" |
| `ports` | `HOST:CONTAINER` publish | same as `-p` |
| `environment` | inline env vars | quick config |
| `env_file` | load vars from a file | for many vars / secrets out of git |
| `volumes` | persistent/mounted data | see Part 9 |
| `networks` | which network(s) this service joins | who can talk to whom |
| `depends_on` | startup order | NOT "is ready," just "is started" |
| `restart` | restart policy | `unless-stopped` common in prod |
| `healthcheck` | readiness probe | pairs with `depends_on: condition: service_healthy` |
| `command` | override image's CMD | one-off tweak |
| `entrypoint` | override image's ENTRYPOINT | rare |
| `profiles` | conditionally include a service | dev-only/debug-only services |

**`depends_on` critical truth**: "container started" ≠ "application ready." Postgres's container can be running while Postgres itself is still initializing. Fix with `healthcheck` + `depends_on: condition: service_healthy`, not with fragile `sleep` hacks.

---

## PART 11 — COMPOSE PATTERNS (BUILDING BLOCKS)

### Backend + PostgreSQL
```yaml
services:
  backend:
    build: .
    ports:
      - "8000:8000"
    environment:
      DATABASE_URL: postgresql://user:pass@db:5432/appdb
    depends_on:
      db:
        condition: service_healthy

  db:
    image: postgres:16
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
      POSTGRES_DB: appdb
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user"]
      interval: 5s
      timeout: 5s
      retries: 5

volumes:
  postgres_data:
```

### + Redis
```yaml
  redis:
    image: redis:7-alpine
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
```

### Frontend + Backend + DB + Redis + Nginx
```yaml
services:
  nginx:
    image: nginx:1.27-alpine
    ports: ["80:80"]
    volumes:
      - ./nginx.conf:/etc/nginx/conf.d/default.conf
    depends_on: [frontend, backend]

  frontend:
    build: ./frontend

  backend:
    build: ./backend
    environment:
      DATABASE_URL: postgresql://user:pass@db:5432/appdb
      REDIS_URL: redis://redis:6379
    depends_on:
      db: { condition: service_healthy }
      redis: { condition: service_started }

  db:
    image: postgres:16
    volumes: ["postgres_data:/var/lib/postgresql/data"]

  redis:
    image: redis:7-alpine

volumes:
  postgres_data:
```

### Backend + Celery worker + beat + Redis + PostgreSQL
```yaml
services:
  backend:
    build: .
    command: gunicorn app.wsgi:application --bind 0.0.0.0:8000
    ports: ["8000:8000"]
    depends_on: [db, redis]

  worker:
    build: .
    command: celery -A app worker -l info
    depends_on: [db, redis]

  beat:
    build: .
    command: celery -A app beat -l info
    depends_on: [db, redis]

  redis:
    image: redis:7-alpine

  db:
    image: postgres:16
    volumes: ["postgres_data:/var/lib/postgresql/data"]

volumes:
  postgres_data:
```

Notice: `backend`, `worker`, `beat` reuse the **same image**, just a different `command`. This is a very common production pattern — one build, multiple roles.

---

## PART 12 — DEVELOPMENT VS PRODUCTION

| | Development | Production |
|---|---|---|
| Build | single-stage, source bind-mounted | multi-stage, immutable image |
| Reload | bind mount + hot reload (`nodemon`, `--reload`) | no source mounts at all |
| User | often root (convenience) | always non-root |
| Image size | irrelevant | minimized |
| Healthchecks | optional | required |
| Secrets | `.env` file (gitignored) | secret manager / injected at deploy |
| Resource limits | none | CPU/memory limits set |

Common hot-reload gotcha: `volumes: [".:/app"]` alongside a container-only `node_modules` — fix by adding an anonymous volume so the host doesn't overwrite the container's installed deps:
```yaml
volumes:
  - .:/app
  - /app/node_modules
```

---

## PART 13 — SECURITY CHECKLIST

- [ ] Non-root `USER` set
- [ ] No secrets in `ENV`/`ARG`/layers — use secret managers or BuildKit secret mounts
- [ ] `.dockerignore` excludes `.env`, `.git`
- [ ] Minimal base image (slim/alpine/distroless)
- [ ] Pinned image tags (no `latest`)
- [ ] `read_only: true` filesystem where possible (Compose)
- [ ] `cap_drop: [ALL]`, add back only what's needed
- [ ] Image scanned (Trivy / Docker Scout) in CI
- [ ] Dockerfile linted (Hadolint) in CI
- [ ] Docker socket never mounted into untrusted containers

Bad → better:
```dockerfile
# BAD
FROM ubuntu
RUN apt install -y curl
COPY . .
ENV PASSWORD=secret
USER root
CMD python app.py
```
```dockerfile
# BETTER
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
RUN addgroup --system app && adduser --system --ingroup app app
USER app
CMD ["python", "app.py"]
```
(Secrets are injected at runtime via the orchestrator/secret manager, never baked in.)

---

## PART 14 — DEBUGGING DECISION TREE

```
Container exits immediately
  → docker logs <container>          (see the actual error)
  → docker run -it --entrypoint sh <image>   (poke around manually)

App works locally, not in Docker
  → check ENV vars are actually set (docker exec env)
  → check the app is binding 0.0.0.0, not 127.0.0.1
  → check working directory / file paths

Can't connect to Postgres/Redis
  → are they on the same Compose network?
  → are you using the SERVICE NAME as hostname, not localhost?
  → is depends_on paired with a healthcheck (not just started)?

Port not accessible from host
  → did you EXPOSE and also -p/ports: publish it?
  → is the app actually listening on that port inside the container?

Compose service won't start
  docker compose ps → docker compose logs <service> → docker inspect
  → check environment → check network → check volumes → check health → check depends_on
```

---

## PART 15 — CHEAT SHEET (ONE PAGE)

**Dockerfile**: `FROM → ARG → ENV → WORKDIR → COPY (deps) → RUN (install) → COPY (source) → RUN (build) → USER → EXPOSE → HEALTHCHECK → ENTRYPOINT/CMD`

**Compose**: `services → build/image → ports → environment/env_file → volumes → networks → depends_on → healthcheck → restart`

**Commands**:
```
docker build -t name:tag .
docker run -p 8080:8000 -e KEY=val name:tag
docker exec -it <container> sh
docker logs -f <container>
docker inspect <container>

docker compose up -d
docker compose down
docker compose logs -f <service>
docker compose exec <service> sh
docker compose config      # validate merged config
```

**Memory sentence**: *Dockerfile = "Build the image: base, deps, source, config, start." Compose = "Run the app: services, ports, env, volumes, networks, dependencies, health."*

---

## PART 16 — ACTIVE RECALL (answer before scrolling back up)

1. What comes right after `FROM` in a well-ordered Dockerfile?
2. RUN vs CMD — one-sentence difference?
3. CMD vs ENTRYPOINT — one-sentence difference?
4. ARG vs ENV — which one survives into the running container?
5. Why does `COPY package.json .` come before `COPY . .`?
6. What does `-p 8080:8000` actually mean?
7. Does `EXPOSE` publish a port? Why or why not?
8. How does a container reach a database by name in Compose?
9. Why is `depends_on` alone not enough for "database is ready"?
10. Name two reasons Alpine isn't always the best base image.

---

## PART 17 — WRITE-FROM-MEMORY DRILLS

Try each before you'd look anything up. Requirements only — no hints:

1. Dockerfile for a Python FastAPI app, port 8000, `uvicorn app.main:app`, needs Postgres.
2. Dockerfile for a Node/Express API, port 3000, needs Redis.
3. Multi-stage Dockerfile for a React app served by Nginx.
4. Multi-stage Dockerfile for a Go binary using distroless runtime.
5. Compose file: backend + Postgres + healthcheck-gated startup.
6. Compose file: Nginx + frontend + backend + Postgres + Redis.
7. Compose file: backend + Celery worker + beat + Redis + Postgres, one shared image.
8. Add a non-root user and healthcheck to any of the above.
9. Convert any of the above from development (bind mount, hot reload) to production (multi-stage, immutable, resource limits).
10. Add `.dockerignore` for a Django project.

**Process for each**: write it → compare against the patterns in Parts 4–11 → identify the specific line you got wrong or forgot → write the memory rule for it yourself in one sentence.

---

## PART 18 — 30-DAY PRACTICE RHYTHM (2 hrs/day)

| Days | Focus |
|---|---|
| 1–4 | Dockerfile keywords + single-stage Dockerfiles (Python, Node) from empty file |
| 5–8 | Multi-stage builds (React, Go, Java) |
| 9–12 | Compose fundamentals: one service → +DB → +Redis |
| 13–16 | Full-stack Compose (Nginx + FE + BE + DB + Redis) |
| 17–19 | Healthchecks, depends_on, networking |
| 20–22 | Security hardening + optimization pass on everything built so far |
| 23–25 | Debugging drills — intentionally break your own files, then fix them |
| 26–27 | Worker/queue architecture (Celery/Sidekiq/BullMQ + Redis) |
| 28 | Full project: React + Nginx + API + Postgres + Redis + worker, from scratch |
| 29 | Code review your Day-28 project as if you were a senior engineer reviewing a PR |
| 30 | Timed exam: given only a plain-English requirement, produce Dockerfile + Compose in under 45 minutes |

Repeat the write-from-memory drills (Part 17) at Day 2, 4, 7, 14, 30, 60, 90 — spaced repetition is what converts this from "I read it once" into "I can't forget it."

---

## PART 19 — DOCKER ↔ KUBERNETES MAPPING (since you already know K8s)

| Compose concept | Kubernetes equivalent |
|---|---|
| `services.<name>` | Deployment + Pod |
| `ports` | Service (ClusterIP/NodePort) |
| `environment` / `env_file` | ConfigMap / Secret |
| `volumes` | PersistentVolumeClaim |
| `healthcheck` | livenessProbe / readinessProbe |
| `depends_on` | initContainers / readiness gates (K8s doesn't guarantee start order either) |
| `networks` | Namespace + Service DNS |
| `deploy.resources` | resources.requests/limits |

Compose is for local dev / small single-host deployments; Kubernetes is the production orchestration layer for the same conceptual pieces at scale.

---

## Closing principle

You now have the **derivation framework**, not a pile of files to memorize:

```
See an application
  → Identify runtime + package manager
  → Identify build step (or none)
  → Pick base image (builder vs runtime if multi-stage)
  → Walk the Dockerfile skeleton: BASE → SYSTEM → DEPS → FILES → BUILD → CONFIG → RUNTIME → START
  → Walk the Compose skeleton: SERVICES → BUILD/IMAGE → PORTS → ENV → VOLUMES → NETWORKS → DEPENDS_ON → HEALTH → RESTART
  → Harden: non-root, no secrets baked in, minimal image, scanned
```

Want the next layer of this course as a follow-up? I can build out, as separate focused docs: (a) 100 progressive Dockerfile challenges, (b) 100 progressive Compose challenges, (c) intentionally-broken files for code-review practice, (d) a timed final exam with a senior-engineer-style scorecard. Just say which one to generate first.
