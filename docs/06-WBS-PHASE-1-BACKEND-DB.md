# 06 — WBS PHASE 1: Backend Go + Postgres Database + Infra Setup

> Phase 1 = "Engine & Data Core": Xây dựng toàn bộ Backend Go (Modular Monolith), Neon PostgreSQL 17, River Background Jobs, FSRS v4.5 engine, và hạ tầng VPS Staging/Production cơ bản.
> Format chuẩn: **Input → Hành động (lệnh/file) → Output → DoD (Kiểm tra tự động & thủ công)**.
> Nguyên tắc: Không viết API tùy tiện; toàn bộ handler bám sát `contracts/openapi.yaml` và `01-CONTRACTS.md`.

---

## Bảng phụ thuộc Phase 1

```
P1-01 DB Schema & Migration ──► P1-02 sqlc CodeGen ──► P1-04 Repository Layer
P0-05 openapi.yaml ──────────► P1-03 oapi-codegen  ──► P1-05 HTTP Strict Server
P1-06 FSRS Core Engine ──────► P1-07 Study Service ──► P1-08 Queue & Review Endpoints
P1-09 Auth & Token Engine ───► P1-10 Auth Middleware ─► P1-11 Protected Routes
P1-12 Dictionary ETL Pipeline ─► P1-13 Seed DB ──────► P1-14 Search API
P1-15 River Worker & Cron ───► P1-16 Stats Aggregator ─► P1-17 Nightly Jobs
P1-18 Caddy & Compose VPS ───► P1-19 CI/CD Pipeline ──► P1-20 Staging Deployment
```

---

## Chi tiết từng gói công việc (Work Packages)

### P1-01 — PostgreSQL Schema Migration (goose)

| ID | Input | Hành động (Lệnh / File) | Output | DoD (Definition of Done) |
|---|---|---|---|---|
| P1-01.1 | `docs/db/ERD.md` | Viết migration `backend/db/migrations/0001_init.sql` khởi tạo extensions (`pg_trgm`, `citext`, `unaccent`, `pg_stat_statements`) và tạo 14 bảng cốt lõi (`users`, `refresh_tokens`, `email_otps`, `idempotency_keys`, `dictionary_entries`, `example_sentences`, `decks`, `cards`, `card_states`, `review_logs`, `stats_daily`, `entitlements`, `billing_events`, `admin_audit_logs`). | File migration SQL | Chạy `goose postgres "$DATABASE_URL" up` thành công 100% không warning. |
| P1-01.2 | `0001_init.sql` | Thêm các constraint toàn vẹn: CHECK constraint cho `role`, `visibility`, `state`, `rating` (1..4), `stability > 0`, `difficulty BETWEEN 1 AND 10`. Thêm trigger/rule append-only trên `review_logs` và `admin_audit_logs` (`REVOKE UPDATE, DELETE ON ... FROM app_rw`). | Migration cập nhật | Cố ý thực hiện `UPDATE review_logs` với role `app_rw` trả về lỗi `permission denied for table review_logs`. |
| P1-01.3 | SQL Schema | Thiết lập chỉ mục (Indexes) hiệu năng cao: GIN trgm index trên `dictionary_entries.headword_norm`, partial composite index `ix_card_states_user_due` on `(user_id, due) WHERE state <> 'new'`, unique index trên `(user_id, client_review_id)`. | Index scripts | Chạy `squawk` lint SQL báo 0 lỗi rule violation. |
| P1-01.4 | CI Setup | Viết test script kiểm tra migration rollback: `goose up` -> `goose down` -> `goose up`. | Script test CI | Migration idempotent, rollback sạch sẽ không sót orphan types/tables. |

---

### P1-02 & P1-03 — Code Generation (sqlc & oapi-codegen)

| ID | Input | Hành động (Lệnh / File) | Output | DoD (Definition of Done) |
|---|---|---|---|---|
| P1-02.1 | `db/migrations/` | Viết các file truy vấn SQL thuần trong `backend/db/queries/`: `users.sql`, `auth.sql`, `dictionary.sql`, `decks.sql`, `cards.sql`, `study.sql`, `stats.sql`, `billing.sql`, `admin.sql`. | 9 SQL query files | Các query dùng đúng parameter type-casting (`$1::uuid`, `$2::timestamptz`). |
| P1-02.2 | `sqlc.yaml` | Cấu hình `sqlc.yaml` với engine `postgresql`, schema `db/migrations`, queries `db/queries`, output `internal/gen/db`, type overrides cho UUID (`github.com/google/uuid`) và JSONB (`encoding/json.RawMessage`). | `backend/sqlc.yaml` | Chạy `sqlc generate` sinh ra mã Go type-safe trong `internal/gen/db/` 0 lỗi. |
| P1-03.1 | `contracts/openapi.yaml` | Cấu hình `backend/oapi-codegen.yaml` sinh `StrictServerInterface`, server types, chi tiết routing cho `net/http` (Go 1.22+ stdlib router). | `backend/oapi-codegen.yaml` | Lệnh `oapi-codegen --config oapi-codegen.yaml contracts/openapi.yaml` sinh file `internal/gen/api/api.gen.go`. |
| P1-03.2 | `Makefile` | Tạo target `make gen-go` tự động chạy sqlc và oapi-codegen; target `make check-drift` kiểm tra `git diff --exit-code`. | `backend/Makefile` | Chạy `make gen-go` thành công; commit mã sinh vào git. |

---

### P1-04 & P1-05 — Platform Core & HTTP Strict Server Implementation

| ID | Input | Hành động (Lệnh / File) | Output | DoD (Definition of Done) |
|---|---|---|---|---|
| P1-04.1 | Go stdlib & pgx/v5 | Xây dựng package `internal/platform/db`: thiết lập connection pool với `pgxpool.Config` (MaxConns: 25, MinConns: 5, MaxConnLifetime: 30m, MaxConnIdleTime: 5m, HealthCheckPeriod: 1m). | `pool.go`, `tx.go` | Unit test bọc transaction tự động rollback khi panic/error, commit khi thành công. |
| P1-04.2 | `log/slog` | Xây dựng logging module có cấu trúc chuẩn JSON, tự động đính kèm `request_id`, `user_id`, `trace_id`, `latency_ms`, che giấu thông tin nhạy cảm (PII, authorization headers, passwords). | `internal/platform/log/logger.go` | Log output chuẩn JSON 100% tuân thủ OTel format. |
| P1-04.3 | `net/http` | Triển khai HTTP Middleware Stack: `RequestIDMiddleware` (đọc `X-Request-Id` hoặc tạo UUIDv7), `LoggerMiddleware`, `RecovererMiddleware`, `CorsMiddleware`, `RateLimitMiddleware` (token bucket theo client IP / User ID), `IdempotencyMiddleware`. | `internal/httpapi/middleware.go` | Mọi request lỗi trả về RFC 7807 `application/problem+json` chuẩn xác. |
| P1-05.1 | `api.gen.go` | Cài đặt `StrictServerInterface` trong `internal/httpapi/server.go`: map toàn bộ method API đã khai báo sang domain services tương ứng. | `server.go` | Server compile thành công, implement 100% interface, 0 stub rỗng chưa xử lý. |

---

### P1-06, P1-07 & P1-08 — FSRS Algorithm Engine & Study System

| ID | Input | Hành động (Lệnh / File) | Output | DoD (Definition of Done) |
|---|---|---|---|---|
| P1-06.1 | FSRS v4.5 Spec | Xây dựng thuật toán Free Spaced Repetition Scheduler trong `internal/study/fsrs.go`: tính toán Stability ($S$), Difficulty ($D$), khoảng cách ngày ôn tập tiếp theo ($Interval$) dựa trên rating (1=Again, 2=Hard, 3=Good, 4=Easy). | `fsrs.go`, `parameters.go` | Chạy bộ test vectors `contracts/fixtures/fsrs_cases.json` qua `TestFSRS_Parity` đạt 100% pass với sai số $\le 10^{-6}$. |
| P1-07.1 | `internal/gen/db` | Xây dựng `StudyService`: logic lấy Study Queue (`GET /v1/study/queue`): lấy tối đa `limit` thẻ bao gồm (cards Due $\le$ now() sắp xếp theo due ASC + new cards sắp xếp theo position ASC). | `internal/study/queue.go` | Queue tôn trọng daily limit (`daily_goal` của user), không bao giờ trả về thẻ trùng lặp trong cùng 1 queue session. |
| P1-08.1 | Review Submission | Xây dựng xử lý nộp kết quả học `POST /v1/study/reviews`: xử lý batch reviews trong 1 database transaction. Cập nhật `card_states`, ghi bản ghi append-only vào `review_logs`, cập nhật `stats_daily`. | `internal/study/review.go` | Đảm bảo tính Idempotent qua `client_review_id`: nộp 2 lần cùng một review ID chỉ ghi nhận 1 lần và trả về kết quả ban đầu (HTTP 200). |

---

### P1-09 & P1-10 — Authentication & Session Security

| ID | Input | Hành động (Lệnh / File) | Output | DoD (Definition of Done) |
|---|---|---|---|---|
| P1-09.1 | Apple Sign In & Resend OTP | Xây dựng `AuthService`: xác thực Apple Identity Token qua JWKS caching của Apple; sinh mã OTP 6 số lưu hash SHA-256 vào `email_otps` với thời hạn sống 10 phút, gửi email qua Resend API. | `internal/auth/service.go` | Rate limit OTP: tối đa 3 lần gửi/15 phút/email, chặn brute force sau 5 lần nhập sai. |
| P1-09.2 | JWT & Refresh Token Family | Phát hành Access Token (Ed25519 hoặc HMAC-SHA256, hạn 15 phút) và Refresh Token (Opaque Token ngẫu nhiên 32 bytes cryptographically secure, hạn 30 ngày, lưu SHA-256 hash vào DB). | `internal/auth/jwt.go`, `internal/auth/refresh.go` | Triển khai **Refresh Token Rotation (RTR)** với **Token Family (sid)**: nếu phát hiện 1 refresh token đã dùng được tái sử dụng -> lập tức revoke toàn bộ token family của user đó (ngăn chặn token theft). |
| P1-10.1 | Auth Middleware | Triển khai `AuthMiddleware`: trích xuất Bearer token, xác minh chữ ký và claims (`sub` = user_id, `role`), gán `UserContext` vào `context.Context`. | `internal/httpapi/auth_ctx.go` | Request không có token hoặc token hết hạn trả về HTTP 401 với Problem Details code `AUTH_UNAUTHORIZED` / `AUTH_TOKEN_EXPIRED`. |

---

### P1-12, P1-13 & P1-14 — Dictionary Engine & ETL Data Import

| ID | Input | Hành động (Lệnh / File) | Output | DoD (Definition of Done) |
|---|---|---|---|---|
| P1-12.1 | Kaikki (Wiktionary JSONL), Tatoeba, NGSL | Viết công cụ ETL trong `backend/tools/import-dictionary/`: parse file Wiktionary JSONL, trích xuất Headword, IPA chuẩn Mỹ, Part of Speech, định nghĩa ngắn gọn, ví dụ song ngữ En-Vi từ Tatoeba, độ phổ biến từ NGSL (frequency rank 1..10000), phân loại trình độ CEFR (A1..C2). | `cmd/app` subcommand `import` | Xử lý streaming file dung lượng 2GB với RAM tiêu thụ $\le 256$ MB, lưu batch insert vào Postgres với batch size 1000 bản ghi. |
| P1-13.1 | Text Normalization | Chuẩn hóa `headword_norm`: chuyển thành chữ thường, loại bỏ dấu thanh điệu / ký tự đặc biệt phục vụ tìm kiếm nhanh (Unaccent). | Trigger/Go normalization logic | Truy vấn tìm kiếm từ điển không phân biệt hoa thường và không phân biệt dấu. |
| P1-14.1 | Dictionary Search API | Triển khai `GET /v1/dictionary/search`: kết hợp Prefix Search (`LIKE 'prefix%'`) và Trigram Fuzzy Search (`similarity(headword_norm, query) > 0.3`), ưu tiên sắp xếp theo `freq_rank ASC`. | `internal/dictionary/search.go` | Thời gian phản hồi truy vấn tìm kiếm p95 $\le 15$ ms trên tập dữ liệu 50,000 từ. |

---

### P1-15, P1-16 & P1-17 — Background Jobs (River) & Operational Maintenance

| ID | Input | Hành động (Lệnh / File) | Output | DoD (Definition of Done) |
|---|---|---|---|---|
| P1-15.1 | River Postgres Engine | Cài đặt và cấu hình River (`riverqueue`): tạo schema bảng job trong Postgres qua `river migrate-up`. Khởi tạo River Client chạy bên trong worker process. | `internal/platform/jobs/river.go` | Enqueue job trong cùng một Database Transaction với business logic (`river.Client.InsertTx`). |
| P1-16.1 | Daily Aggregations | Tạo River Worker `DailyStatsAggregatorArgs`: định kỳ lúc 00:05 mỗi ngày (theo từng múi giờ của user), tổng hợp số từ đã học, số phút học, tỷ lệ ghi nhớ retention, cập nhật `stats_daily`. | `internal/stats/worker.go` | Job xử lý 10,000 users trong $\le 60$ giây, an toàn với retry khi lỗi mạng. |
| P1-17.1 | Database Housekeeping | Tạo River Worker `CleanupExpiredDataArgs`: xóa các `idempotency_keys` quá hạn (> 24h), xóa `email_otps` quá hạn, xóa `refresh_tokens` đã revoked > 30 ngày. | `internal/platform/jobs/cleanup.go` | Chạy định kỳ 6 tiếng/lần, giữ bảng gọn nhẹ. |

---

### P1-18, P1-19 & P1-20 — Infrastructure, Docker & Staging Deployment

| ID | Input | Hành động (Lệnh / File) | Output | DoD (Definition of Done) |
|---|---|---|---|---|
| P1-18.1 | Go Binary | Viết `backend/Dockerfile` multi-stage: stage 1 dùng `golang:1.24-alpine` biên dịch binary tĩnh với cgo disabled (`CGO_ENABLED=0`, `-ldflags="-s -w"`), stage 2 dùng Google Distroless `gcr.io/distroless/static-debian12:nonroot`. | `backend/Dockerfile` | Kích thước Docker image cuối cùng $\le 25$ MB, chạy dưới quyền non-root (UID 65532). |
| P1-18.2 | Compose & Caddy | Viết `infra/compose/docker-compose.yml` và `infra/compose/Caddyfile`: cấu hình reverse proxy Caddy tự động quản lý chứng chỉ SSL, proxy `/v1/*` tới container `api:8080`, chặn truy cập ngoài tới port `6060` (metrics/pprof). | File cấu hình infra | Khởi động qua `docker compose up -d` kiểm tra `/readyz` trả về HTTP 200. |
| P1-19.1 | GitHub Actions CI/CD | Tạo `.github/workflows/backend.yml`: chạy `golangci-lint`, `go test -race ./...`, `schemathesis`, build docker image và push lên GitHub Container Registry (GHCR) gắn tag Git SHA. | Workflow file | CI chạy hoàn tất trong $\le 3$ phút, pass 100% tests. |
| P1-20.1 | SSH Auto Deploy | Tạo `.github/workflows/deploy-staging.yml`: SSH vào VPS Staging, thực hiện pull image mới, chạy `app migrate up` qua container tạm thời, khởi động lại service với zero-downtime, kiểm tra `/readyz` trong 60s, tự động rollback nếu thất bại. | Deployment workflow | Deploy Staging tự động kích hoạt sau khi merge PR vào branch `main`. |

---

## Tiêu chí hoàn thành Phase 1 (Exit Criteria)

1. [ ] Toàn bộ 22 endpoints định nghĩa trong OpenAPI 3.1 hoạt động chính xác trên môi trường Staging.
2. [ ] Test coverage của core domain package (`internal/study`, `internal/auth`, `internal/dictionary`) đạt $\ge 80\%$, chạy với cờ `-race` 0 lỗi race condition.
3. [ ] Schemathesis chạy tự động quét toàn bộ API trên Staging đạt **0 failure** và **0 server 500 error**.
4. [ ] Thuật toán FSRS v4.5 trên Go khớp 100% kết quả với file fixture `fsrs_cases.json`.
5. [ ] Tải trọng kiểm thử k6 với 200 Virtual Users đồng thời đạt: Latency p95 $\le 120$ ms, Error rate $< 0.1\%$.
6. [ ] Docker image backend đạt dung lượng $\le 25$ MB, 0 lỗ hổng bảo mật mức HIGH/CRITICAL khi quét qua Trivy/govulncheck.
