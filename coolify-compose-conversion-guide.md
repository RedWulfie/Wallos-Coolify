# Converting a `docker-compose.yaml` project into a true One-Click Coolify app

> **Goal.** A finished conversion must be *indistinguishable* from a first-class Coolify **Service** template: you (or anyone) paste the compose file into Coolify, point a domain at it, hit deploy, and everything — subdomain, SSL, DB password, storage, routing, healthchecks — is wired automatically. No manual port mapping, no hand-edited Traefik labels, no hardcoded secrets, no "worked once then broke on redeploy."

This guide is grounded in the **live Coolify docs** (fetched `2026-08-10`; raw corpus in `./coolify-docs/raw/`) and in **two real conversions**:

- **AdventureLog** — [`RedWulfie/AdventureLog`](https://github.com/RedWulfie/AdventureLog) (already converted on the `main` branch — the reference implementation).
- **Wallos** — [`ellite/wallos`](https://github.com/ellite/wallos) (worked end-to-end below).

Two repos, two archetypes: AdventureLog is a **multi-service stack** (app + PostGIS) that needs a *shared generated DB password*; Wallos is a **single-SQLite container** that needs *storage + first-run auto-init*. Covering both teaches you the whole trick set.

---

## TL;DR — the ONE principle that makes this work

> **In Coolify, the compose file is the single source of truth.** Everything you would normally click in the UI — env vars, volumes, domains, healthchecks, the proxy labels — must be *declared in the compose file itself*. Coolify then reads that file and generates the rest (routers, labels, certs, networks, generated secrets) for you.

The mechanism that gives you **one-click parity** is Coolify's **magic environment variables**, `SERVICE_<TYPE>_<IDENTIFIER>`. They let you write a compose file that auto-generates:

| Need | Magic variable | What Coolify gives you |
|------|----------------|------------------------|
| Auto subdomain per service | `SERVICE_URL_APP` | `http://app-<uuid>.example.com` |
| Auto FQDN per service | `SERVICE_FQDN_APP` | `app-<uuid>.example.com` |
| Auto-shared DB password | `SERVICE_PASSWORD_64_DB` | a 64-char random password, **the same value** in every service that references it |
| Auto secret / encryption key | `SERVICE_BASE64_32_SECRET_KEY` | a 32-char random string |
| Random credentials | `SERVICE_USER_*`, `SERVICE_PASSWORD_*` | random usernames / passwords |

Once you've sprayed `SERVICE_*` references through your compose, the project behaves like a Coolify-native service — because that is literally how a Coolify service template is written. See [`get-started/contribute/service`](coolify-docs/raw/get-started_contribute_service.txt).

---

## Part 0 — Coolify architecture in 60 seconds (what you're turning your project INTO)

Understand the moving parts before you touch a single line.

- **Server** — a host Coolify manages (could be `localhost`).
- **Project → Environment → Resource** — Coolify's hierarchy. A "resource" is either an **Application** (1 container, built from source / image) or a **Service / Service Stack** (N containers, defined by one compose file). **Your converted project is a Service Stack.**
- **Destination** — a Docker network. Each Compose stack is deployed onto its **own isolated bridge network, named after the resource UUID** (e.g. `ewc08w0`). Every service in the stack talks to its siblings by service name (`http://db:5432`).
- **Reverse proxy** (by default **Traefik**) — owns host `80/443`. Coolify adds the proxy container to your stack's network and generates the routing labels for you. **You never expose host ports** — Traefik reaches services on their *container* ports directly.
- **The compose file** — single source of truth. If it's not in the compose, it won't reliably survive — because in the Service-Stack model the UI doesn't hold your app's config, the file does.

> ⚠️ **The #1 rule that saves every conversion:** there is exactly **one** thing on the host listening on `80/443` — Coolify's proxy. A converted project must **not** declare host port publishes for its frontend; it must **not** ship its own competing proxy. (Cross-reference: [`selfhosted-stack-coexistence` skill](..%2F..%2F.home%2F.hermes%2Fskills%2Fdevops%2Fselfhosted-stack-coexistence%2FSKILL.md).)

---

## Part 1 — The magic variable system, completely

The core of the whole trick. From [`knowledge-base/docker/compose`](coolify-docs/raw/knowledge-base_docker_compose.txt) and [`knowledge-base/environment-variables`](coolify-docs/raw/knowledge-base_environment-variables.txt):

```
SERVICE_<TYPE>_<IDENTIFIER>
```

where `<IDENTIFIER>` is your service name (uppercased). Coolify detects these at deploy time (for Git-based compose sources this needs **v4.0.0-beta.411+**, otherwise it only works for pasted Service templates), **generates the value once, and injects the same value into every service in the stack that references it**.

### The full `TYPE` table

| `TYPE` | Generated value | Example output |
|--------|-----------------|----------------|
| `URL` | URL from your wildcard domain | `http://app-vgsco4o.example.com` |
| `URL_<PORT>` | URL proxied to a specific container port | `http://app-vgsco4o.example.com:3000` |
| `URL=<path>` | URL with a path appended | `http://app-vgsco4o.example.com/api` |
| `URL_<PORT>=<path>` | Both port routing + path | `http://app-vgsco4o.example.com:3000/api` |
| `FQDN` | FQDN (no scheme) of the generated URL | `app-vgsco4o.example.com` |
| `FQDN_<PORT>` / `FQDN=<path>` … | same modifiers as URL | `app-vgsco4o.example.com/api` |
| `USER` | random string, 16 chars | `a8Kd3fR2mNpQ1xYz` |
| `PASSWORD` | random password, **no symbols** | `G7hkL9mpQ2rT4vXw` |
| `PASSWORD_64` | random password, no symbols, **64 chars** | `qG7hkL9mpQ2rT4vXw8BnP6sYd…` |
| `PASSWORDWITHSYMBOLS` | random password **with symbols** | `G7!kL9#pQ2rT4vXw` |
| `PASSWORDWITHSYMBOLS_64` | symbols, 64 chars | `qG7!kL9#pQ2rT4vXw8BnP6sYd…` |
| `BASE64` / `BASE64_32` | random string, **not** base64-encoded, 32 chars | `x9Yf2KqLm4NpR7TdWb8ZcA1eG3hJ5kM` |
| `BASE64_64` / `BASE64_128` | same, 64 / 128 chars | — |
| `REALBASE64` / `REALBASE64_32` | **base64-encoded**, 32 chars | `eDlZZjJLcUxtNE5wUjdUZA==` |
| `REALBASE64_64` / `REALBASE64_128` | base64-encoded, 64 / 128 | — |
| `HEX_32` / `HEX_64` / `HEX_128` | random hex string | `7f8c98a98db56b0c6c8768b1db6d24…` |

### Rules that bite

1. **All generated variables are reusable and consistent.** `SERVICE_PASSWORD_64_DB` referenced by `app` **and** by `db` yields the *same* value in both — that is how you hand the DB password to your app without ever writing it down.
2. **Identifier with underscores can't take a port.** `SERVICE_URL_APPWRITE_SERVICE_3000` ❌. Use hyphens: `SERVICE_URL_APPWRITE-SERVICE_3000` ✅. (The underscore is swallowed by the port suffix parser.)
3. **`SERVICE_URL_*` and `SERVICE_FQDN_*` are read-only in the UI** (they're derived). All the password/secret types **are editable** in the env UI — so you can override a generated secret if you need a fixed value.
4. **Declare magic vars as first-class entries** so Coolify both generates *and* shows them. In AdventureLog:
   ```yaml
   environment:
     SERVICE_URL_APP:
     SERVICE_PASSWORD_64_DB:
     SERVICE_BASE64_32_SECRET_KEY:
     SITE_URL: "${SERVICE_URL_APP:-${SITE_URL:-http://localhost:8015}}"
     POSTGRES_PASSWORD: "${SERVICE_PASSWORD_64_DB:-${POSTGRES_PASSWORD:-changeme123}}"
     SECRET_KEY: "${SERVICE_BASE64_32_SECRET_KEY}"
   ```
   - `SERVICE_URL_APP:` → Coolify generates the subdomain; `SITE_URL` falls back to it, then to a localhost default.
   - `SERVICE_PASSWORD_64_DB:` → Coolify generates a 64-char no-symbol password; it's handed to the app as `POSTGRES_PASSWORD` *(and independently to the `db` service)* while still being editable in the UI if you ever want to pin one.
   - The `:-default` chain means **the file still works outside Coolify** (plain `docker compose --env-file .env up`), which is what keeps it a good citizen in both worlds.

---

## Part 2 — Automatic service subdomains (`SERVICE_URL_*`, `SERVICE_FQDN_*`, `COOLIFY_*`)

This is how a single compose stack fans out to *multiple automatically-generated subdomains*.

### 2.1 Set up the wildcard domain (one-time, server level)

From [`knowledge-base/dns-configuration`](coolify-docs/raw/knowledge-base_dns-configuration.txt) and [`knowledge-base/domains`](coolify-docs/raw/knowledge-base_domains.txt):

1. **DNS:** create an `A` record for `*.example.com` pointing at your server's IP (a wildcard). Or an `A` record for the bare domain if you only want one host.
2. **Coolify → Server → Settings → Wildcard Domain:** set it to e.g. `https://example.com`.
3. Now **every** resource you create gets a random subdomain (`random.example.com`), and preview deploys get `<PRId>.random.example.com`.

### 2.2 Wire the subdomain for each service

In your compose, reference `SERVICE_URL_<SERVICENAME>` (or `SERVICE_FQDN_...`). Coolify generates a **distinct subdomain per service**, so a stack with `app`, `api`, and `db` can expose three different URLs from one file:

```yaml
services:
  app:
    environment:
      # → http://app-<uuid>.example.com   (proxied to container port 80)
      SERVICE_URL_APP:
      # app knows its own public URL
      PUBLIC_URL: "${SERVICE_URL_APP}"
  api:
    environment:
      # → http://api-<uuid>.example.com:3000  (proxied to container port 3000)
      SERVICE_URL_API_3000:
  # db is deliberately NOT given a SERVICE_URL_ — so it stays internal-only.
  db:
    image: postgres:16
```

Rules to remember:
- **Port suffix** (`_3000`) tells Traefik which *container* port to route to. The user still sees a normal `https://` on `80/443`.
- **No `SERVICE_URL_` on a private service** = it never gets a public entrypoint (ideal for `db`, `redis`, internal workers).
- **`COOLIFY_FQDN` / `COOLIFY_URL`** are *predefined* env vars Coolify injects into an Application with a domain — handy if the app needs its URL without the compose magic.

### 2.3 THE footgun: custom networks

From [`applications/build-packs/docker-compose`](coolify-docs/raw/applications_build-packs_docker-compose.txt) + issue refs #4483/#6215/#6153:

> **If your compose defines custom `networks:`, remove them.**

```
services:
  frontend:
    networks:
      - my-network
  backend:
    networks:
      - my-network
networks:
  my-network:
    driver: bridge
```

If you keep a custom network, your containers land on **two** networks — Coolify's and yours. Traefik only lives on Coolify's network but nondeterministically picks *which* IP to route to. If it picks the custom-network IP it can't reach the container → **intermittent 504 Gateway Timeout (or hangs) that "worked yesterday"**. Delete the whole `networks:` block; Coolify's auto-created network already provides inter-service DNS by service name.

### 2.4 Connect across stacks (optional)

To reach a resource in a *different* stack (e.g. a separately-deployed database), enable **Connect to Predefined Network** on the Service Stack page — then reference the other service by its **uuid-suffixed** full name (`postgres-<uuid>`), not the short name, and note that *internal Docker DNS may not behave as expected*.

---

## Part 3 — DB password handling (two strategies)

There are two correct ways to give a compose stack a database, and you should pick deliberately.

### Strategy A — ship the DB inside the stack (AdventureLog's approach)

Best when the app and DB are meant to be one unit, and you want the secret to be **auto-generated and shared** (nothing in your repo). Use `SERVICE_PASSWORD_64_<DB>` so both services read the same generated password:

```yaml
services:
  app:
    image: myapp:latest
    environment:
      SERVICE_PASSWORD_64_DB:            # generates a 64-char no-symbol password
      PGHOST: db
      POSTGRES_DB: "${POSTGRES_DB:-database}"
      POSTGRES_USER: "${POSTGRES_USER:-app}"
      POSTGRES_PASSWORD: "${SERVICE_PASSWORD_64_DB:-${POSTGRES_PASSWORD:-changeme123}}"
    depends_on:
      db:
        condition: service_healthy
  db:
    image: postgis/postgis:16-3.5
    environment:
      SERVICE_PASSWORD_64_DB:            # SAME identifier → SAME generated value
      POSTGRES_DB: "${POSTGRES_DB:-database}"
      POSTGRES_USER: "${POSTGRES_USER:-app}"
      POSTGRES_PASSWORD: "${SERVICE_PASSWORD_64_DB:-${POSTGRES_PASSWORD:-changeme123}}"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER:-app} -d ${POSTGRES_DB:-database}"]
    volumes:
      - postgres_data:/var/lib/postgresql/data
```

Why this is "parity": `SERVICE_PASSWORD_64_DB` is a **single** source of truth Coolify generates; both the app and the DB use it, so they always agree; it's editable/locked in the env UI; and it **never** lives in a `.env` file in your repo.

### Strategy B — use a managed Coolify database (separate resource)

If you want backups, scaling, versioning, and SSL handled by Coolify instead of bolt-on, deploy a **Database resource** separately (one click), then reference it. Coolify exposes predefined vars for the inbound connection; service templates commonly consume `COOLIFY_DATABASE_URL`:

```yaml
services:
  app:
    image: myapp:latest
    environment:
      DATABASE_URL: "${COOLIFY_DATABASE_URL}"
```

You then use the **Connect to Predefined Network** option and point at `postgres-<uuid>`. Managed DBs get automated backup/S3, SSL modes, and import/restore out of the box ([`databases`](coolify-docs/raw/databases_postgresql.txt)).

### Choosing

- **In-stack (A)** when the DB is bespoke (PostGIS, specific extensions) or you want a self-contained, portable compose file (Coolify's own one-click templates almost always ship the DB in-stack).
- **Managed (B)** when you want Coolify-managed backups, want to share one DB across several apps, or want automatic SSL/retention.

### Password gotchas

- **Symbols break things.** Postgres passwords, encryption keys, `SECRET_KEY`s, and passphrases stuffed into `$`-bearing `PASSWORDWITHSYMBOLS` values can be interpreted as env references by Docker/Compose. Prefer the **no-symbol** variants (`SERVICE_PASSWORD_64`, `SERVICE_BASE64_32`, `SERVICE_HEX_64`) for most container uses. If you *must* use symbols, mark the variable **Literal** in the UI or escape `$` as `$$` in labels.
- **Never hardcode** a real secret in the commit. `changeme123` as a `:-` fallback is fine; a real password is not.
- **Startup ordering is mandatory.** Use `depends_on: db: condition: service_healthy` **and** give the DB a `healthcheck`, or the app can connect before Postgres finishes init and crash in a restart loop.

---

## Part 4 — Traefik / routing config

You almost never write raw Traefik labels for a converted compose stack — Coolify generates them from your **domain + ports**. But you must *understand* them, because healthchecks and the few cases where you *do* need labels depend on it.

### 4.1 How a domain becomes a router

From [`knowledge-base/domains`](coolify-docs/raw/knowledge-base_domains.txt): you enter an FQDN on the Service Stack page (`https://api.example.com:3000` or `https://example.com/api`). Coolify:

- Generates the `traefik.http.routers.*` rule (`Host(\`api.example.com\`)`),
- auto-applies `redirect-to-https` + `gzip` middlewares,
- wires the `tls.certresolver=letsencrypt` + `tls=true` so **HTTPS/Let's Encrypt is free** (renewed automatically; if issuance fails you get a self-signed cert, not a dead site).

**The port in the domain is the CONTAINER port, not the host port.** `https://example.com:3000` tells Traefik to route to the container's *internal* port 3000; the browser still talks to `https://example.com` on 443.

### 4.2 Container vs host port — the classic mistake

```
ports:
  - "8282:80"        # ← host:container
  - "80"             # ← random host port, or
                     #  no publish at all for a frontend
```

- **For anything you route through Traefik, drop the host publish** and let the domain's container port drive it.
- **Only keep a host `ports:` map** if something *cannot* be proxied (UDP, WebRTC STUN/TURN 3478, LiveKit 7881/7882 — Traefik is HTTP/TCP only). For those, bind to localhost first (`127.0.0.1:3478:3478`) or open the cloud firewall rule deliberately.
- If you keep a `ports:` entry on a *public* frontend, it bypasses the proxy and your HTTPS/DNS-cert logic entirely — you're back to raw HTTP.

### 4.3 Path-based routing

Append a path to a domain to share one host across apps: `https://example.com/api`. Coolify applies priority automatically (more specific path wins): `/api/v2/users` > `/api/v2` > `/api` > `/`. And if the app serving `/api` goes unhealthy, traffic falls back to the root app.

### 4.4 Adding custom middlewares (the compose shorthand)

From [`knowledge-base/proxy/traefik/custom-middlewares`](coolify-docs/raw/knowledge-base_proxy_traefik_custom-middlewares.txt). For compose stacks, use the `coolify.traefik.middlewares` label — Coolify injects it into the router chain for you instead of you hand-editing router labels:

```yaml
services:
  myapp:
    image: nginx:alpine
    labels:
      - "traefik.http.middlewares.my-ratelimit.ratelimit.average=100"
      - "traefik.http.middlewares.my-ratelimit.ratelimit.period=1m"
      - "traefik.http.middlewares.security-headers.headers.frameDeny=true"
      - "coolify.traefik.middlewares=my-ratelimit,security-headers"
```

- Multiple middlewares → comma-separated.
- Referencing a middleware defined server-side in Traefik's dynamic config: append `@file` (`coolify.traefik.middlewares=my-ipallowlist@file`).
- `coolify.traefik.middlewares` is a **deploy-time directive**, removed at runtime — it does not appear on the running container.

### 4.5 Basic auth, wildcard certs, custom SSL

- **Basic auth** for an app or service: create the middleware with an `htpasswd` value and reference it as above (remember to escape `$` in hashes, or tick "Escape special characters in labels").
- **Wildcard cert** (`*.example.com`): one cert covering every subdomain instead of per-resource — needs the **DNS challenge** (ACME via DNS provider), not the HTTP challenge. Useful if you fan out many generated subdomains from this stack. (See [`wildcard-certs`](coolify-docs/raw/knowledge-base_proxy_traefik_wildcard-certs.txt).)
- **Custom / private CA certs**: Coolify lets you plug in your own certificates without relying on Let's Encrypt.

### 4.6 Raw Compose Deployment (advanced escape hatch)

If you deploy with **Raw Compose Deployment** you bypass most magic and must hand-write the Traefik labels to reach the proxy:
```yaml
labels:
  - "coolify.managed=true"
  - "coolify.applicationId=5"
  - "coolify.type=application"
  - "traefik.enable=true"
  - "traefik.http.routers.myapp.rule=Host(`example.com`) && PathPrefix(`/`)"
  - "traefik.http.routers.myapp.entryPoints=http"
```
(Warning: advanced-only, as docs state; you lose the auto-domain/auto-cert flow.)

---

## Part 5 — Persistent storage / volumes

From [`knowledge-base/persistent-storage`](coolify-docs/raw/knowledge-base_persistent-storage.txt). Two kinds:

| Kind | Where data lives | When to use |
|------|------------------|-------------|
| **Named volume** | Docker-managed; Coolify appends the resource UUID to the name (no collisions) | Everything persistent: DBs, uploads, config |
| **Bind mount** | A host path you give Coolify | You need the files on the host directly, or a known path |

### Coolify-only compose extensions

Coolify adds two extensions valid **only inside its compose parser** (not stock Docker Compose):

```yaml
volumes:
  # 1. Tell Coolify to CREATE a host directory for a bind mount
  - type: bind
    source: ./srv
    target: /srv
    is_directory: true            # Coolify creates ./srv if missing

  # 2. Tell Coolify to WRITE a file with content (optionally pulling an env value)
  - type: bind
    source: ./99-roles.sql
    target: /docker-entrypoint-initdb.d/init-scripts/99-roles.sql
    content: |
      -- env value is injected at deploy time
      ALTER USER authenticator WITH PASSWORD :'pgpass';
```

Or the top-level `configs:` element for file-based config. These are exactly how one-click templates ship seed config without a `config/` folder in the repo.

### The big conversion rule for bind mounts

**Most upstream compose files use *relative* bind mounts** (`./db:...`, `./logos:...`). Relative paths resolve **relative to the compose file's directory**, which:
- is fine on the app's own Docker host,
- **fragile inside Coolify** (where the compose may be loaded from a git clone / service template, and you don't control the working dir).

**Convert relative bind mounts to named volumes** for clean one-click parity:

```yaml
# before (upstream)
volumes:
  - './db:/var/www/html/db'
  - './logos:/var/www/html/images/uploads/logos'

# after (Coolify-ready): named volumes Coolify managers, UUID-uniquified
volumes:
  - wallos_db:/var/www/html/db
  - wallos_logos:/var/www/html/images/uploads/logos
volumes:
  wallos_db:
  wallos_logos:
```

If you genuinely need a bind mount, use an **absolute** host path and/or `is_directory: true`.

### Non-root / UID ownership

Images that run as a non-root user (Wallos runs php-fpm as `www-data`, UID 82; many others use UID 65532 or `PUID/PGID`) can fail to write to a freshly-created volume because the mount is owned by root. Fixes:
- In **named volumes**, the container's user usually works because Docker copies ownership from the image at mount time. When it doesn't, inject an env to set ownership (Wallos: `PUID` / `PGID`).
- In **bind mounts**, you may need to `chown` the host dir to the container UID.

---

## Part 6 — Environment variables in depth

From [`knowledge-base/environment-variables`](coolify-docs/raw/knowledge-base_environment-variables.txt).

### Compose-side syntax Coolify understands

```yaml
environment:
  # Hardcoded → passes to container, NOT shown/editable in Coolify UI
  - FIXED=value
  # Creates an uninitialized env editable in Coolify's UI
  - MY_VAR=${MY_VAR}
  # Creates an editable env with a default
  - PORT=${PORT:-8080}
  # REQUIRED — deploy is blocked until set; red border if empty
  - DATABASE_URL=${DATABASE_URL:?}
  # REQUIRED with a prefilled default
  - LOG_LEVEL=${LOG_LEVEL:?info}
```

`${VAR:?}` forces the user to set it before deploy (great for critical config); `${VAR:-default}` is optional-friendly. Coolify highlights missing required vars in red and refuses partial deploys.

### UI views

- **Normal view** — one card per variable, with flags: *Build* / *Runtime* (default both), *Multiline* (SSH keys, certs, multi-line configs), *Literal* (prevent `$VAR` interpolation — use for regexes and passwords containing `$`).
- **Developer view** — a `.env`-style text editor for bulk paste; locked secrets show as `(Locked Secret…)`; multiline vars must be edited in Normal view.

### Build vs runtime

Every var has a Build and Runtime flag, both on by default. Turn **Build off** for runtime-only secrets (API keys read at boot) so they're not baked into the image. For anything sensitive that must reach the *build*, turn on **Use Docker Build Secrets** → Coolify passes it via BuildKit `--secret` (not `--build-arg`), so it stays out of `docker history` and image layers.

### Shared variables

Define a shared var once (Team / Project / Environment) and reference it in any compose var:
```yaml
environment:
  NODE_ENV: "{{environment.NODE_ENV}}"    # environment-scoped
  # project-scoped: {{project.X}}   team-scoped: {{team.X}}
```

### Predefined variables Coolify injects

| Var | Meaning |
|-----|---------|
| `COOLIFY_FQDN` | FQDN(s) of the App |
| `COOLIFY_URL` | URL(s) of the App |
| `COOLIFY_BRANCH` | source branch |
| `COOLIFY_RESOURCE_UUID` | resource UUID |
| `COOLIFY_CONTAINER_NAME` | container name |
| `SOURCE_COMMIT` | git commit hash (only if "Include Source Commit in Build" is on) |
| `PORT` | defaults to first Port Exposes value |
| `HOST` | defaults to `0.0.0.0` |
| `SERVICE_NAME_<ID>` | the service name in a stack (handy for preview deploys where names vary) |

---

## Part 7 — Health checks & startup ordering

From [`knowledge-base/health-checks`](coolify-docs/raw/knowledge-base_health-checks.txt).

- **Service stacks (compose) require healthchecks in the Dockerfile `HEALTHCHECK` or in the compose `healthcheck:`.** UI-configured checks are for Applications, not compose stacks.
- **Traefik only routes to healthy containers** when healthchecks are on. Unhealthy → `404 Not Found` / `No available server` (this is the #1 reason a "deployment works but the URL 404s").
- **`depends_on: condition: service_healthy`** gates child startup so the app doesn't race the DB.

Docker Compose build pack requires the `healthcheck:` to be *on* for the domain to resolve traffic. Use `exclude_from_hc: true` on a one-shot/migration service so it doesn't hold the whole stack "unhealthy" when it exits.

**AdventureLog** (app healthcheck) + **DB** (`pg_isready`):
```yaml
app:
  healthcheck:
    test: ["CMD", "python3", "-c", "import urllib.request; urllib.request.urlopen('http://127.0.0.1:80/health')"]
    interval: 30s
    timeout: 10s
    retries: 3
    start_period: 60s
  depends_on:
    db:
      condition: service_healthy
db:
  healthcheck:
    test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER:-adventure} -d ${POSTGRES_DB:-database}"]
    interval: 10s
    timeout: 5s
    retries: 5
    start_period: 10s
```

**Wallos** already has a Dockerfile `HEALTHCHECK` (`curl http://127.0.0.1/health.php`). Redeclaring it in the compose makes it explicit and "coolify-visible" (compose `healthcheck:` overrides the image one). Note the image's healthcheck uses `--start-interval`, which requires **Docker Engine 25+** — if your server is older, use a compose `healthcheck:` that omits it so the container isn't perpetually "unhealthy".

---

## Part 8 — Worked conversion #1: AdventureLog (the reference)

Already done on `main`. This is a **multi-service + shared-DB-password** layout. The whole file (source: [`RedWulfie/AdventureLog`](https://github.com/RedWulfie/AdventureLog)) does three things that make it Coolify-first-class:

1. **Declare the magic vars first-class** so Coolify generates them:
   ```yaml
   SERVICE_URL_APP:
   SERVICE_PASSWORD_64_DB:
   SERVICE_BASE64_32_SECRET_KEY:
   ```
2. **Map generated values onto the app's real config**, each with a `:-` fallback so it still runs outside Coolify:
   ```yaml
   SITE_URL: "${SERVICE_URL_APP:-${SITE_URL:-http://localhost:8015}}"
   POSTGRES_PASSWORD: "${SERVICE_PASSWORD_64_DB:-${POSTGRES_PASSWORD:-changeme123}}"
   SECRET_KEY: "${SERVICE_BASE64_32_SECRET_KEY}"
   ```
3. **Keep the DB in-stack, healthy-gated, with no host port publish** (the proxy routes container 80; `db` has no `SERVICE_URL_` so it stays private):
   ```yaml
   ports:
     - "${HOST_PORT:-8015}:80"     # optionally keep for non-Coolify runs
   depends_on:
     db:
       condition: service_healthy
   ```

Note the intentional choice of `SERVICE_BASE64_32_SECRET_KEY` (no symbols, stable length) for a Django secret key, and `SERVICE_PASSWORD_64_DB` for the Postgres password — both picked to dodge shell-escape and signups-footguns you'd hit with symbol-laden secrets.

---

## Part 9 — Worked conversion #2: Wallos (end-to-end)

Wallos is the **single-container, SQLite, bind-mount** archetype. Upstream compose:

```yaml
services:
  wallos:
    image: bellamy/wallos:latest
    ports:
      - "8282:80/tcp"
    environment:
      TZ: 'America/Toronto'
    volumes:
      - './db:/var/www/html/db'
      - './logos:/var/www/html/images/uploads/logos'
    restart: unless-stopped
```

### Step 1 — Drop the host port publish

`- "8282:80/tcp"` publishes the app on the host, bypassing Coolify's proxy. Remove it; let the assigned domain route to container port 80.

### Step 2 — Convert relative bind mounts to named volumes

`./db` and `./logos` are relative to the compose dir → fragile inside Coolify. Switch to named volumes Coolify manages (and UUID-uniquifies).

### Step 3 — Add the service-template frontmatter (so it's a bona fide one-click)

```yaml
# documentation: https://github.com/ellite/wallos
# slogan: Open-source, self-hostable personal subscription tracker
# category: productivity
# tags: subscriptions,budget,finance,self-hosted
# port: 80
```

### Step 4 — Add the magic subdomain + expose editable vars

```yaml
environment:
  SERVICE_URL_WALLOS:                       # → http://wallos-<uuid>.example.com, routed to :80
  TZ: "${TZ:-Europe/Berlin}"
  PUID: "${PUID:-82}"
  PGID: "${PGID:-82}"
```

(`SERVICE_URL_WALLOS:` declared first-class so Coolify generates & shows it; `: -default` keeps `docker compose up` working standalone.)

### Step 5 — Add an explicit healthcheck (and note the Docker 25 caveat)

```yaml
healthcheck:
  test: ["CMD", "curl", "-fsS", "http://127.0.0.1/health.php"]
  interval: 2m
  timeout: 2s
  retries: 3
  start_period: 20s
```

### Final Wallos compose

```yaml
# documentation: https://github.com/ellite/wallos
# slogan: Open-source, self-hostable personal subscription tracker
# category: productivity
# tags: subscriptions,budget,finance,self-hosted
# port: 80

services:
  wallos:
    image: bellamy/wallos:latest
    restart: unless-stopped
    environment:
      # Coolify magic — auto subdomain for this service, shown in the env UI.
      SERVICE_URL_WALLOS:
      # Editable, with a safe default so the file still runs outside Coolify.
      TZ: "${TZ:-Europe/Berlin}"
      # php-fpm runs as www-data (UID/PGID 82); expose to match your host user.
      PUID: "${PUID:-82}"
      PGID: "${PGID:-82}"
      # ---- Optional OIDC (enable via Coolify env vars if you use SSO) ----
      # OIDC_ENABLED: "true"
      # OIDC_PROVIDER_NAME: "Authelia"
      # OIDC_CLIENT_ID: "wallos"
      # OIDC_CLIENT_SECRET: "replace-me"
      # OIDC_ISSUER: "https://auth.example.com"
      # OIDC_REDIRECT_URL: "https://wallos.example.com/index.php"
      # OIDC_LOGOUT_URL: "https://auth.example.com/logout"
    # No host port publish — Traefik routes container :80 via the assigned domain.
    volumes:
      - wallos_db:/var/www/html/db
      - wallos_logos:/var/www/html/images/uploads/logos
    healthcheck:
      test: ["CMD", "curl", "-fsS", "http://127.0.0.1/health.php"]
      interval: 2m
      timeout: 2s
      retries: 3
      start_period: 20s

volumes:
  wallos_db:
  wallos_logos:
```

### First-run note (why Wallos is genuinely one-click)

The README tells you to rename `db/wallos.empty.db` → `db/wallos.db`. **You don't need to** — the container's `startup.sh` runs `createdatabase.php` (which creates `/var/www/html/db/wallos.db` if absent and writes the schema) then `migrate.php`, then sets ownership on `db/` and `logos/`. So on a fresh named volume, Wallos initializes itself. That's true one-click parity.

---

## Part 10 — Deploying your converted project (both paths)

### Path A — as a Coolify Service (true one-click, mirrors built-in templates)

1. **Project → + New Resource → Service → Docker Compose Empty** (or, if you forked it and it's ≥1000 stars / you're adding it to Coolify, use the template directory).
2. **Paste your converted compose file** (the one from Part 9). Coolify validates it.
3. On the **Service Stack** page, Coolify lists every service + its detected ports. **Assign a domain** per service you want public (or leave internal services domain-less).
4. **Deploy.**

This is literally how Coolify's own one-click templates run — the "Docker Compose Empty" path mimics it ("mimics the one-click service deployment", per the docs).

### Path B — as a Git-backed Application with the Docker Compose build pack

Use this if you forked the repo and want CI/CD (auto-deploy on push, preview deploys, PR URLs):

1. **+ New Resource → Application**, pick your repo (**Public Repository** for public, **Github App / Deploy Key** for private).
2. **Build Pack → Docker Compose.**
3. Set **Base Directory** (`/` or the subfolder) and **Docker Compose Location** (must match the filename extension exactly; path is joined to Base Directory).
4. Branch auto-detected. You get **auto-deploy** on push, **preview deployments** per PR, rollback, and commit-linked deploys.
5. Coolify honors magic vars from Git sources — but only on **`v4.0.0-beta.411+`** (older: pasted Service templates only).

> 💡 **Prefer the Service path for a converted "peer project"** (like your Wallos). It keeps the source untouched, no forks, no merge-backlog — and it's exactly how a one-click app behaves. Use the Git path when you want to own the deployment pipeline. (Cross-reference: the *source-fork vs deployment-overlay* decision in the `selfhosted-stack-coexistence` skill.)

### After deploy — where to look

- **Health** tab: container status + healthcheck result. If unhealthy, fix the healthcheck before worrying about routing.
- **Domains** tab: confirm the FQDN + port mapping. If a service listens on non-80, the domain must carry that port.
- **Logs** tab: streaming logs (and Send to Axiom/New Relic if you want centralized logging).
- **Environment**: the generated `SERVICE_*` secrets are visible/editable here (security aside, that's where you pin a value if ever needed).

---

## Part 11 — Making it a *first-class* one-click Coolify Service (public template)

If you want your converted project to appear in Coolify's built-in **Services** catalog, the exact recipe is in [`get-started/contribute/service`](coolify-docs/raw/get-started_contribute_service.txt):

1. **Metadata frontmatter** at the top of the compose — `documentation`, `slogan`, `category` (one word), `tags` (comma-sep), `logo` (path, must match service name), and **`port`** (the main entrypoint port — **required**; "Caddy Proxy cannot automatically determine the service's port").
2. **Use the magic variables** everywhere (as above). Sprinkle `:?`-required and `:-`-default vars for a good user experience.
3. **Ship storage** via named volumes / `is_directory` / `content:`.
4. **Test via "Docker Compose Empty"** — it mimics one-click deployment.
5. **PR** the compose under `coollabsio/coolify` `templates/compose/`, add the SVG logo, plus a docs page.
6. **≥1000 ⭐** — the project must have at least 1,000 GitHub stars to be accepted as a one-click service. Below that, keep it as a self-hosted Service (Path A) you can paste anywhere.

---

## Part 12 — Debugging & hardening checklist

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| `404 Not Found` / `No available server` on the URL | Healthcheck failing (Traefik won't route to unhealthy) | Fix the compose `healthcheck:`; check the Health tab. |
| Intermittent `504`/hangs that come and go | **Custom `networks:` in the compose** | Remove the `networks:` block entirely. |
| App reaches DB too early, crash-looping | No `depends_on: condition: service_healthy` on the DB | Add it + a DB healthcheck. |
| Service "unreachable" / not on the expected port | Wrong container port in the domain | Domain port = **container** port, not host. |
| Secret value with `$` turns into a broken value | Env interpolation of a symbol password | Use no-symbol magic vars, or set **Literal**, or escape `$`→`$$`. |
| Persist data lost on redeploy | Relative bind mount (`./db`) in a compose without a stable working dir | Convert to a **named volume** Coolify can manage across deploys. |
| Container can't write its volume | Non-root UID vs root-owned mount | Set `PUID/PGID` (Wallos), or `chown` the bind-mount dir to the container UID. |
| Domain 404s but cert issued | Healthcheck placed on internal-only path | Use the real `/health` or `/health.php`; not a path only reachable behind the proxy. |
| First run does nothing / empty DB | DB never initialized | Confirm startup runs the init script (Wallos auto-creates; others need a `content:` init file). |
| Let's Encrypt not issuing | HTTP challenge can't reach port 80 / wildcard needed | Switch to the **DNS challenge**; or use a **wildcard cert**. |
| Magic vars not generated | Coolify older than `v4.0.0-beta.411` (Git-compose) | Upgrade, or deploy as a pasted Service. |
| One service exits 0 (migrations) makes stack "unhealthy" | One-shot service holds healthcheck | Add `exclude_from_hc: true`. |

### Readiness checklist before you call it "parity"
- [ ] No host port publish on any proxied frontend.
- [ ] No custom `networks:` block.
- [ ] All secrets are Coolify-generated (`SERVICE_*`) or env-referenced — nothing real in the repo.
- [ ] Every service needed for routing has a compose `healthcheck:` (or a Dockerfile `HEALTHCHECK`).
- [ ] DB is health-gated via `depends_on: condition: service_healthy` (if any).
- [ ] Persistent data uses **named volumes** (or `is_directory` bind mounts), not repo-relative paths.
- [ ] Outside Coolify, `docker compose --env-file .env up` still works (every var has a `:-` fallback).
- [ ] Assigning a domain (with the right container port) yields a working, HTTPS, healthchecked URL — no manual edits.

---

## Appendix A — How the file stays portable

Every `SERVICE_*` reference is paired with a `:-` fallback (`${SERVICE_PASSWORD_64_DB:-${POSTGRES_PASSWORD:-changeme123}}`). Outside Coolify those variables are empty strings, so the fallback chain kicks in. This means your *converted* compose is **strictly better** than the upstream one: it runs identically via `docker compose up` (or any other Compose host) **and** achieves full Coolify magic when Coolify is present. That's the whole game in a sentence.

## Appendix B — Full magic variable reference (quick table)

| `SERVICE_<ID>` | URL | FQDN | password | secret |
|---|---|---|---|---|
| URL / FQDN | `SERVICE_URL_<ID>` | `SERVICE_FQDN_<ID>` | — | — |
| + port | `SERVICE_URL_<ID>_<PORT>` | `SERVICE_FQDN_<ID>_<PORT>` | — | — |
| + path | `SERVICE_URL_<ID>=<path>` | `SERVICE_FQDN_<ID>=<path>` | — | — |
| port + path | `SERVICE_URL_<ID>_<PORT>=<path>` | `SERVICE_FQDN_<ID>_<PORT>=<path>` | — | — |
| user | — | — | `SERVICE_USER_<ID>` | — |
| password (no symbols) | — | — | `SERVICE_PASSWORD_<ID>` | `SERVICE_PASSWORD_64_<ID>` |
| password (symbols) | — | — | `SERVICE_PASSWORDWITHSYMBOLS_<ID>` | `…_64_<ID>` |
| random string | — | — | — | `SERVICE_BASE64[_\*]_<ID>` |
| base64-encoded | — | — | — | `SERVICE_REALBASE64[_\*]_<ID>` |
| hex | — | — | — | `SERVICE_HEX_<N>_<ID>` |

`<ID>` = uppercase service name; use **hyphens** (not underscores) in the name if you need `_<PORT>` routing.

---

*Sources: live Coolify docs at `https://coolify.io/docs` (raw text corpus: `./coolify-docs/raw/`), AdventureLog `main` project, Wallos `main` repo (compose, Dockerfile, `health.php`, `startup.sh`, `createdatabase.php`).*
