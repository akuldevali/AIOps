# Make nginx the sole public entrypoint

**Project:** `projects/boutique-microservices`
**Summary:** Backend service ports `3001–3006` are still published to the host, bypassing the nginx reverse proxy. The frontend also hardcodes `http://localhost:3001` as the API base URL, so today's browser traffic skips nginx entirely.

**Status:** Not started. Deferred — captured here for future work.
**Captured:** 2026-05-24

---

## Why this matters

- **Defeats the purpose of the reverse proxy.** Any per-route policy added to nginx later — auth headers, rate limits, caching, TLS termination, request logging — can be trivially skipped by hitting the direct host ports.
- **Increases attack surface.** Every microservice is reachable from the host network, not just the gateway through one controlled entrypoint.
- **Hides routing bugs.** A request that "works" via `http://localhost:3001/...` may silently fail via `http://localhost/api/...` and no one notices in dev. We want both paths to converge on the nginx path.

---

## Current state

### `docker-compose.yml` — six services still publish ports directly

[projects/boutique-microservices/docker-compose.yml](../projects/boutique-microservices/docker-compose.yml) — `ports:` blocks:

| Service           | Lines      | Port |
|-------------------|------------|------|
| `gateway`         | [69-70](../projects/boutique-microservices/docker-compose.yml#L69-L70)   | 3001 |
| `auth`            | [92-93](../projects/boutique-microservices/docker-compose.yml#L92-L93)   | 3002 |
| `product-service` | [111-112](../projects/boutique-microservices/docker-compose.yml#L111-L112) | 3003 |
| `order-service`   | [130-131](../projects/boutique-microservices/docker-compose.yml#L130-L131) | 3004 |
| `orders`          | [155-156](../projects/boutique-microservices/docker-compose.yml#L155-L156) | 3005 |
| `user-service`    | [174-175](../projects/boutique-microservices/docker-compose.yml#L174-L175) | 3006 |

### Frontend hardcodes the direct gateway URL

- [frontend/src/services/api.ts:3](../projects/boutique-microservices/frontend/src/services/api.ts#L3)
  ```ts
  const API_BASE_URL = process.env.REACT_APP_API_URL || 'http://localhost:3001/api';
  ```
  `REACT_APP_API_URL` is **not** set anywhere (it was removed from `docker-compose.yml` during the nginx introduction), so the fallback is what every production build bakes in. The browser then calls `localhost:3001` directly — nginx is bypassed.

- [frontend/src/services/productService.ts:17](../projects/boutique-microservices/frontend/src/services/productService.ts#L17) — stale `console.log` mentioning `localhost:3003`. Cosmetic only (no real network call), but misleading. Worth cleaning up at the same time.

---

## Fix when ready

Three steps:

### 1. Frontend — make the API base URL relative

In [frontend/src/services/api.ts:3](../projects/boutique-microservices/frontend/src/services/api.ts#L3):

```diff
- const API_BASE_URL = process.env.REACT_APP_API_URL || 'http://localhost:3001/api';
+ const API_BASE_URL = process.env.REACT_APP_API_URL || '/api';
```

Relative `/api` means the browser hits whatever origin served the page (nginx on :80), and nginx already proxies `/api/` → `gateway:3001`.

(Optional) clean up the stale log in [productService.ts:17](../projects/boutique-microservices/frontend/src/services/productService.ts#L17).

### 2. Compose — swap `ports:` → `expose:` for the six backend services

For each service in the table above, replace:

```yaml
    ports:
      - "<port>:<port>"
```

with:

```yaml
    expose:
      - "<port>"
```

Inter-service traffic over the `boutique-network` bridge is unaffected — Compose service names (`gateway`, `auth`, etc.) resolve regardless of whether ports are `ports:` or `expose:`.

### 3. Rebuild

The API base URL is baked into the React bundle at build time, so the frontend image must be rebuilt:

```bash
docker compose build frontend
docker compose up -d
```

---

## Out of scope / deliberately left published

- `postgres:5432`, `prometheus:9090`, `grafana:3007` — dev/ops tooling, separate decision (close them if/when you want a fully sealed stack).
- `nginx:80` — the new public entrypoint, must stay published.

---

## Verification checklist (for whoever picks this up)

1. `docker compose down && docker compose up -d --build`
2. Open `http://localhost` — site loads.
3. Browser DevTools → Network: every API call URL starts with `http://localhost/api/...` (no `:3001`).
4. Full app flow works: login, browse products, add to cart, place order.
5. `curl http://localhost/api/products` → 200 (through nginx).
6. `curl http://localhost:3001/api/products` → connection refused (direct access closed).
7. `docker compose exec nginx wget -qO- http://gateway:3001/api/products` → 200 (internal network still works).
