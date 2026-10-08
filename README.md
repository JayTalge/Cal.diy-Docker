# Cal.diy-Docker

Docker image for [calcom/cal.diy](https://github.com/calcom/cal.diy) (community fork of Cal.com), built with the upstream Dockerfile. This repo contains only the build workflow and optional patches in `patches/`. The upstream code is not forked.

| Image | Tags |
|---|---|
| `ghcr.io/jaytalge/cal.diy` | web app, `main-<upstream sha7>`, `latest` |
| `ghcr.io/jaytalge/cal.diy-api` | API v2 (NestJS, `apps/api/v2`), same tags |

## Build

- cal.diy has no regular release tags any more (last tag v6.2.0, 2026-03). Fixes land on `main`, so this repo builds **main HEAD**.
- Every day at 02:17 UTC the workflow resolves upstream `main`. It builds only if `main-<sha7>` is not in GHCR yet.
- `patches/*.patch` are applied with `git apply` before the build. A push to `patches/` or the workflow rebuilds the current upstream commit (same tag is overwritten).
- Before pushing, the image starts against a throwaway Postgres (migrations + app-store seed) and the first-admin page `/auth/setup` has to return 200.
- Manual build: Actions → "Build image from upstream main" → Run workflow (optional commit/branch/tag, "force").

Baked-in build args (generic, not site-specific): `NEXT_PUBLIC_LICENSE_CONSENT=agree`, `CALCOM_TELEMETRY_DISABLED=1`, `ORGANIZATIONS_ENABLED=false`, `NEXT_PUBLIC_API_V2_URL=http://calcom-api:5555/v2`.

`NEXT_PUBLIC_API_V2_URL` is a Next.js rewrite target, not a public URL: the web app proxies `/api/v2/*` to it. Run the API image as container/alias `calcom-api` on port 5555 in the same network (env: `API_PORT=5555`, `API_URL`, `WEB_APP_URL`, `DATABASE_READ_URL`, `DATABASE_WRITE_URL`, `REDIS_URL`, `NEXTAUTH_SECRET`, `STRIPE_API_KEY`/`STRIPE_WEBHOOK_SECRET` may be empty). Without the API container only `/api/v2/*` fails, the rest of the app works.

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

## Patches

### 0001-generic-oidc-login

Adds a generic OpenID Connect login (cal.diy itself only has Google and Azure AD). Off unless configured:

| Env | Meaning |
|---|---|
| `OIDC_LOGIN_ENABLED=true` | enables the provider |
| `OIDC_ISSUER` | issuer URL, e.g. `https://auth.example.com/application/o/cal/` (Authentik) |
| `OIDC_CLIENT_ID`, `OIDC_CLIENT_SECRET` | confidential client |
| `OIDC_NAME` | button label on the login page (default `SSO`) |
| `OIDC_SCOPES` | default `openid email profile` |
| `OIDC_TRUST_EMAIL=true` | treat the IdP e-mail as verified even without `email_verified` (only if users cannot change their e-mail in the IdP) |

Redirect URI: `<NEXT_PUBLIC_WEBAPP_URL>/api/auth/callback/oidc`. Users are stored with identity provider `SAML` (no DB migration). A first SSO login creates the user; an existing password user with the same verified e-mail is switched to SSO.

## Runtime notes

- The public URL is not baked in: `scripts/start.sh` replaces the build URL with `NEXT_PUBLIC_WEBAPP_URL` on every start.
- Every start runs `prisma migrate deploy`, so an automatic update (e.g. Watchtower) migrates the DB. Back up Postgres before major jumps.
- `/auth/*` and `/signup` send `X-Frame-Options: DENY`; booking pages can be embedded in an iframe.
