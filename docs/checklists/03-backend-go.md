# Checklist 03 — Backend Go (modular monolith: `api | worker | migrate | import`)

> Dùng để AI check source `backend/`, bạn re-check `[M]`. Tham chiếu D01–D06, D15, D16, D20, D22, D23; `01-CONTRACTS.md` toàn bộ; số đo `03-MEASUREMENT-TOOLS.md` §3.

## 0. Biên giới: Backend nhận gì, phải trả gì

| Hướng | MUST | OPT | Nguồn |
|---|---|---|---|
| **Input** contract | `contracts/openapi.yaml` → `internal/gen/api/api.gen.go` (oapi-codegen strict server, commit); `fixtures/fsrs_cases.json` sinh **từ** backend (server là nguồn) | | Contracts §6 |
| **Input** DB | Schema từ `db/migrations/*.sql` (goose); query từ `db/queries/*.sql` (sqlc) → `internal/gen/db` | | Checklist 04 |
| **Input** env | §8 Contracts Backend; validate lúc boot bằng `envconfig`/tự viết, thiếu MUST → exit 1 với message rõ | | |
| **Input** ngoài | Apple JWKS (cache 24 h), RevenueCat webhook (Bearer secret), Resend API, R2 S3 API, Piper local | RevenueCat REST reconcile | Contracts §5 |
| **Output** HTTP | Đúng openapi (schemathesis 0 fail); Problem Details `application/problem+json` với `code`, `request_id`; header `X-Request-Id`, `Cache-Control`, `ETag`, `RateLimit-*`, `Retry-After`; cursor pagination HMAC | `Deprecation`/`Sunset` | Contracts §2 |
| **Output** vận hành | `/healthz`, `/readyz`, `/metrics` (nội bộ), slog JSON có `request_id, user_id, route, status, duration_ms`, OTel traces, Sentry | | |
| **Output** artifact | Docker image `ghcr.io/<org>/app:<sha>` multi-arch ≤ 25 MB, distroless, non-root; `app migrate up` chạy trước switch traffic | | Checklist 05 |
| **Output** evidence | `docs/evidence/backend/*` | | |

Quy tắc module (ép bằng `go vet` + `depguard` trong golangci): `internal/httpapi` → `internal/<domain>` → `internal/platform` + `internal/gen/db`. Domain package **không import** `net/http` và **không import** nhau (dùng interface nhỏ định nghĩa tại consumer). `internal/gen/*` không import gì của app.

## 1. Cấu trúc file kỳ vọng

```
backend/
├── cmd/app/main.go                    # cobra hoặc switch os.Args[1]: api|worker|migrate|import|fsrs-fixtures
├── internal/
│   ├── httpapi/{server.go (StrictServerInterface impl), middleware.go, problem.go, cursor.go, auth_ctx.go, routes_test.go}
│   ├── auth/{apple.go, otp.go, jwt.go, refresh.go, service.go, *_test.go}
│   ├── user/ dictionary/ deck/ study/{fsrs.go, queue.go, review.go} billing/ admin/ stats/
│   ├── platform/{config, db (pool.go, tx.go, testdb.go), log, otel, mail, storage (r2.go), jobs (river.go), ratelimit, clock, ids}
│   └── gen/{api/api.gen.go, db/*.go}
├── db/migrations/0001_init.sql …      db/queries/{users,decks,cards,study,billing,admin}.sql
├── loadtest/k6/{study-flow.js, dictionary-search.js, auth-refresh.js, lib/}
├── loadtest/pgbench/review_insert.sql
├── tools/import-dictionary/{main.go, wiktionary.go, tatoeba.go, freq.go, audio.go, report.go}
├── Makefile (gen, lint, test, test-integration, bench, build, docker, k6)
├── sqlc.yaml  oapi-codegen.yaml  .golangci.yml  Dockerfile  .dockerignore  .env.example  go.mod
```

## 2. A — Kiến trúc & đúng contract (A01–A18)

| ID | Mục | Target Bare | Target Perfect | Cách check | M |
|---|---|---|---|---|---|
| A01 | 1 binary, subcommands `api\|worker\|migrate\|import`; mỗi subcommand có `--help`; `api` không tự migrate | | `fsrs-fixtures` subcommand | `go run ./cmd/app --help` liệt kê 4 lệnh; grep `migrate` trong `api` path = 0 | |
| A02 | Config: struct duy nhất `platform/config.Config`, load từ env, validate MUST, in ra config **đã che secret** lúc boot | | Reload không restart (không cần) | `config_test.go` case thiếu `DATABASE_URL` → error; boot log không chứa giá trị secret | |
| A03 | Skeleton + middleware chain đúng thứ tự: `Recover → RequestID → RealIP (chỉ tin `CF-Connecting-IP`/`X-Forwarded-For` từ Caddy) → Logger → OTel → CORS → RateLimit → ClientVersion (426) → Auth → Handler`; graceful shutdown ≤ 10 s (`server.Shutdown` + River stop); `/healthz` process-only, `/readyz` = DB ping + goose version == head | | | `middleware.go` thứ tự; `TestGracefulShutdownDrainsInflight`; `curl /readyz` khi DB down → 503 + `Retry-After` | |
| A04 | Router stdlib `net/http` ≥ 1.22 pattern; handler sinh từ oapi-codegen `strict-server`; **không** viết handler tay ngoài generated interface (trừ `/healthz /readyz /metrics /debug/pprof`) | | | `grep -rn "HandleFunc(" internal/httpapi` chỉ 4 route hệ thống; `oapi-codegen.yaml` có `strict-server: true, std-http-server: true, embedded-spec: true` | |
| A05 | Problem Details: 1 hàm `problem.Write(w, r, err)` duy nhất; domain trả `*problem.Error{Code, Status, Detail, Errors}`; unknown error → 500 `internal` + Sentry, **không lộ** message gốc; `type` URL = `https://api.<domain>/errors/<code>` | | | `grep -rn "http.Error(" internal` = 0; `grep -rn "json.NewEncoder(w)" internal/httpapi` chỉ trong generated/problem.go; test 500 không chứa `err.Error()` | |
| A06 | Cursor pagination: `cursor.go` encode base64url(`created_at\|id`) + HMAC-SHA256 (key từ env); cursor sửa → 400; `limit` clamp 1..200 default 50; sort `(created_at DESC, id DESC)` với index tương ứng | | | `cursor_test.go` tamper case; sqlc query dùng `WHERE (created_at, id) < ($1, $2)` | |
| A07 | ID UUIDv7 sinh bằng `platform/ids` (`google/uuid.NewV7`); thời gian qua `platform/clock` interface (test được); mọi timestamp UTC RFC3339 ms | | | `grep -rn "time.Now()" internal --include=*.go \| grep -v _test \| grep -v platform/clock` = 0; `grep -rn "uuid.New()" internal` = 0 | |
| A08 | Auth: Apple `identity_token` verify (JWKS cache 24 h, ETag, fallback cache 7 ngày, check `aud=APPLE_BUNDLE_ID`, `nonce` = SHA256(raw_nonce)); Email OTP 6 số, hash, TTL 10 phút, 5 lần sai → khoá 15 phút, luôn 202; JWT EdDSA `kid`, TTL 15 phút, claims đúng §2.6; refresh opaque 32 B, lưu SHA-256, rotation, reuse → revoke family `sid` + trả `refresh_reused`; web → cookie `rt` đúng attribute; logout `all_devices` | | WebAuthn admin | `auth/*_test.go`: `TestRefreshReuseRevokesFamily`, `TestOTPLockoutAfter5`, `TestAppleNonceMismatch`, `TestWebRefreshSetsCookieNotBody`; JWKS `GET /.well-known/jwks.json` 2 key khi rotate | |
| A09 | Auth context: `auth_ctx.go` đưa `UserID, SessionID, Role` vào `context`; handler lấy qua `auth.From(ctx)`; endpoint admin check `Role == admin` ở middleware **và** ghi `admin_audit_logs` (actor, action, target, diff, ip, request_id) trong cùng tx | | | `TestAdminEndpointsWriteAudit` chạy mọi admin route; grep mỗi admin handler gọi `audit.Record` | |
| A10 | **Mọi query đụng dữ liệu user có `user_id`** (không RLS): sqlc query trong `db/queries/{decks,cards,study,billing,users}.sql` đọc/ghi bảng user-owned bắt buộc có tham số `user_id`/`owner_id` trong `WHERE`; test tĩnh `TestAllUserScopedQueriesHaveUserID` parse file `.sql`, danh sách bảng user-owned hardcode, fail nếu thiếu; ngoại lệ ghi rõ `-- allow-no-user-id: <lý do>` (chỉ admin/system) | | | Test pass; `grep -c "allow-no-user-id" db/queries/*.sql` ≤ 5 và mỗi cái có lý do | |
| A11 | Idempotency: middleware cho 3 POST theo Contracts: key + user + route → lưu response hash 24 h (bảng `idempotency_keys`); cùng key khác body → 422 `idempotency_mismatch`; `POST /study/reviews` thêm dedupe theo `client_review_id` unique `(user_id, client_review_id)` → `status: duplicate` | | | `TestIdempotentCreateDeckTwiceSameKey` → 1 row; `TestReviewDuplicateClientID` → `duplicate`, `review_logs` count không đổi | |
| A12 | Study: FSRS `go-fsrs` (pinned version), `fsrs_version` trong `/v1/version`; client `fsrs_version` major khác → 409; batch ≤ 100 (413 khi hơn); mỗi review: lock `card_states` row `FOR UPDATE`, apply, insert `review_logs` (append-only, không UPDATE/DELETE), trả state; cả batch trong 1 tx nhưng lỗi 1 review → `rejected` riêng, không rollback cả batch (savepoint) | Undo ≤ 5 phút | Optimizer job | `TestFSRSFixturesParity` 100% `fsrs_cases.json`; `TestBatchPartialRejectKeepsOthers`; migration `review_logs` có `REVOKE UPDATE, DELETE` cho `app_rw` | |
| A13 | Queue: `GET /study/queue` 1 query (không N+1): due cards + new cards theo `daily_goal` + `intervals_preview` tính trong Go; `X-Timezone` quyết định "hôm nay" | | | `TestQueueSingleQuery` đếm query qua pgx tracer = 1 (hoặc ≤ 2); EXPLAIN evidence D2 | |
| A14 | Deck/Card: ownership check trong query (`WHERE id=$1 AND owner_id=$2`) không phải chỉ trong Go; optimistic `version` → 409 `conflict`; public/system deck đọc không cần auth; `clone` copy trong 1 tx | | | `TestPatchOtherUsersDeck403`; `TestVersionMismatch409` | |
| A15 | Rate limit: `platform/ratelimit` token bucket in-memory (1 instance) key theo IP (`/auth/*` 10/phút), user (`/v1/*` 600/phút), OTP request per email 3/10 phút; header `RateLimit-Limit/Remaining/Reset` mọi response; 429 Problem `rate_limited` + `Retry-After` | Redis/Postgres khi > 1 replica | | `TestRateLimitHeaders`; k6 `auth-refresh.js` thấy 429 đúng format | |
| A16 | Billing: RevenueCat webhook verify Bearer secret (constant-time), lưu raw event vào `billing_events` (unique `event_id`) **rồi mới** 200, xử lý bằng River job → cập nhật `entitlements`; replay → no-op; middleware `entitlement_required` đọc DB (không tin claim `ent`) | Promo | Reconcile job hàng ngày | `TestWebhookReplayIdempotent`; `TestEntitlementCheckReadsDB` | |
| A17 | Jobs (River): `otp_email`, `hard_delete_user` (30 ngày), `stats_rollup_daily` (per timezone), `billing_event_apply`, `audio_render`; enqueue trong cùng tx với ghi dữ liệu; retry policy + max attempts; `worker` subcommand riêng | | Periodic jobs UI admin | `grep -rn "river.Client.*InsertTx" internal` ≥ 4; `TestStatsRollupTimezone` | |
| A18 | Dictionary: search `pg_trgm` + prefix, `limit ≤ 50`, response `Cache-Control: public, max-age=3600, stale-while-revalidate=86400`, `ETag` trên entry; không auth bắt buộc | Word of day | | `curl -I /v1/dictionary/entries/<id>` có ETag; `If-None-Match` → 304 test | |

## 3. S — Bảo mật (S01–S10)

| ID | Mục | Target Bare | Target Perfect | Cách check | M |
|---|---|---|---|---|---|
| S01 | Input: mọi body decode qua generated types + `openapi` validation middleware (`oapi-codegen` `nethttp-middleware` với `kin-openapi`) → 422 trước handler; `MaxBytesReader` 1 MB (reviews 256 KB) | | | `TestUnknownFieldOrOversizeBody`; middleware trong chain | |
| S02 | SQL chỉ qua sqlc (parameterized); `grep -rn "fmt.Sprintf(.*SELECT\|Exec(ctx, \"" internal` = 0 ngoài migrations tooling | | | Output grep; golangci `sqlclosecheck`, `rowserrcheck` | |
| S03 | Secret: không log token/OTP/email đầy đủ (mask), `slog` `ReplaceAttr` redact keys `authorization, refresh_token, otp, password`; Sentry `BeforeSend` scrub | | | `TestLogRedaction`; grep `slog.*token` | |
| S04 | CORS: allowlist `CORS_ALLOWED_ORIGINS` chính xác (không `*` khi credentials), `Access-Control-Allow-Credentials: true` chỉ cho web/admin origin, preflight cache 600 s | | | `TestCORSRejectsUnknownOrigin` | |
| S05 | Cookie refresh web: `HttpOnly; Secure; SameSite=Strict; Domain=api.<domain>; Path=/v1/auth/refresh; Max-Age=30d`; endpoint refresh yêu cầu `X-Client-Platform` header + `Origin` ∈ allowlist (chống CSRF) | | | `TestRefreshWithoutPlatformHeader403` | |
| S06 | Headers response API: `X-Content-Type-Options: nosniff`, `Cache-Control: no-store` mặc định, `Strict-Transport-Security` (Caddy cũng set, 1 nơi — chọn Caddy, backend không set HSTS) | | | `curl -I /v1/me` | |
| S07 | `govulncheck` 0; `golangci-lint` với `gosec` 0; dependency ≤ 25 direct trong `go.mod`; `go mod verify` ok; renovate | | SBOM (`syft`) + cosign sign image | B6 evidence; `grep -c "^\t" go.mod` (require block) | |
| S08 | Delete account: soft delete (`deleted_at`) → revoke mọi refresh → job hard delete 30 ngày xoá user + decks + states + logs (review_logs của user anonymize hoặc xoá theo privacy policy); export OPT | | Data export self-serve | `TestDeleteMeRevokesAllSessions`; job test | |
| S09 | pprof + `/metrics` chỉ bind `127.0.0.1:6060`/docker network, không qua Caddy | | | `grep -rn "6060" cmd internal` bind loopback; nmap ngoài không thấy (05) | |
| S10 | Timeout: `http.Server{ReadHeaderTimeout: 5s, ReadTimeout: 15s, WriteTimeout: 30s, IdleTimeout: 60s}`; outbound client timeout 10 s; `statement_timeout=5s` trên pool | | | grep struct `http.Server`; pool config | |

## 4. D — Data layer (D01–D06) (chi tiết schema ở checklist 04)

| ID | Mục | Target Bare | Target Perfect | Cách check | M |
|---|---|---|---|---|---|
| D01 | pgx v5 pool: `MaxConns=10`, `MinConns=2`, `MaxConnLifetime=30m`, `MaxConnIdleTime=5m`, `HealthCheckPeriod=30s`; `default_query_exec_mode=cache_describe` (Neon pooler); TLS `verify-full` | | | `pool.go`; `TestPoolConfigFromURL` | |
| D02 | Transaction helper `db.WithTx(ctx, fn)` rollback khi panic/err; không nested tx; mọi multi-write dùng nó | | | `grep -rn "pool.Begin(" internal \| grep -v platform/db` = 0 | |
| D03 | sqlc: `emit_pointers_for_null_types`, `emit_json_tags` snake_case, `query_parameter_limit: 1` (struct params); không `SELECT *`; `sqlc vet` với `EXPLAIN` rule chặn seq scan | | | `sqlc.yaml`; `grep -rn "SELECT \*" db/queries` = 0; `sqlc vet` exit 0 | |
| D04 | Migration: goose embed (`embed.FS`), naming `NNNN_<slug>.sql` với `-- +goose Up/Down`; `app migrate up` dùng `DATABASE_URL_DIRECT`, lock advisory; expand/contract; `squawk` 0 error | | | `TestMigrateUpDownUp` trên testcontainers | |
| D05 | Integration test: testcontainers Postgres 17 (+ `pg_trgm`), mỗi test 1 schema hoặc truncate; chạy `make test-integration` < 3 phút | Neon branch per PR | | CI log thời gian | |
| D06 | `/readyz` check goose version == embedded head; lệch → 503 `migration_pending` | | | `TestReadyzMigrationPending` | |

## 5. O — Observability (O01–O05)

| ID | Mục | Target Bare | Target Perfect | Cách check | M |
|---|---|---|---|---|---|
| O01 | OTel SDK: traces qua OTLP → Grafana Cloud Tempo; span cho HTTP (otelhttp), pgx (otelpgx), River job, outbound (Resend/R2/Apple); `traceparent` propagate; sample 20% prod, 100% staging | 100% + tail sampling | | Screenshot 1 trace `/v1/study/reviews` có span `pgx.query` ≥ 1; B7 | |
| O02 | Prometheus `/metrics` (`promhttp`): `http_request_duration_seconds` histogram (route, method, status), `http_requests_total`, `db_pool_*`, `river_jobs_*`, `go_*`; route label dùng pattern (không id) | RED dashboard | | `curl 127.0.0.1:6060/metrics \| grep http_request_duration_seconds_bucket`; label cardinality ≤ 200 series | |
| O03 | Sentry Go: `Recover` middleware gửi panic + 5xx với `request_id`, `user_id` (không email); release = git sha; env tag | | | Sentry issue mẫu có tags | |
| O04 | slog JSON: mỗi request 1 dòng `level, ts, msg, request_id, user_id, method, route, status, duration_ms, bytes, ip, ua`; lỗi có `err`, `code`; level `DEBUG` chỉ dev | | Log → Loki | Log mẫu dán vào evidence; `jq` parse được 100% dòng | |
| O05 | Alert rules (`infra/grafana/alerts`): 5xx rate > 1% 5 phút, p95 > 300 ms 10 phút, `/readyz` fail, River job failed > 10/giờ, pool wait > 100 ms | | | Alert YAML; 1 drill fire → Telegram/email (05) | [M] |

## 6. T — Tests (T01–T05)

| ID | Mục | Target Bare | Target Perfect | Cách check | M |
|---|---|---|---|---|---|
| T01 | Unit: domain packages `auth, study, deck, billing, stats` ≥ 80% dòng; table-driven; không sleep; `clock` mock | ≥ 90% | | B3 evidence `cover.txt` | |
| T02 | Integration: mọi endpoint MUST có ≥ 1 test happy + 1 test lỗi qua HTTP thật (`httptest.Server` + testcontainers); assert Problem `code` và header | | | `routes_test.go` bảng endpoint × test; script so với openapi paths: 100% MUST covered | |
| T03 | schemathesis trên staging: 0 failure, `--checks all` | Fuzz nightly | | B5 evidence | |
| T04 | `go test -race` toàn bộ = 0 race; `-shuffle=on` | | | B3 | |
| T05 | Bench + benchstat cho FSRS, JWT, cursor; regress > 10% → CI cảnh báo | | | B4 | |

## 7. B — Build & CI (B01–B06)

| ID | Mục | Target Bare | Target Perfect | Cách check | M |
|---|---|---|---|---|---|
| B01 | `Makefile`: `gen` (oapi-codegen + sqlc), `lint`, `test`, `test-integration`, `bench`, `build` (`-trimpath -ldflags="-s -w -X main.version=$(GIT_SHA)"`), `docker`, `k6`; `go.mod` `go 1.2x` mới nhất stable; `toolchain` pin | | | `make -n` liệt kê target; `go version` khớp Dockerfile | |
| B02 | `GET /v1/version` trả `api_version` (từ openapi `info.version`), `git_sha`, `fsrs_version`, `min_ios_build`, `min_web_build` (2 cái sau từ config) | | | `curl /v1/version` | |
| B03 | Dockerfile: multi-stage, `golang:1.2x-alpine` build `CGO_ENABLED=0`, runtime `gcr.io/distroless/static-debian12:nonroot`, `USER nonroot`, `EXPOSE 8080`, `HEALTHCHECK` không cần (compose làm), image ≤ 25 MB, multi-arch `linux/amd64,linux/arm64` (VPS ARM); `.dockerignore` loại `.git, *_test.go, loadtest` | ≤ 15 MB; SBOM + cosign | | B8 evidence; `docker run --rm img id` fail (distroless không shell) | |
| B04 | CI `backend.yml`: `make gen && git diff --exit-code` (drift) → lint → `test -race` → integration → `govulncheck` → build image → trivy → push `ghcr.io` tag `sha` + `main`; `contract.yml`: `redocly lint` + gen 3 phía + drift. PR có drift → đỏ | Nightly `nightly-loadtest.yml` (k6 staging 100 VU, schemathesis) | | Workflow files; 1 PR mẫu đỏ do drift (screenshot) | |
| B05 | Image chạy read-only FS: không ghi file (log stdout, temp không dùng); `--read-only` test pass | | | `docker run --read-only img api` boot ok | |
| B06 | Reproducible: `go build` 2 lần cùng sha → cùng hash (trimpath, không timestamp) | | | `sha256sum` 2 build | |

## 8. Bare Minimum vs Perfect

| Bare | Perfect |
|---|---|
| A01–A18 (trừ undo/promo/word-of-day), S01–S10, D01–D06, O01–O05, T01–T05, B01–B06; p95 ≤ 120 ms @200 VU | Undo, promo, export self-serve, reconcile RC, FSRS optimizer job, rate limit phân tán, Neon branch per PR, 100% tracing + tail sampling, Loki logs, SBOM/cosign, 2 replicas zero-downtime |

## 9. G — Prompt cho AI (copy nguyên văn)

```
Bạn là Go backend reviewer. Đầu vào: docs/checklists/03-backend-go.md, docs/01-CONTRACTS.md, contracts/openapi.yaml,
thư mục backend/ (toàn bộ, gồm db/queries, db/migrations, Dockerfile, Makefile, .golangci.yml, sqlc.yaml, oapi-codegen.yaml),
.github/workflows/{backend,contract}.yml và evidence docs/evidence/backend/ nếu có.
Kiểm tra từng mục A01→A18, S01→S10, D01→D06, O01→O05, T01→T05, B01→B06.
Với mỗi mục trả về đúng 1 dòng bảng:
ID | PASS/FAIL/N-A | bằng chứng (file:dòng hoặc lệnh + output rút gọn) | việc cần sửa (nếu FAIL)
Quy tắc:
- Không PASS nếu không có bằng chứng. Số đo (p95, coverage, image size) trích từ evidence, không ước lượng.
- Mục [M] chỉ ghi "CẦN MANUAL".
- Với A10: liệt kê TỪNG query trong db/queries đụng bảng users, decks, cards, card_states, review_logs, pending, entitlements,
  refresh_tokens, idempotency_keys và cho biết có user_id/owner_id trong WHERE hay không. Không có = FAIL A10.
- Với A05: liệt kê mọi chỗ ghi response không qua problem.Write hoặc generated code.
- Cuối cùng liệt kê: mọi time.Now(), uuid.New(), fmt.Sprintf chứa SQL, SELECT *, http.Error, pool.Begin ngoài platform/db,
  và mọi endpoint MUST trong openapi chưa có integration test.
- Kết thúc bằng bảng tổng PASS/FAIL/MANUAL theo nhóm.
```

## 10. Re-check thủ công (bạn làm, ~40 phút, trên staging)

1. **Refresh reuse (A08)**: dùng `curl` lấy refresh R1 → refresh thành R2 → dùng lại R1 → phải nhận `refresh_reused` và R2 cũng chết (family revoke). Đăng nhập lại mới được.
2. **Idempotency (A11)**: gửi `POST /v1/study/reviews` cùng body 3 lần cùng `Idempotency-Key`; DB `select count(*) from review_logs where user_id=...` tăng đúng 1 lần.
3. **Ownership (A14)**: user A `PATCH /v1/decks/{id của B}` → 403 `forbidden`, không 404 lộ tồn tại? (Quyết định: trả 404 cho private deck người khác để không lộ; ghi vào Contracts nếu đổi.) Kiểm tra đúng như contract ghi.
4. **Rate limit (A15)**: bắn 11 request `/auth/email/otp/request` trong 1 phút từ 1 IP → request 11 = 429, header `Retry-After` hợp lý.
5. **Graceful (A03)**: chạy `hey -z 30s`, giữa chừng `docker compose restart api`; đếm non-2xx; log có "shutdown complete".
6. **DB down (A03)**: pause Neon (hoặc đổi DATABASE_URL sai) → `/readyz` 503 trong ≤ 30 s; Caddy trả 503 + `Retry-After`; alert nổ (O05).
7. **Log (S03)**: grep log staging 1 ngày cho `eyJ`, `@gmail`, 6 chữ số liền sau "otp" = 0 kết quả.
8. **Webhook replay (A16)**: lấy 1 event RevenueCat sandbox, gửi lại 3 lần → `entitlements` không đổi, `billing_events` 1 dòng.

## 11. Evidence phải có khi đóng Phase 1

`docs/evidence/backend/`: `<date>-k6-study-flow.json` + `summary.txt`, `-k6-dictionary-search.json`, `-k6-auth-refresh.json`, `-pprof-cpu.txt`, `-pprof-heap.txt`, `-cover.txt`, `-bench.txt`, `-schemathesis.txt`, `-govulncheck.txt`, `-golangci.json`, `-image.txt`, `-trivy.json`, `-trace-sample.png`, `-log-sample.jsonl`, `checklist-03-run-<date>.md`.
