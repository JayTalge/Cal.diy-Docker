# Cal.diy-Docker

Docker image for [calcom/cal.diy](https://github.com/calcom/cal.diy) (community fork of Cal.com), built with the upstream Dockerfile. This repo contains only the build workflow and optional patches in `patches/`. The upstream code is not forked.

| Image | Tags |
|---|---|
| `ghcr.io/jaytalge/cal.diy` | `main-<upstream sha7>`, `latest` |

## Build

- cal.diy has no regular release tags any more (last tag v6.2.0, 2026-03). Fixes land on `main`, so this repo builds **main HEAD**.
- Every day at 02:17 UTC the workflow resolves upstream `main`. It builds only if `main-<sha7>` is not in GHCR yet.
- `patches/*.patch` are applied with `git apply` before the build. A push to `patches/` or the workflow rebuilds the current upstream commit (same tag is overwritten).
- Before pushing, the image starts against a throwaway Postgres (migrations + app-store seed) and `/auth/login` has to return 200.
- Manual build: Actions → "Build image from upstream main" → Run workflow (optional commit/branch/tag, "force").

Baked-in build args (generic, not site-specific): `NEXT_PUBLIC_LICENSE_CONSENT=agree`, `CALCOM_TELEMETRY_DISABLED=1`, `ORGANIZATIONS_ENABLED=false`, no `NEXT_PUBLIC_API_V2_URL` (API v2 not included).

## Usage

```yaml
services:
  calcom:
    image: ghcr.io/jaytalge/cal.diy:latest
    restart: unless-stopped
    env_file: .env   # NEXT_PUBLIC_WEBAPP_URL, NEXTAUTH_URL, DATABASE_*, NEXTAUTH_SECRET, CALENDSO_ENCRYPTION_KEY, EMAIL_*
    ports:
      - 3000:3000
    depends_on: [db]
  db:
    image: postgres:16-alpine
    environment: { POSTGRES_USER: calcom, POSTGRES_PASSWORD: change-me, POSTGRES_DB: calcom }
    volumes: [db:/var/lib/postgresql/data]
volumes:
  db:
```

## Runtime notes

- The public URL is not baked in: `scripts/start.sh` replaces the build URL with `NEXT_PUBLIC_WEBAPP_URL` on every start.
- Every start runs `prisma migrate deploy`, so an automatic update (e.g. Watchtower) migrates the DB. Back up Postgres before major jumps.
- `/auth/*` and `/signup` send `X-Frame-Options: DENY`; booking pages can be embedded in an iframe.
