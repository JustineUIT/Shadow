# ADR 0001 — Backend: modular monolith Go trên stdlib, pgx + sqlc, goose, River

- Status: **Accepted** (2026-09-10)
- Bao phủ: D01, D03, D04, D05, D06

## Context
Một người làm toàn bộ. Mục tiêu kép: ship sản phẩm học tiếng Anh **và** học Go đúng cách. Chưa có bất kỳ số đo tải nào. Cần background job (rollup stats, hard-delete account, gửi mail) nhưng không muốn thêm hệ thống phải vận hành.

## Options
| Hạng mục | Options | Loại vì |
|---|---|---|
| Kiến trúc | Microservices; Serverless functions; **Modular monolith** | Microservices: chi phí vận hành/ debug cho 1 người; Serverless: cold start, vendor lock, khó chạy River/pgx pool |
| HTTP | gin, fiber, echo, chi, **stdlib net/http ≥ 1.22** | Framework che mất cách Go hoạt động; stdlib routing đã đủ pattern `GET /v1/decks/{id}` |
| DB layer | GORM, ent, sqlx, **pgx v5 + sqlc** | ORM sinh SQL khó kiểm soát index; sqlc cho type-safe từ SQL thật |
| Migration | golang-migrate, atlas, tern, **goose** | goose đơn giản, SQL thuần, embed vào binary, chạy `app migrate up` lúc deploy |
| Jobs | cron goroutine, Redis + asynq, **River** | River dùng chính Postgres, enqueue trong cùng transaction → không mất job; không thêm Redis |

## Decision
1 module Go, 1 binary `cmd/app` với subcommand `api | worker | migrate | import`. Cấu trúc `internal/httpapi → internal/<domain> → internal/platform + internal/gen/db`. Domain package không import `net/http`, không import nhau (ép bằng `depguard`).

## Consequences
- (+) 1 image, 1 compose service `api` + 1 `worker`; deploy 1 lệnh.
- (+) Test integration bằng testcontainers Postgres, không mock DB.
- (−) Không có tác động cô lập theo service: 1 bug panic có thể hạ cả API → bắt buộc recover middleware + healthcheck restart (checklist 03 A03, 05 A04).
- (−) Học stdlib chậm hơn framework lúc đầu; bù bằng oapi-codegen `std-http` sinh routing.
- Tách service **chỉ khi** có số đo (k6 p95 vượt target sau khi đã tối ưu query) — ghi ADR mới.

## Checklist liên quan
03-backend-go: A01–A12, B03. 04-database: mục A, M.
