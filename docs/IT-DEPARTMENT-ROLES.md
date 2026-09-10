# IT DEPARTMENT: Tổ chức Phòng ban IT & Bản đồ Trách nhiệm Triển khai Shadow

> **Mục tiêu**: Đóng vai trò là một Phòng ban IT hoàn chỉnh (Phần mềm, Hạ tầng, Dữ liệu, QA, Bảo mật) để phân rã chi tiết toàn bộ dự án Shadow từ A đến Z, lựa chọn các giải pháp tối ưu nhất, và cam kết chất lượng thông qua các chỉ số đo lường thực tế.

---

## 1. Bản đồ Cơ cấu Phòng ban IT (Department Organizational Chart)

```
                       ┌─────────────────────────────────────────┐
                       │      Principal Enterprise Architect      │
                       │     (Kiến trúc sư Trưởng Toàn hệ thống)  │
                       └────────────────────┬────────────────────┘
                                            │
         ┌──────────────────┬───────────────┴───────────────┬──────────────────┐
         │                  │                               │                  │
┌────────▼─────────┐ ┌──────▼───────────┐         ┌─────────▼────────┐ ┌───────▼────────┐
│ Lead iOS Engineer│ │Lead Frontend Eng │         │ Lead Backend Eng │ │Lead DBA/Data Eng│
│ (Swift 6/SwiftUI)│ │(Next.js 16 Static│         │ (Go Monolith)    │ │(Postgres 17 Neon│
└────────┬─────────┘ └──────┬───────────┘         └─────────┬────────┘ └───────┬────────┘
         │                  │                               │                  │
         └──────────────────┼───────────────────────────────┴──────────────────┘
                            │
         ┌──────────────────┴──────────────────┐
         │                                     │
┌────────▼──────────────┐             ┌────────▼──────────────┐
│  Lead DevOps/SRE Eng  │             │ Lead QA & Performance │
│ (VPS / Cloudflare /CD)│             │ (k6 / Instruments/LH) │
└───────────────────────┘             └───────────────────────┘
```

---

## 2. Chi tiết Từng Vai trò Chuyên môn trong Phòng ban

---

### Vai trò 1: Principal Enterprise Architect (Kiến trúc sư Trưởng)

#### 1. Trách nhiệm Cốt lõi
- Thiết lập ranh giới hệ thống, quản lý 4 Nguồn sự thật (Single Source of Truth) trong `contracts/`.
- Quyết định kiến trúc tổng thể, loại bỏ hoàn toàn tình trạng lệch pha giữa các nền tảng (API drift, Token mismatch).

#### 2. Đánh giá Tùy chọn & Quyết định Tối ưu
- **Tùy chọn cân nhắc**:
  1. *Microservices (gRPC + Event Bus)*: Quá phức tạp, chi phí vận hành cao cho đội ngũ nhỏ.
  2. *Serverless Functions (AWS Lambda / Vercel)*: Cold start cao, khó tối ưu FSRS engine và stateful background jobs.
  3. **Modular Monolith Go (1 Binary + Subcommands)**: **ĐƯỢC CHỌN**. Đơn giản, siêu nhẹ ($\le 25$ MB RAM khi nhàn rỗi), deploy 1 binary duy nhất, dễ dàng tách microservices sau này khi có số liệu tải thực tế.
- **Quản lý Contract**:
  - *Chọn*: **OpenAPI 3.1 Contract-First**. Định nghĩa tại `contracts/openapi.yaml`, sinh tự động mã nguồn cho Go (`oapi-codegen`), iOS Swift (`swift-openapi-generator`), và Web React (`openapi-typescript`).

---

### Vai trò 2: Lead iOS Engineer (Trưởng nhóm Kỹ sư iOS Swift)

#### 1. Trách nhiệm Cốt lõi
- Xây dựng ứng dụng iOS Native theo chuẩn Swift 6 Strict Concurrency, SwiftUI, Offline-First với GRDB SQLite.
- Đảm bảo trải nghiệm 60/120 FPS, không rò rỉ bộ nhớ (0 Leaks), tương thích 100% Apple Human Interface Guidelines.

#### 2. Đánh giá Tùy chọn & Quyết định Tối ưu
- **Kiến trúc UI**:
  - *Tùy chọn*: UIKit MVC vs The Composable Architecture (TCA) vs SwiftUI MV.
  - *Chọn*: **SwiftUI MV (`@Observable`) kết hợp Local SPM Packages**. TCA tạo boilerplates quá nặng; SwiftUI MV với `@Observable` (iOS 17+) tinh gọn, hiệu năng cao, tách package ép dependency 1 chiều chống circular dependency.
- **Lưu trữ Cục bộ (Local Database)**:
  - *Tùy chọn*: Core Data vs SwiftData vs GRDB (SQLite).
  - *Chọn*: **GRDB.swift**. Kiểm soát 100% SQL schema, migration minh bạch, hiệu năng vượt trội, test isolation tuyệt đối.

#### 3. Bảng Khai báo Input & Output
- **Input (Require Must Have)**: `contracts/openapi.yaml`, `contracts/tokens/tokens.json`, `contracts/events.yaml`, `contracts/fixtures/fsrs_cases.json`, Backend APIs.
- **Output (Require Must Have)**: `.ipa` Release build $\le 30$ MB, Cold launch $\le 400$ ms, 0 memory leak, Offline study loop hoàn chỉnh.
- **Optional (Perfect Tier)**: Hỗ trợ iPad Layout, Interactive Home Screen Widgets, Live Activities, WatchOS companion.

#### 4. Tool Đo Hiệu Năng Chuyên dụng & Mục tiêu
- **Tool**: **Xcode Instruments (App Launch + Allocations/Leaks + Animation Hitches) + XCTest MetricKit**.
- **Chỉ số chứng minh thật**: Cold Launch $\le 400$ ms, Hitch Ratio $< 5$ ms/s, Memory Peak $\le 150$ MB, 0 Leaks.

---

### Vai trò 3: Lead Frontend Engineer (Trưởng nhóm Kỹ sư React / Web)

#### 1. Trách nhiệm Cốt lõi
- Xây dựng Landing Page, Cổng Pháp lý (Legal Pages), và Admin Portal SPA trên nền Next.js 16 Static Export.
- Triển khai Design System CSS Variables sinh tự động từ Design Tokens, tối ưu First Load JS $\le 90$ KB.

#### 2. Đánh giá Tùy chọn & Quyết định Tối ưu
- **Mô hình Triển khai Web**:
  - *Tùy chọn*: Next.js SSR (Node Server) vs Remix vs Next.js Static Export (`output: 'export'`).
  - *Chọn*: **Next.js 16 Static Export trên Cloudflare Pages**. Loại bỏ chi phí server, 0 rủi ro bảo mật Node runtime, phân phối toàn cầu qua Cloudflare Edge Network với chi phí 0đ.
- **UI Component Library**:
  - *Tùy chọn*: Material UI (MUI) vs Ant Design vs shadcn/ui.
  - *Chọn*: **shadcn/ui map sang CSS variables `--ds-*`**. Nhẹ, copy-paste mã nguồn trực tiếp, tùy biến 100% theo token design system.

#### 3. Bảng Khai báo Input & Output
- **Input (Require Must Have)**: `contracts/tokens/tokens.json`, `contracts/openapi.yaml`, nội dung PRD/Legal.
- **Output (Require Must Have)**: Static HTML/JS trong `web/out/`, Landing JS $\le 90$ KB, Lighthouse Mobile $\ge 95$, 5 Legal pages chuẩn App Store.
- **Optional (Perfect Tier)**: Dark mode theme switcher, Design system visual documentation site (`/design`), Blog CMS tích hợp MDX.

#### 4. Tool Đo Hiệu Năng Chuyên dụng & Mục tiêu
- **Tool**: **Lighthouse CI (`@lhci/cli`) + `@axe-core/playwright` + `size-limit`**.
- **Chỉ số chứng minh thật**: Performance $\ge 95$, Accessibility $\ge 95$, LCP $\le 2.0$ s, CLS $\le 0.05$, TBT $\le 150$ ms, 0 Critical a11y errors.

---

### Vai trò 4: Lead Backend Engineer (Trưởng nhóm Kỹ sư Go)

#### 1. Trách nhiệm Cốt lõi
- Xây dựng Go HTTP Server tuân thủ nghiêm ngặt OpenAPI 3.1 Strict Interface, tối ưu kết nối DB pgx v5 pool, xử lý Background Jobs qua River.
- Triển khai thuật toán FSRS v4.5 engine, bảo đảm tính Idempotency và chuẩn hóa lỗi RFC 7807 Problem Details.

#### 2. Đánh giá Tùy chọn & Quyết định Tối ưu
- **HTTP Routing & Server Engine**:
  - *Tùy chọn*: Gin vs Fiber vs Echo vs Go stdlib `net/http`.
  - *Chọn*: **Go 1.22+ stdlib `net/http` + `oapi-codegen` strict server**. Không phụ thuộc third-party router nặng, tối ưu hiệu năng và bộ nhớ tuyệt đối.
- **Database Access Layer**:
  - *Tùy chọn*: GORM vs Ent vs SQLx vs `pgx/v5` + `sqlc`.
  - *Chọn*: **pgx/v5 + sqlc**. Viết SQL thuần có type-checking lúc compile time, 0 overhead runtime reflection, hiệu năng gần như C/Go thô.
- **Background Jobs Engine**:
  - *Tùy chọn*: Cron in-memory vs Redis + Asynq vs River (Postgres-backed).
  - *Chọn*: **River**. Tận dụng Postgres sẵn có, không cần thêm Redis, hỗ trợ transactional enqueue (`InsertTx`) chống mất job.

#### 3. Bảng Khai báo Input & Output
- **Input (Require Must Have)**: `contracts/openapi.yaml`, Postgres Connection String, Env Variables.
- **Output (Require Must Have)**: Docker Image Distroless $\le 25$ MB, 22 Endpoints chuẩn REST/OpenAPI, RFC 7807 Errors, Metrics `/metrics`.
- **Optional (Perfect Tier)**: Tự động gán nhãn CEFR qua Local LLM/Piper TTS, OpenTelemetry Distributed Tracing đầy đủ.

#### 4. Tool Đo Hiệu Năng Chuyên dụng & Mục tiêu
- **Tool**: **k6 Load Tester + Schemathesis Contract Fuzzer + Go Benchmark (`go test -bench`) + govulncheck**.
- **Chỉ số chứng minh thật**: k6 @200 VUs: Latency p95 $\le 120$ ms, Error rate $< 0.1\%$, Schemathesis 0 failures, 0 CVEs.

---

### Vai trò 5: Lead DBA & Data Engineer (Trưởng nhóm Kỹ sư Dữ liệu)

#### 1. Trách nhiệm Cốt lõi
- Thiết kế mô hình cơ sở dữ liệu Postgres 17 (Neon Singapore), quản lý migration bằng goose, tối ưu hóa chỉ mục (Indexes) và kế hoạch thực thi câu lệnh (Query Execution Plan).
- Xây dựng luồng ETL trích xuất dữ liệu từ điển Kaikki/Tatoeba và thực hiện quy trình sao lưu khôi phục thảm họa (Disaster Recovery Drill).

#### 2. Đánh giá Tùy chọn & Quyết định Tối ưu
- **Cơ sở dữ liệu**:
  - *Tùy chọn*: Self-hosted Postgres trên VPS vs Supabase vs Neon PostgreSQL 17.
  - *Chọn*: **Neon PostgreSQL 17 (Singapore)**. Managed connection pooling, tự động backup, point-in-time restore, serverless branching cho testing.
- **Tìm kiếm Từ điển**:
  - *Tùy chọn*: Elasticsearch/Meilisearch vs PostgreSQL Full-Text Search.
  - *Chọn*: **PostgreSQL Extensions (`pg_trgm` + `unaccent` + B-Tree prefix)**. Đủ đáp ứng 50,000 từ với độ trễ $\le 15$ ms, không tốn tài nguyên duy trì thêm cụm search engine rời.

#### 3. Bảng Khai báo Input & Output
- **Input (Require Must Have)**: Raw data Wiktionary/Tatoeba JSONL, Migration files `db/migrations/*.sql`.
- **Output (Require Must Have)**: 14 bảng chuẩn hóa 3NF, Chỉ mục composite/partial tối ưu, Quyền hạn phân định (`app_migrate`, `app_rw`, `app_ro`), Backup mã hóa AES-256 trên R2.
- **Optional (Perfect Tier)**: Neon Database Branching tự động cho mỗi Pull Request, pgvector lưu trữ Word Embeddings cho tìm kiếm ngữ nghĩa.

#### 4. Tool Đo Hiệu Năng Chuyên dụng & Mục tiêu
- **Tool**: **`EXPLAIN (ANALYZE, BUFFERS)` + `pg_stat_statements` + `squawk` linter + Kịch bản diễn tập `restore-drill.sh`**.
- **Chỉ số chứng minh thật**: Mean query execution time $\le 20$ ms, 0 Sequential Scans trên bảng $> 10,000$ dòng, RTO $\le 30$ phút, RPO $\le 24$h.

---

### Vai trò 6: Lead DevOps & SRE Engineer (Trưởng nhóm Vận hành & Hạ tầng)

#### 1. Trách nhiệm Cốt lõi
- Quản lý hạ tầng VPS Linux ARM Singapore, Docker Compose, Caddy Reverse Proxy, Cloudflare WAF/CDN/DNS, CI/CD GitHub Actions.
- Thiết lập giám sát cảnh báo thời gian thực qua Grafana Cloud & Alloy, bảo đảm máy chủ đạt chuẩn an toàn thông tin (Hardening).

#### 2. Đánh giá Tùy chọn & Quyết định Tối ưu
- **Hạ tầng Compute**:
  - *Tùy chọn*: Kubernetes (K8s) vs Fly.io / Render vs 1 VPS ARM Singapore + Docker Compose.
  - *Chọn*: **1 VPS ARM (Oracle Always Free / Hetzner CAX11) + Docker Compose + Caddy**. Tối ưu chi phí ($\sim 4-6$ EUR/tháng), không cold start, độ trễ cực thấp tới người dùng Việt Nam ($\le 30$ ms).
- **Reverse Proxy**:
  - *Tùy chọn*: Nginx vs Traefik vs Caddy.
  - *Chọn*: **Caddy v2**. Cấu hình đơn giản, hỗ trợ HTTP/3 và tự động nén zstd/gzip hiệu quả cao.

#### 3. Bảng Khai báo Input & Output
- **Input (Require Must Have)**: Backend Docker Image, Caddyfile, Docker Compose file, Environment Secrets (`/opt/app/.env`).
- **Output (Require Must Have)**: `https://api.<domain>/readyz` 200, Tường lửa UFW khóa sạch cổng thừa, Kịch bản deploy zero-downtime $\le 3$ phút kèm auto-rollback, Cảnh báo Telegram $\le 5$ phút.
- **Optional (Perfect Tier)**: Hạ tầng Blue/Green deployment hoàn toàn tự động, Cấu hình SOPS + Age mã hóa secrets trong Git.

#### 4. Tool Đo Hiệu Năng Chuyên dụng & Mục tiêu
- **Tool**: **`lynis audit system` + `testssl.sh` + `nmap` + `goss` server specification validator**.
- **Chỉ số chứng minh thật**: Lynis Hardening Index $\ge 75/100$, SSL Labs điểm A+, Chỉ mở duy nhất 3 cổng mạng (22, 80, 443), Deploy time $\le 3$ phút.

---

### Vai trò 7: Lead QA & Security Engineer (Trưởng nhóm Đảm bảo Chất lượng & An ninh)

#### 1. Trách nhiệm Cốt lõi
- Điều phối kiểm thử Parity giải thuật FSRS, kiểm tra E2E luồng người dùng đa thiết bị, quét lỗ hổng mã nguồn và container.
- Thu thập và cập nhật toàn bộ bằng chứng số liệu thật vào `docs/evidence/SCORECARD.md`.

#### 2. Bảng Khai báo Input & Output
- **Input (Require Must Have)**: Toàn bộ artifacts từ iOS, Web, Backend, DB, Infra; 6 Checklists trong `docs/checklists/`.
- **Output (Require Must Have)**: `SCORECARD.md` được điền đầy đủ 100% số liệu đo thực tế, Báo cáo quét bảo mật Trivy/govulncheck, Báo cáo FSRS Parity.

#### 3. Tool Đo Hiệu Năng Chuyên dụng & Mục tiêu
- **Tool**: **Bộ 6 công cụ chính hợp nhất (XCTest/MetricKit + Lighthouse CI + k6 + pg_stat_statements + Lynis/testssl + Schemathesis/Trivy)**.
- **Chỉ số chứng minh thật**: 100% tiêu chí Target Bare đạt trạng thái **PASS**.
