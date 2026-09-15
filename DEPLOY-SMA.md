# GoatCounter - Deployment, Smoke, and Mutation Notes

## Quick Start

```bash
./tester-env deploy    # Build image, start container, create site + admin user
./tester-env seed      # Populate with deterministic seed data
./tester-env verify    # Check seeded state (landing, login, seeded totals)
/tester-env reset     # Clean slate (stop + remove container + volume)
```

## CLI Commands

| Command | Description |
|---------|-------------|
| `deploy` | Build Docker image, start container, create site + admin user |
| `seed` | Populate app with deterministic seed data (23 pageviews, viewer user, API token) |
| `verify` | Assert landing renders, login works, 23 pageviews with expected per-path counts |
| `reset` | Stop container, remove container and data volume |
| `stop` | Stop container (preserves data) |
| `logs` | Show container logs |
| `status` | Show container status |

Options:
- `--run-id <id>` isolates container and volume names for parallel scenario runs.
- `--port <port>` binds the app to a specific host port (default: 18181).

## Build

```bash
./tester-env deploy
```

Prerequisites:
- `docker`

The build process:
1. `go mod download` (cached dependency layer; invalidated only on go.mod/go.sum change)
2. `go build ./cmd/goatcounter` (CGO sqlite build; small source mutations only re-run this layer)
3. Multi-stage Alpine runtime image running `goatcounter serve -automigrate -listen :8080 -tls http`

- Port: `18181`
- URL: `http://localhost:18181`
- Vhost/site: `goatcounter.localhost` (single-site install; any Host header falls back to this site)

## Credentials

- Admin: `admin@tester.local` / `TesterEnv123!`
- Viewer (read-only): `viewer@tester.local` / `ViewerEnv123!`

Site + admin user are created idempotently by `deploy` via `goatcounter db create site`.

## Seed Data

```bash
./tester-env seed
```

Seeded entities (deterministic, dates relative to seed time so the week-view dashboard stays populated):
- 23 pageviews across 11 sessions and 6 paths (`/`, `/pricing`, `/contact`, `/blog/launch`, `/docs/getting-started`, `/docs/api`); theme: docs/product site for fictional SaaS "Acme Forms"
- Per-path counts: `/`=4, `/pricing`=4, `/contact`=3, `/blog/launch`=4, `/docs/getting-started`=5, `/docs/api`=3
- 5 distinct referrers, >=3 browser families, >=7 countries
- Read-only viewer user + `tester-env-seed` (count perm) API token (config-element fixtures)

Seed is idempotent on non-empty analytics; run `./tester-env reset && ./tester-env deploy && ./tester-env seed` for a clean reseed.

## Reset

```bash
./tester-env reset
```

Then run `deploy` and `seed` to start fresh.

## Browser Verification

```bash
./tester-env verify
```

Checks:
- Landing page renders and mentions GoatCounter
- Admin login via `POST /user/requestlogin` yields an authenticated session (dashboard no longer shows login form)
- `hit_counts` totals 23 pageviews with expected per-path counts
- Dashboard HTML renders seeded paths server-side (`/docs/getting-started`, `/blog/launch`, `/pricing`)

Browser smoke (evidence-backed): dashboard shows 23 visits and 6 seeded paths; settings toggle round-trip verified.

## Mutation Smoke

Edit any Go source (e.g. a template string in `tpl/`), then `docker build -f Dockerfile.tester-env`. The change is visible in the served HTML after `deploy`.

## Baseline

Pinned SHA: `7e91d8a9bdbb0dd48496e498c5680f8f3477a1b4` (tag `v2.7.0`)
