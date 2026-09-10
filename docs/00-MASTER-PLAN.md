# 00 — MASTER PLAN: English Learning App (Go + iOS Swift + React Web)

> Đọc file này đầu tiên. Mọi file khác trong `docs/` là chi tiết của file này.
> Không có ước lượng thời gian. Mỗi phase có **exit criteria**; chưa đạt thì chưa sang phase sau.

## 0. Cách dùng bộ tài liệu

| File | Dùng khi nào | Vai trò đọc |
|---|---|---|
| `00-MASTER-PLAN.md` | Bắt đầu; review cuối mỗi phase | Product Owner + Architect (bạn) |
| `01-CONTRACTS.md` | Trước khi viết bất kỳ code nào chạm biên giới giữa 2 hệ | Tất cả |
| `02-DESIGN-SYSTEM.md` | Trước khi viết UI đầu tiên (iOS hoặc web) | iOS, Web, Design |
| `03-MEASUREMENT-TOOLS.md` | Cuối mỗi phase để lấy số liệu thật | QA, SRE |
| `checklists/01-ios-swift.md` | Cho AI check, rồi tự re-check | iOS |
| `checklists/02-web-react.md` | | Web |
| `checklists/03-backend-go.md` | | Backend |
| `checklists/04-database-postgres.md` | | Data/DBA |
| `checklists/05-server-infra.md` | | DevOps/SRE |
| `checklists/06-domain-ops.md` | | Legal, Growth, Ops |

**Quy trình check 2 lớp** (áp dụng cho mọi checklist):
1. Paste checklist + source liên quan cho AI, dùng prompt mẫu ở mục G của từng checklist. AI phải trả bảng `ID | PASS/FAIL | bằng chứng (file:dòng hoặc output lệnh)`. Không có bằng chứng = không PASS.
2. Bạn tự re-check các mục đánh dấu `[M]` (manual). Đó là các mục AI hay "pass ảo": chạy trên thiết bị thật, tắt mạng, VoiceOver, restore backup, nmap từ ngoài, v.v.

---

## 1. Decision Log (options → chọn → lý do)

Đổi bất kỳ dòng nào phải viết ADR mới trong `docs/adr/NNNN-<slug>.md`.

| # | Hạng mục | Options đã cân nhắc | CHỌN | Lý do chính |
|---|---|---|---|---|
| D01 | Kiến trúc | Microservices / Serverless functions / Modular monolith | **Modular monolith Go, 1 binary, subcommand `api` \| `worker` \| `migrate` \| `import`** | 1 người, chưa có scale problem. Tách sau khi có số đo |
| D02 | Contract giữa các hệ | Code-first mỗi bên tự viết / GraphQL / gRPC / OpenAPI contract-first | **OpenAPI 3.1 là source of truth, codegen cho Go + Swift + TS** | Đây là câu trả lời cho "không khớp, sửa tới sửa lui": 1 file sinh 3 phía, CI fail nếu lệch |
| D03 | Go HTTP | gin / fiber / echo / chi / stdlib | **stdlib `net/http` (Go ≥ 1.22 routing) + oapi-codegen `std-http` strict server** | Ít dependency, học Go thật, đủ dùng |
| D04 | Go DB layer | GORM / ent / sqlx / pgx + sqlc | **pgx v5 + sqlc** | SQL thật, type-safe, không ORM magic |
| D05 | Migration | golang-migrate / atlas / tern / goose | **goose (SQL, embed vào binary)** | Đơn giản, chạy `app migrate up` lúc deploy |
| D06 | Background jobs | cron goroutine / Redis + asynq / River | **River (Postgres-backed)** | Không thêm Redis, enqueue trong transaction |
| D07 | Database | Supabase / self-host trên VPS / Neon | **Neon Postgres 17, region Singapore, pooled connection** | Managed backup + PITR + branch cho test. Free tier đủ MVP (chú ý giới hạn storage, xem checklist 04) |
| D08 | Compute | Cloud Run / Fly.io / Render / Oracle Free / Hetzner | **1 VPS ARM Singapore (Oracle Always Free nếu đăng ký được, fallback Hetzner CAX11) + Docker Compose + Caddy** | Học ops thật, ~4–6 EUR/tháng, không cold start, region gần user VN |
| D09 | Object storage | S3 / Backblaze B2 / Cloudflare R2 | **R2 + custom domain `cdn.`** | Egress free — quyết định với app nhiều audio |
| D10 | Edge / DNS / WAF | Route53 / Cloudflare | **Cloudflare (Registrar, DNS, proxy, WAF free, Pages)** | Free, DDoS, cache, cho phép dùng thương mại |
| D11 | iOS UI | UIKit / SwiftUI | **SwiftUI, iOS 17+, Swift 6 strict concurrency** | `@Observable`, năng suất, hướng Apple đang đẩy |
| D12 | iOS kiến trúc | MVVM / TCA / VIPER / MV + packages | **SwiftUI MV (ViewModel nhẹ `@Observable`) + local SPM packages theo feature** | TCA quá nặng cho solo; tách package để build nhanh, test được, ép dependency direction |
| D13 | iOS local DB | Core Data / SwiftData / GRDB | **GRDB (SQLite)** | Kiểm soát schema/migration, hiệu năng, test dễ. SwiftData chưa ổn với sync phức tạp |
| D14 | iOS networking | Alamofire / tự viết / swift-openapi-generator | **swift-openapi-generator + URLSession transport** | Sinh từ cùng `openapi.yaml`, không lệch contract |
| D15 | Auth | Firebase / Clerk / Supabase Auth / tự làm | **Tự làm trên Go: Sign in with Apple + Email OTP, JWT access 15 phút + refresh rotation** | Học đúng chỗ đáng học, không lock-in, Apple bắt buộc SIWA nếu có login bên thứ ba |
| D16 | Thanh toán | StoreKit 2 + tự verify JWS / StoreKit 2 + RevenueCat | **StoreKit 2 trên client, RevenueCat quản lý entitlement, webhook về Go** | Edge case subscription (grace, billing retry, refund, family) là bãi mìn; RC free đến 2.5k USD MTR |
| D17 | Web framework | Vite SPA / Astro / Remix / Next.js | **Next.js 16 `output: 'export'` (static) → Cloudflare Pages** | Landing SEO + admin SPA trong 1 app. **Vercel Hobby cấm dùng thương mại**, Cloudflare Pages free thì được |
| D18 | Admin UI | Retool / Refine / React-Admin / tự ghép | **shadcn/ui + TanStack Table + TanStack Query** | Dùng chung tokens với landing; đủ cho admin CRUD |
| D19 | Web API client | axios tự viết / openapi-typescript + openapi-fetch | **openapi-typescript + openapi-fetch** | Type sinh từ cùng contract |
| D20 | Thuật toán học | SM-2 / Leitner / FSRS | **FSRS. `go-fsrs` server-side là source of truth; client tính preview offline cùng version, test parity bằng fixtures chung** | Hiện đại, có optimizer, là lõi sản phẩm |
| D21 | Design tokens | Viết tay 2 nơi / Figma Tokens / tokens.json + Style Dictionary | **`contracts/tokens/tokens.json` (DTCG) → sinh `Tokens.swift` + `tokens.css`** | 1 nguồn, 2 platform khớp nhau |
| D22 | Observability | Datadog / New Relic / tự ghép free | **Sentry (error) + Grafana Cloud free (metrics/logs/traces) + PostHog (product)** | Đều free tier, đủ đo mọi target |
| D23 | Email | SES / Postmark / Resend | **Resend** | Free tier, DKIM dễ, API sạch |
| D24 | Repo | Multi-repo / Monorepo | **Monorepo 1 GitHub repo** | Contract, tokens, docs ở 1 chỗ; 1 PR đổi cả 3 phía |
| D25 | Dữ liệu từ điển | Crawl Oxford/Cambridge / LLM sinh / nguồn mở | **Wiktionary (kaikki.org) + Tatoeba + CMUdict + NGSL, import 1 lần vào Postgres, trim theo tần suất** | Hợp pháp (CC BY-SA / CC BY), offline, không tốn token LLM |
| D26 | TTS | ElevenLabs / OpenAI TTS / Piper | **Piper self-host pre-render → R2 (bulk); API TTS chỉ cho on-demand** | Chi phí ~0, audio cache vĩnh viễn |
| D27 | iOS CI | GitHub Actions macOS / Xcode Cloud | **Xcode Cloud (25 giờ/tháng free)** | GitHub macOS runner tính x10 phút, 2000 phút = 200 phút thật |
| D28 | Web token | localStorage / memory + cookie | **Access token trong memory; refresh token là httpOnly cookie scope `api.` path `/v1/auth/refresh`** | Không lộ refresh cho XSS; same-site nên cookie hoạt động |

---

## 2. Nguyên tắc bất biến

1. **Contract-first**: đổi API = sửa `openapi.yaml` trước → `make gen` → mới sửa code. CI job `contract-drift` fail nếu code sinh ra khác code đã commit.
2. **Measure-first**: mọi claim "nhanh/tối ưu" phải có file trong `docs/evidence/`. Không có số = chưa xong.
3. **Boring tech**: chỉ thêm dependency khi một mục checklist yêu cầu nó.
4. **Server là source of truth**, client là cache + queue offline. Mọi ghi từ client đều idempotent (`client_review_id`, `Idempotency-Key`).
5. **Mọi query đụng dữ liệu user phải có `user_id`** (không có RLS, kỷ luật bằng code + test tĩnh, xem 03-BE-A10).
6. **AI viết, bạn review**: code AI sinh phải qua checklist tương ứng trước khi merge. Mục `[M]` bạn tự check.
7. **Một nơi cho một việc**: error format (Problem Details), pagination (cursor), ID (UUIDv7), thời gian (RFC 3339 UTC). Không có ngoại lệ "chỗ này làm khác cho tiện".

---

## 3. Monorepo và file output kỳ vọng

```
englishapp/
├── contracts/
│   ├── openapi.yaml                 # D02 source of truth API
│   ├── tokens/tokens.json           # D21 design tokens (DTCG format)
│   ├── tokens/style-dictionary.config.mjs
│   ├── events.yaml                  # analytics events schema
│   ├── fixtures/fsrs_cases.json     # parity test FSRS (server sinh, client verify)
│   └── README.md                    # cách chạy `make gen`
├── backend/                         # Go
│   ├── cmd/app/main.go              # subcommands: api | worker | migrate | import
│   ├── internal/
│   │   ├── httpapi/                 # server impl của StrictServerInterface, middleware, problem.go
│   │   ├── auth/ user/ dictionary/ deck/ study/ billing/ admin/ stats/
│   │   ├── platform/{config,db,log,otel,mail,storage,jobs,ratelimit}
│   │   └── gen/{api,db}             # oapi-codegen + sqlc output (COMMIT, không sửa tay)
│   ├── db/migrations/*.sql          # goose
│   ├── db/queries/*.sql             # sqlc
│   ├── loadtest/k6/*.js
│   ├── tools/import-dictionary/     # ETL Wiktionary/Tatoeba/NGSL
│   ├── sqlc.yaml  oapi-codegen.yaml  .golangci.yml  Dockerfile  Makefile  .env.example
├── ios/
│   ├── App/                         # Xcode project chỉ chứa App target, Assets, entitlements, xcconfig
│   └── Packages/
│       ├── APIClient/               # generated từ openapi.yaml + middleware auth/retry
│       ├── DesignSystem/            # Tokens.swift generated + components + previews
│       ├── Core/                    # models, GRDB, FSRS, SyncEngine, Keychain
│       └── Features/{Onboarding,Auth,Decks,Study,Stats,Paywall,Settings}
├── web/                             # Next.js 16 static export
│   ├── app/(marketing)/  app/admin/  app/legal/
│   ├── src/{api,design-system,features,lib}
│   ├── public/{_headers,_redirects,.well-known/}
│   └── lighthouserc.json  playwright.config.ts  next.config.ts
├── infra/
│   ├── compose/{docker-compose.yml,docker-compose.staging.yml,Caddyfile,.env.example}
│   ├── scripts/{bootstrap-vps.sh,deploy.sh,backup.sh,restore-drill.sh,cf-ip-allowlist.sh}
│   ├── grafana/{dashboards/*.json,alerts/*.yaml}
│   └── k6/ (dùng chung backend/loadtest)
├── .github/workflows/{contract.yml,backend.yml,web.yml,deploy.yml,nightly-loadtest.yml}
└── docs/
    ├── 00..03 *.md  checklists/  adr/  evidence/  product/  security/  ops/runbooks/  reviews/
```

---

## 4. Phòng ban ảo và deliverable

| Vai | Sở hữu | Deliverable chính | Checklist |
|---|---|---|---|
| Product Owner | Niche, PRD, pricing, KPI | `docs/product/PRD.md`, `pricing.md` | 06 |
| Solution Architect | Decision log, contracts, ADR | `01-CONTRACTS.md`, `contracts/*`, `docs/adr/*` | 01–06 |
| Backend Lead | Go service, auth, FSRS, jobs | `backend/*` | 03 |
| Data Engineer | ETL từ điển, audio, seed decks, chất lượng dữ liệu | `backend/tools/import-dictionary`, `docs/evidence/db/import-report.md` | 04 |
| DBA | Schema, index, migration, backup | `backend/db/*`, `docs/db/ERD.md` | 04 |
| iOS Lead | App, offline, IAP, TestFlight | `ios/*` | 01 |
| Web Lead | Landing, admin, SEO | `web/*` | 02 |
| DevOps/SRE | VPS, compose, CI/CD, monitoring, runbooks | `infra/*`, `.github/*`, `docs/ops/runbooks/*` | 05 |
| Security | Threat model, hardening, scans, secret rotation | `docs/security/*` | 03, 05 |
| QA | Test strategy, contract test, load test, evidence | `docs/evidence/*` | 03 (03-MEASUREMENT) |
| Design | Tokens, components, a11y | `contracts/tokens`, `02-DESIGN-SYSTEM.md` | 02-DS |
| Legal/Compliance | Privacy, Terms, licenses, App Review | `web/app/legal/*`, `docs/legal/*` | 06 |
| Growth | ASO, analytics, channels | `docs/growth/*`, PostHog dashboards | 06 |

---

## 5. Lộ trình theo Phase

Mỗi task: `ID | Việc | Input | Output (file/artifact) | Definition of Done`.

### Phase 0 — Foundation (chưa viết code sản phẩm)

Mục tiêu: chốt hướng, dựng khung để 3 phía không lệch nhau ngay từ commit đầu.

| ID | Việc | Input | Output | DoD |
|---|---|---|---|---|
| P0-01 | Chọn niche và viết PRD 1 trang | Phỏng vấn 10 người mục tiêu | `docs/product/PRD.md` (persona, job-to-be-done, 3 core loop, non-goals, 1 success metric) | 10 người xác nhận "tôi sẽ thử" |
| P0-02 | Đặt tên, check trademark/domain/App Store/social handle | — | `docs/product/naming.md` | Domain đã mua, tên chưa bị trùng trên App Store |
| P0-03 | Mở account: GitHub, Apple Developer, Cloudflare (domain, R2, Pages), Neon, Sentry, PostHog, Resend, RevenueCat, Grafana Cloud, UptimeRobot, healthchecks.io, Hetzner/Oracle | Thẻ, email `admin@domain` | `docs/ops/accounts.md` (KHÔNG chứa secret; secret vào password manager) | Đăng nhập được tất cả, 2FA bật |
| P0-04 | Monorepo skeleton, branch protection, CODEOWNERS, conventional commits, `.editorconfig` | — | Repo | PR không merge được nếu CI đỏ |
| P0-05 | Contract v0.1: toàn bộ endpoint MUST trong `01-CONTRACTS.md` | 01-CONTRACTS | `contracts/openapi.yaml` | `redocly lint` 0 error; `make gen` chạy được 3 phía và compile |
| P0-06 | Tokens v0.1 + Style Dictionary | 02-DESIGN-SYSTEM | `tokens.json`, `Tokens.swift`, `tokens.css` | Build ra 2 file, màu đạt contrast theo bảng |
| P0-07 | ERD v0.1 + migration 0001 | 04 checklist | `docs/db/ERD.md`, `backend/db/migrations/0001_init.sql` | `squawk` 0 error, migrate up/down chạy trên Neon branch |
| P0-08 | Landing "coming soon" + waitlist (Resend) trên Cloudflare Pages | Tokens | `web/` tối thiểu | Live trên domain thật HTTPS; số đăng ký ≥ ngưỡng bạn đặt (gợi ý 30) trước khi build app |
| P0-09 | ADR 0001–0006 ghi lại D01–D28 theo nhóm | Decision log | `docs/adr/*.md` | Mỗi ADR có Context/Decision/Consequences |
| P0-10 | Threat model 1 trang (STRIDE rút gọn) | Contracts | `docs/security/threat-model.md` | Liệt kê ≥ 10 threat với mitigation trỏ tới checklist ID |
| P0-11 | FSRS fixtures | go-fsrs | `contracts/fixtures/fsrs_cases.json` (≥ 200 case: state, rating, expected interval/stability/difficulty) | File sinh bằng script, có version |

**Exit Phase 0**: codegen 3 phía compile; landing live; waitlist đạt ngưỡng; ADR đủ; ERD đã review.

### Phase 1 — Backend core + Data pipeline

| ID | Việc | Input | Output | DoD |
|---|---|---|---|---|
| BE-01 | Skeleton: config, slog JSON, graceful shutdown, `/healthz` `/readyz` `/v1/version`, middleware chain | 03 checklist A03 | `cmd/app`, `internal/platform/*`, `internal/httpapi/middleware.go` | `curl /readyz` 200, log JSON có `request_id` |
| BE-02 | DB platform: pgx pool (max 10), goose embed, sqlc, testcontainers harness | 04 | `internal/platform/db`, `Makefile` targets | `make test-integration` xanh |
| BE-03 | Auth: Apple identity token verify, Email OTP, JWT EdDSA + `kid`, refresh rotation + reuse detection, logout, delete account (soft → job hard delete) | Contracts §2.6 | `internal/auth/*` | Test: reuse refresh → cả family bị revoke |
| BE-04 | `/v1/me` GET/PATCH/DELETE | Contracts | `internal/user` | |
| BE-05 | Dictionary search (pg_trgm + prefix) + entry detail | 04 | `internal/dictionary` | p95 < 15 ms trên 100k entries |
| DATA-01 | ETL Wiktionary (kaikki jsonl) → `dictionary_entries` | kaikki dump | `tools/import-dictionary`, `docs/evidence/db/import-report.md` | Số entry, % có IPA, % có ví dụ, duplicate = 0 |
| DATA-02 | ETL Tatoeba eng–vie → `example_sentences` | Tatoeba CSV | như trên | Mỗi câu giữ `source_id` để attribution |
| DATA-03 | Tần suất NGSL/COCA → `freq_rank`, `cefr` | list | | 100% top 5000 có rank |
| DATA-04 | Audio: Piper → opus/mp3 → R2, ghi `audio_url` (top 5000 trước) | DATA-03 | bucket `cdn.domain/audio/{id}.opus` | 100% top 5000 có audio, size trung bình < 20 KB |
| DATA-05 | Seed decks hệ thống: NGSL 1000, IELTS core, theo CEFR | DATA-03 | migration seed hoặc import script | Deck hiển thị đúng trong API |
| DATA-06 | Đo dung lượng DB sau import; nếu > 400 MB thì trim hoặc tách content sang SQLite pack trên R2 | DATA-01..05 | Quyết định ghi ADR | Fit free tier hoặc ADR chuyển plan |
| BE-06 | Decks/Cards CRUD, ownership, public decks, optimistic `version` | Contracts | `internal/deck` | 403 test cho deck người khác |
| BE-07 | Study: queue query, FSRS (`go-fsrs`), batch review idempotent, `review_logs` append-only | D20, fixtures | `internal/study` | Fixtures 100% pass; gửi trùng `client_review_id` → 200 không ghi đôi |
| BE-08 | Stats daily rollup (River job) + `/v1/study/stats` | BE-07 | `internal/stats` | Rollup chạy đúng timezone user |
| BE-09 | Billing: RevenueCat webhook, `entitlements`, `/v1/me/entitlements`, middleware `entitlement_required` | D16 | `internal/billing` | Replay cùng event id → không đổi state |
| BE-10 | Admin endpoints + role + audit log | Contracts §4 Admin | `internal/admin` | Mọi admin call có dòng trong `admin_audit_logs` |
| BE-11 | Rate limit per IP/user/OTP | 03 A15 | `internal/platform/ratelimit` | Header `RateLimit-*` đúng, 429 đúng format |
| BE-12 | OTel + Prometheus `/metrics` + Sentry | 03 O01–O03 | `internal/platform/otel` | Trace 1 request thấy span DB |
| BE-13 | Tests: unit ≥ 80% domain, integration mọi endpoint MUST, schemathesis | 03 T01–T03 | CI xanh | |
| BE-14 | Dockerfile distroless non-root multi-arch | 03 B03 | image < 25 MB | `trivy` 0 HIGH |
| BE-15 | CI `backend.yml`, `contract.yml` | 03 B04 | workflows | PR có drift → fail |
| OPS-01 | `bootstrap-vps.sh` | 05 A02 | script | Chạy 2 lần không lỗi (idempotent) |
| OPS-02 | Compose staging + Caddy TLS + `deploy.sh` | 05 A03–A08 | `infra/compose/*` | `api-staging.domain/readyz` 200 qua HTTPS |
| OPS-03 | Contract test trên staging | BE-13 | `docs/evidence/backend/schemathesis-*.txt` | 0 failure |

**Exit Phase 1**: mọi endpoint MUST chạy trên staging; schemathesis 0 fail; k6 baseline có trong evidence; checklist 03 và 04 đạt toàn bộ mục Bare Minimum.

### Phase 2 — iOS MVP

| ID | Việc | Input | Output | DoD |
|---|---|---|---|---|
| IOS-01 | Xcode project + SPM packages + SwiftLint/swift-format + Swift 6 strict + xcconfig 3 env | 01 A01–A02 | `ios/` | 0 concurrency warning |
| IOS-02 | DesignSystem package: tokens generated + 8 core + 3 product components + previews | 02-DS | `Packages/DesignSystem` | Preview catalog compile |
| IOS-03 | APIClient package: generated + middleware (auth, refresh, retry, request-id, metrics) | openapi.yaml | `Packages/APIClient` | 401 → refresh → retry 1 lần test pass |
| IOS-04 | Core: GRDB schema mirror + migrations + `PendingReview` queue + `SyncState` | 04 mirror | `Packages/Core/Database` | Migration test từ v1 → head |
| IOS-05 | Auth flow: SIWA, Email OTP, Keychain, session | Contracts | `Features/Auth` | Token không xuất hiện trong UserDefaults/log |
| IOS-06 | Onboarding: level, daily goal, notification permission đúng thời điểm | PRD | `Features/Onboarding` | Analytics `onboarding_step` bắn đủ |
| IOS-07 | Decks: browse public, my decks, detail, thêm card từ dictionary | Contracts | `Features/Decks` | List 1000 item cuộn không hitch |
| IOS-08 | Study session: FlashCard, audio, RatingBar có preview interval, undo, summary; **hoạt động offline hoàn toàn** | FSRS Swift + fixtures | `Features/Study` | Airplane mode học 50 thẻ OK `[M]` |
| IOS-09 | SyncEngine: push pending reviews batch idempotent, pull queue, conflict server-wins, `NWPathMonitor` | Contracts §4 study | `Packages/Core/Sync` | Kill app giữa sync không mất/đôi review |
| IOS-10 | Stats: streak, Swift Charts reviews/day, retention | `/v1/study/stats` | `Features/Stats` | |
| IOS-11 | Paywall + StoreKit 2 + RevenueCat SDK + restore + gating | D16, 06 A03–A04 | `Features/Paywall` | Sandbox mua/huỷ/restore đúng |
| IOS-12 | Settings: account, delete account, reminders, licenses, privacy/terms | 06 L03, L05 | `Features/Settings` | Delete xoá Keychain + DB local |
| IOS-13 | Local notification nhắc học hằng ngày (MUST); APNs token (OPT) | | | |
| IOS-14 | Sentry + MetricKit subscriber | 01 P01 | `Packages/Core/Observability` | dSYM upload tự động |
| IOS-15 | Tests: Swift Testing (FSRS parity, sync, GRDB), XCUITest smoke, Performance test plan + baseline | 01 T01–T06 | `.xctestplan`, `.xcbaseline` | Xcode Cloud xanh |
| IOS-16 | Accessibility pass | 01 X01–X07 | Accessibility Inspector audit 0 issue | `[M]` VoiceOver walk |
| IOS-17 | Xcode Cloud workflows: PR test, main → TestFlight internal | D27 | | |
| IOS-18 | TestFlight external 20–50 người thật | | `docs/evidence/ios/testflight-round-1.md` | Crash-free ≥ 99.5%, ≥ 10 feedback |

**Exit Phase 2**: TestFlight external chạy 1 vòng; checklist 01 Bare Minimum PASS; evidence launch/memory/hitch/size có số.

### Phase 3 — Web landing + Admin (có thể chạy song song cuối Phase 2)

| ID | Việc | Input | Output | DoD |
|---|---|---|---|---|
| WEB-01 | Next.js 16 static export, TS strict, Tailwind v4 + `tokens.css`, shadcn/ui, Inter (latin+vietnamese) | 02-DS | `web/` | `next build` 0 warning |
| WEB-02 | Landing: hero, features, pricing, FAQ, download CTA, SEO đầy đủ, i18n vi/en | PRD | `app/(marketing)` | Lighthouse mobile ≥ 95/95/100/100 |
| WEB-03 | Legal pages: privacy, terms, licenses, support, account-deletion | 06 L01–L05 | `app/legal/*` | URL đưa vào App Store Connect |
| WEB-04 | Admin auth (Email OTP, role admin, cookie refresh) + route guard | D28 | `app/admin/(auth)` | Không có token trong localStorage |
| WEB-05 | Admin: users (search, detail, entitlement), decks/cards CRUD, dictionary search, import job trigger, metrics links | Contracts Admin | `app/admin/*`, `src/features/admin/*` | Bảng 10k dòng virtualized |
| WEB-06 | API client generated + TanStack Query + Problem Details mapping | 01 §2.3 | `src/api/*` | Mọi lỗi hiện toast theo bảng `code` |
| WEB-07 | Tests: Vitest, Playwright E2E, axe | 02 T01–T03 | CI | 0 serious a11y |
| WEB-08 | CI `web.yml`: lint, typecheck, test, build, Playwright, LHCI budgets, deploy Pages (preview/PR) | 02 B03 | workflow | Budget fail → PR đỏ |
| WEB-09 | PostHog (cookieless) + Search Console + sitemap | 06 | | Index OK |

**Exit Phase 3**: LHCI đạt budget trên prod URL; checklist 02 Bare Minimum PASS.

### Phase 4 — Hardening, đo lường, submit

| ID | Việc | Output | DoD |
|---|---|---|---|
| QA-01 | Chạy toàn bộ tools trong `03-MEASUREMENT-TOOLS.md`, điền scorecard | `docs/evidence/SCORECARD.md` + file thô | Mọi ô có số + commit hash |
| SEC-01 | Lynis, Trivy, ssh-audit, SSL Labs, securityheaders, schemathesis fuzz, ZAP baseline, nmap từ ngoài | `docs/evidence/server/*`, `docs/evidence/domain/*` | Đạt target checklist 05/06 |
| OPS-04 | Drill: restore backup, tắt api → alert, rollback | `docs/ops/runbooks/*`, evidence | Mỗi drill có timestamp và kết quả |
| LEGAL-01 | App privacy labels, review notes + reviewer account, screenshots, ASO metadata | App Store Connect | Checklist 06 A01–A10 |
| LEGAL-02 | Submit; xử lý reject theo runbook | | Approved |
| GROWTH-01 | Launch plan + dashboards PostHog (activation, retention, paywall) | `docs/growth/launch.md` | Dashboard có dữ liệu thật từ TestFlight |

**Exit Phase 4**: App Store approved; scorecard không còn ô FAIL ở Bare Minimum.

### Phase 5 — Vận hành và tăng trưởng (lặp)

- Nhịp tuần: đọc Sentry, PostHog retention, RevenueCat, chi phí; sửa top 3 vấn đề.
- Nhịp tháng: `docs/reviews/YYYY-MM.md` (KPI, incident, chi phí, quyết định), restore drill, rotate secret theo lịch, chạy lại scorecard.
- Roadmap Perfect tier theo thứ tự tác động: dark mode → FSRS optimizer per-user → dictionary SQLite pack offline → pronunciation scoring → Android (RN/Expo hoặc Kotlin) → web app học.

---

## 6. Bare Minimum vs Perfect (tổng hợp; chi tiết trong từng checklist)

| Phần | Bare Minimum (gate để ship) | Perfect (làm sau, có thứ tự) |
|---|---|---|
| iOS | Swift 6 strict, packages, offline study, SIWA+OTP, IAP, Sentry, launch < 400 ms, crash-free ≥ 99.5%, VoiceOver + Dynamic Type | Snapshot tests, cert pinning, widget, Live Activity streak, Siri Shortcut, iPad layout, dark mode |
| Web | Static export, LCP ≤ 2.0 s lab, Lighthouse ≥ 95, legal pages, admin CRUD, cookie refresh | Enforcing CSP với hash, visual regression, Storybook, web app học |
| Backend | Contract-first, stdlib + sqlc, auth đầy đủ, FSRS parity, idempotent, p95 < 120 ms @200 VU, `-race` clean, govulncheck clean | 2 replicas zero-downtime, OTel 100%, FSRS optimizer job, data export endpoint |
| DB | ERD, index cho hot path, migration expand/contract, nightly dump → R2 encrypted, restore drill pass | Partition `review_logs`, Neon branch per PR, read replica cho admin analytics |
| Server | VPS hardened (Lynis ≥ 75), Caddy TLS A+, compose read-only containers, alerts, runbooks | Cloudflare IP allowlist, Tailscale SSH, SOPS secrets, blue/green |
| Domain/Ops | DNSSEC, SPF/DKIM/DMARC, privacy/terms/licenses, account deletion, App Review pass, PostHog retention | DMARC reject, status page, data export self-serve, Search Ads |

---

## 7. Rủi ro và cách né

| Rủi ro | Xác suất | Tác động | Né |
|---|---|---|---|
| Không ai dùng (vấn đề thị trường) | Cao | Chết dự án | P0-01, P0-08 gate trước khi code |
| 3 phía lệch contract | Cao nếu không kỷ luật | Mất tuần | D02 + CI drift |
| Free tier đổi/cắt | Trung bình | Chi phí bất ngờ | Mọi thành phần có fallback trả tiền ghi trong D07/D08; export data dễ (Postgres, S3 API) |
| Neon storage 0.5 GB không đủ cho từ điển | Trung bình | Phải trả 19 USD/tháng | DATA-06 đo sớm, trim, hoặc SQLite pack |
| App Review reject (IAP, SIWA, account deletion, privacy) | Trung bình | Trễ launch | Checklist 06 mục A, làm từ Phase 2 |
| Wiktionary CC BY-SA share-alike | Thấp | Pháp lý | Attribution page; dữ liệu content derived giữ CC BY-SA; code app không ảnh hưởng |
| VPS single point of failure | Chắc chắn xảy ra lúc nào đó | Downtime | Stateless api + Neon managed; deploy lại từ image < 10 phút; runbook |
| Chi phí thời gian Swift chậm hơn RN | Chắc chắn | Trễ | Chấp nhận có chủ đích (mục tiêu năng lực). Không thêm Android trước khi có doanh thu |
| Burnout solo | Cao | Bỏ dở | Exit criteria nhỏ, ship TestFlight sớm, nhịp review tháng |

---

## 8. Chi phí dự kiến (năm đầu)

| Khoản | Free tier | Trả tiền khi |
|---|---|---|
| Apple Developer | — | 99 USD/năm (bắt buộc) |
| Domain `.com` | — | ~10 USD/năm |
| VPS | Oracle Always Free | Hetzner ~4–6 EUR/tháng nếu Oracle không đăng ký được |
| Neon | Free (≈0.5 GB, kiểm tra lúc đăng ký) | Launch 19 USD/tháng khi > storage hoặc cần PITR dài |
| R2 | 10 GB, egress free | ~0.015 USD/GB-tháng |
| Cloudflare Pages/DNS/WAF | Free | — |
| Sentry / PostHog / Grafana Cloud / Resend / RevenueCat / UptimeRobot / healthchecks.io | Free tier đủ đến vài nghìn user | Theo usage |
| Xcode Cloud | 25 giờ/tháng | 50 USD/tháng cho 100 giờ |
| **Tổng năm đầu** | | **~110–200 USD** nếu Oracle free; **~170–270 USD** nếu Hetzner |

---

## 9. KPI sản phẩm (đo bằng PostHog + RevenueCat + App Store Connect)

| Chỉ số | Bare | Tốt |
|---|---|---|
| Install → signup | ≥ 50% | ≥ 65% |
| Signup → first study session | ≥ 60% | ≥ 75% |
| D1 / D7 / D30 retention | 35 / 15 / 8 % | 45 / 25 / 12 % |
| Paywall view → trial start | ≥ 8% | ≥ 15% |
| Trial → paid | ≥ 25% | ≥ 40% |
| Crash-free sessions | ≥ 99.5% | ≥ 99.8% |
| App Store product page conversion | ≥ 25% | ≥ 35% |

---

## 10. Thứ tự đọc và làm

1. Đọc `01-CONTRACTS.md` toàn bộ. Nếu có gì chưa rõ trong contract, sửa contract trước, không sửa code.
2. Đọc `02-DESIGN-SYSTEM.md`, tạo `tokens.json`.
3. Làm Phase 0. Chỉ sang Phase 1 khi waitlist đạt ngưỡng.
4. Với mỗi phase: làm → chạy tools trong `03-MEASUREMENT-TOOLS.md` → cho AI check bằng checklist → tự re-check `[M]` → điền scorecard → đóng phase.
