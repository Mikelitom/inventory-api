# 08 — Setup & Development

&gt; Environment and workflow guide for contributors.
&gt; Status: 📝 Draft | Last revised: 2026-10-23
&gt; Verified on: CachyOS (Arch) and Ubuntu 24.04.

---

## 1. Prerequisites

| Tool | Version | Purpose |
|---|---|---|
| [mise](https://mise.jdx.dev) | latest | Node version manager (alternative: nvm) |
| Docker + Compose plugin | ≥ 24 | PostgreSQL container |
| git | ≥ 2.40 | version control |

Install per OS:

```bash
# CachyOS / Arch
sudo pacman -S mise docker docker-compose-plugin

# Ubuntu 24.04
sudo apt install mise docker.io docker-compose-v2
sudo usermod -aG docker $USER   # re-login afterwards
```

## 2. First-time setup (5 minutes)

```bash
git clone https://github.com/&lt;you&gt;/inventory-api.git
cd inventory-api

# Node 20 via mise (reads .mise.toml)
mise install

# Environment file
cp .env.example .env
# Edit .env: nothing to change for local development — defaults work.

# Database
docker compose up -d db

# Install, migrate, seed
npm install
npx prisma migrate deploy
npx prisma db seed        # creates first ADMIN + demo data

# Run
npm run dev               # → http://localhost:3000  (Swagger at /api/docs)
```

Default seeded credentials (dev only, never in production):
`admin@inventory.dev` / `Admin1234!`

## 3. Environment variables

| Variable | Example | Notes |
|---|---|---|
| `NODE_ENV` | `development` | `development` \| `test` \| `production` |
| `PORT` | `3000` | |
| `DATABASE_URL` | `postgres://postgres:postgres@localhost:5432/inventory` | used by Prisma |
| `JWT_SECRET` | `openssl rand -hex 32` | required in prod; dev default is a loud warning |
| `JWT_ACCESS_TTL` | `15m` | |
| `JWT_REFRESH_TTL` | `30d` | |
| `CORS_ORIGINS` | `http://localhost:5173` | comma-separated; empty = same-origin |
| `LOG_LEVEL` | `info` | pino levels |

&gt; **Security:** `.env` is gitignored. `.env.example` is the template with empty
&gt; secrets. Never commit a real `.env`; if one is ever committed, rotate every
&gt; secret it contained — deletion from history is not enough (see doc 07
&gt; ADR-007 and README security notes).
&gt;
&gt; **Secrets beyond local development:** CI injects values from GitHub Actions
&gt; Secrets; production should source them from a secret manager (Vault,
&gt; Doppler, AWS Secrets Manager, …) that exports the same variable names. The
&gt; application reads only `process.env` and is agnostic to the provider.

## 4. Daily workflow

```bash
docker compose up -d db     # if not running
npm run dev                 # tsx watch
npm run test                # unit + integration (needs test DB up)
npm run test:coverage       # coverage report
npm run lint                # eslint
npm run typecheck           # tsc --noEmit
npx prisma studio           # optional DB GUI at :5555
```

Docker compose services:

```yaml
# docker-compose.yml (excerpt)
services:
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: inventory
    ports: ["5432:5432"]
    volumes: [pgdata:/var/lib/postgresql/data]

  db-test:                    # isolated test DB, never persisted
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: inventory_test
    ports: ["5433:5432"]
```

## 5. Testing

- Test suite requires `db-test` running (`docker compose up -d db-test`).
- Each integration test file gets a fresh schema: global setup runs
  `prisma migrate reset --force --skip-seed` against `inventory_test`.
- Integration tests boot the real Express app via supertest on an ephemeral
  port; unit tests mock repositories.

```bash
npm run test                 # everything
npm run test:unit            # vitest --exclude integration
npm run test:integration     # vitest run tests/integration
```

## 6. Branching & commits

- Long-lived branches: `main` (protected), `develop`.
- Short-lived branches per feature: `feat/auth`, `feat/stock-movements`,
  `docs/data-model`, merged via PR into `develop`; `develop` → `main` at
  each roadmap milestone (doc 09).
- Commit convention: Conventional Commits, in English.

```
feat(auth): add refresh token rotation
fix(movements): reject OUT exceeding available stock
docs(api-spec): document report endpoints
```

## 7. CI/CD (GitHub Actions)

Pipeline on every PR to `develop`/`main`:

1. **setup**: checkout, mise install (Node 20), `npm ci`
2. **db**: start `db-test` service container
3. **quality**: `typecheck` → `lint` → `test:coverage` (fails &lt; 70% on business
   modules)
4. **build**: `npm run build` (catches TS strict errors a dev server may hide)

Merge requires all checks green (branch protection).

## 8. Production deployment notes

The app ships as a stateless container; the only external dependency is
PostgreSQL. Minimal production checklist:

- [ ] `NODE_ENV=production`, real `JWT_SECRET`, `DATABASE_URL` from secret manager
- [ ] Migrations run as a release step (`prisma migrate deploy`) before traffic
- [ ] HTTPS terminated at a reverse proxy; `CORS_ORIGINS` set explicitly
- [ ] Logs shipped to an aggregator (pino JSON lines)
- [ ] Health check wired to the orchestrator (`GET /api/v1/health`)