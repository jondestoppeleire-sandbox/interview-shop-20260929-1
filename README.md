# Shop

Shop is an online store at <https://shop.interview.tubi.io>.

```
browser -> Route53 -> ALB -> web -> api -> payments
                                    |  -> recs
                                    +-> PostgreSQL (RDS, shop-db)
worker -> PostgreSQL, receipts (file server)
```

Everything runs in the EKS cluster `sre-interview-lab` (namespace `shop`), region
`us-east-2`, except the database, which is RDS.

## Repo layout

| Path | What it is | How it ships |
|---|---|---|
| `k8s/shop/` | Kubernetes manifests (kustomize) | `kubectl apply -k k8s/shop` |
| `terraform/shop/` | RDS `shop-db`, its security group, its parameter group, and its subnet group | `terraform apply` |
| `.github/workflows/deploy.yml` | Deploys `main`: Terraform first, then the manifests | Every push to `main` |

**Deploys:** fork this repo, push a branch to your fork, and open a pull request.
The interviewer reviews and merges it, and the `deploy` workflow applies `main`
within about two minutes. A new commit on the pull request needs a new review. Pull
requests run no workflow. At the end of the session, delete your fork.

To preview a Terraform change with your own credentials:

```
cd terraform/shop
terraform init
terraform plan
```

## Services

Every service is the same image (`shop`), started in a different mode. Every HTTP
service listens on port 8080 and serves:

- `GET /healthz`: 200 while the process runs.
- `GET /healthz/ready`: 200 when the service is ready for traffic (see each service).
- `GET /metrics`: Prometheus metrics.

Logs are JSON lines with `level`, `msg`, and, on failures, `error`.

| Service | Kind | What it does | Endpoints | Ready when |
|---|---|---|---|---|
| `web` | Deployment | The storefront; the ALB sends every request here | `GET /`, `GET /product/{id}`, `POST /cart`, `POST /checkout`, `GET /receipt/{order}` | The process runs |
| `api` | Deployment + HPA | Products, carts, and checkouts | `GET /products/{id}`, `GET /products/{id}/recommendations`, `POST /cart`, `POST /checkout` | Its product cache is warm and its database pool is open |
| `payments` | Deployment | Payment provider | `POST /charge` | The process runs |
| `recs` | Deployment | Product recommendations | `GET /recommendations?product={id}` | The process runs |
| `worker` | Deployment | Makes a receipt for each new order: renders it into its spool directory, then uploads it to `receipts` | — | Its database pool is open |
| `receipts` | StatefulSet (nginx) | Receipt file server: `PUT` stores a file, `GET` returns it | `PUT/GET /receipts/{order}.txt`, `GET /healthz` | nginx runs |
| `cart-cleanup` | CronJob, every 5 min | Deletes abandoned carts | — | — |

## Settings

Environment variables, with their defaults.

**Database** (`api`, `worker`, `cart-cleanup`). `DB_HOST`, `DB_PASSWORD`, the
other connection values, and `DB_POOL_SIZE` come from the Secret `shop-db`.

| Setting | Default | Meaning |
|---|---|---|
| `DB_HOST`, `DB_PORT` | —, `5432` | Database address |
| `DB_NAME`, `DB_USER`, `DB_PASSWORD` | `shop`, `shop`, — | Database and credentials |
| `DB_SSLMODE` | `prefer` | libpq SSL mode |
| `DB_POOL_SIZE` | `14` | Connections per process, all opened at startup |
| `DB_POOL_HEALTHCHECK` | `false` | Check a connection before use, and replace a failed one |
| `DB_CONN_MAX_LIFETIME` | `0` (no limit) | Replace connections older than this, for example `5m` |
| `DB_QUERY_TIMEOUT` | `2s` | Time limit for one query, including the wait for a free connection |
| `DB_CONNECT_TIMEOUT` | `5s` | Time limit for opening one connection |

**api**

| Setting | Default | Meaning |
|---|---|---|
| `CACHE_MAX_MB` | `0` (no cap) | Size cap of the in-memory product page cache |
| `WARMUP_PRODUCTS` | `2000` | Products loaded into the cache at startup |
| `WARMUP_RATE` | `100` | Products loaded per second during the warm-up |
| `PAYMENTS_URL` | `http://payments:8080` | payments address |
| `RECS_URL` | `http://recs:8080` | recs address |
| `RECS_TIMEOUT` | `200ms` | Time limit for a recommendations call; recommendations are optional |

**web:** `API_URL` (`http://api:8080`), `RECEIPTS_URL` (`http://receipts:8080`).

**payments:** `DECLINE_RATE` (`0.02`): share of charges the provider declines.

**worker**

| Setting | Default | Meaning |
|---|---|---|
| `RECEIPTS_URL` | `http://receipts:8080` | receipts address |
| `SPOOL_DIR` | `/spool` | Where receipts are rendered before upload |
| `WORKER_BATCH` | `20` | Orders claimed per round |
| `WORKER_INTERVAL` | `1s` | Time between rounds |

**All services:** `PORT` (`8080`), `LOG_LEVEL` (`info`).

## Metrics

| Family | Metrics | From |
|---|---|---|
| HTTP server | `http_requests_total`, `http_request_duration_seconds`, `http_requests_in_flight` (labels `route`, `code`) | web, api, payments, recs |
| HTTP client | `http_client_requests_total`, `http_client_request_duration_seconds` (labels `target`, `code`) | web, api, worker |
| Database | `db_pool_connections` (`state`), `db_pool_max`, `db_pool_wait_seconds`, `db_query_duration_seconds` (`query`), `db_errors_total` (`error`: a SQLSTATE code, `timeout`, or `connection_lost`) | api, worker |
| Cache | `shop_cache_entries`, `shop_cache_hits_total`, `shop_cache_misses_total` | api |
| Business | `shop_product_views_total`, `shop_cart_adds_total`, `shop_checkouts_total` (`result`), `shop_orders_created_total` | api |
| Orders and receipts | `shop_orders_pending`, `shop_worker_jobs_total`, `shop_worker_job_duration_seconds`, `shop_receipt_uploads_total` (`result`) | worker |
| nginx | `nginx_up`, `nginx_http_requests_total`, `nginx_connections_active` | receipts |
| Runtime | `process_*`, `go_*`, `shop_build_info` | every service |

## Observability

Grafana and Prometheus run in namespace `monitoring`:

```
kubectl -n monitoring port-forward svc/grafana 3000:80            # http://localhost:3000
kubectl -n monitoring port-forward svc/kps-prometheus 9090:9090   # http://localhost:9090
```

Grafana has the `Shop` dashboards and Explore. Its CloudWatch data source has the
ALB and RDS metrics.
