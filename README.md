````markdown
# GameHub — Local Setup

A microservices-based gaming platform: register, log in (JWT), play games
(Snake, Tic-Tac-Toe, Rock-Paper-Scissors, Memory) and track scores on a dashboard.

## Architecture

| Service | Tech | Port |
|---|---|---|
| `frontend/` | React + Tailwind, served by nginx (also proxies `/api/*`) | 3000 → 80 |
| `auth-service/` | Flask + JWT (`/register`, `/login`), gunicorn | 8080 |
| `game-service/` | Flask (`/game/stats`, `/snake/score`, `/users/scores`), gunicorn | 8081 |
| `postgres` | PostgreSQL 13, schema from `init.sql` | 5432 |

```
Browser → nginx (:3000) ──/api/auth/*──→ auth-service:8080 ─┐
        └──/api/game/*──→ game-service:8081 ─┴──→ postgres:5432
```

## Prerequisites

- Docker Desktop (compose v2) — no Python/Node needed, everything runs in containers.

## Run

```powershell
docker compose up --build        # first run / after code changes
docker compose up -d             # subsequent runs
```

Open http://localhost:3000 → Sign up → Log in → Play.

```powershell
docker compose down              # stop
docker compose down -v           # stop + wipe DB (re-runs init.sql next start)
```

## How it works

- Postgres runs `init.sql` automatically on first boot (mounted into
  `/docker-entrypoint-initdb.d/`); data persists in the `pgdata` volume.
- Backends get DB config via env vars (`POSTGRES_HOST=postgres` — the compose
  service name, not `localhost`).
- `frontend/nginx.conf` proxies `/api/auth/` → `auth-service:8080` and
  `/api/game/` → `game-service:8081` (compose DNS names; the upstream
  `*.svc.cluster.local` names only work inside Kubernetes).
- Frontend port mapping must be `"3000:80"` — nginx listens on 80, not 3000.

## Verify

```powershell
docker compose ps                       # all Up, postgres (healthy)
curl.exe http://localhost:3000          # React HTML
curl.exe http://localhost:8080          # 404 = healthy (no root route)
curl.exe http://localhost:8081/users/scores
```

## Troubleshooting

| Symptom | Fix |
|---|---|
| `Container name already in use` | `docker rm -f gamehub-postgres auth-service game-service frontend` |
| nginx `host not found ...svc.cluster.local` | Wrong `nginx.conf` baked in — point upstreams to `auth-service`/`game-service` and `docker compose up --build frontend` |
| curl `(52) Empty reply` on :3000 | Port mapping must be `"3000:80"`, then `up -d --force-recreate frontend` |
| Dashboard not updating | DevTools → Network: check `POST /api/game/game/stats`; `docker logs game-service`. Dashboard refetches on mount/focus — refresh the page |
| `psycopg2-binary` build error (local pip) | Use Docker instead — containers build on Linux Python, no wheel issue |

## Kubernetes (optional)

Manifests deploy the same images on any cluster (Docker Desktop k8s, or cloud).
The gateway/cert-manager/ExternalDNS manifests are cloud-specific — locally use
`kubectl port-forward svc/frontend 3000:80 -n crash-course` instead.
````

---

## Notes

- The app details (React + Flask + Postgres, JWT auth, games list) come from `DEVELOPMENT.md` [1](#16-0) ; the original `README.md` you deleted only covered the Exoscale k8s deployment, so this new file reflects your local compose workflow instead. [2](#16-1) 
- Adjust the compose service names/ports in the README if your final `docker-compose.yml` differs (e.g., no `container_name`).
- If you later restore the k8s `manifests/` folder, expand the Kubernetes section with the full apply order from the original README. [3](#16-2) 

Wiki pages you might want to explore:
- [Overview (saiyam1814/Kubernetes-crash-course-2025)](https://github.com/saiyam1814/Kubernetes-crash-course-2025)

