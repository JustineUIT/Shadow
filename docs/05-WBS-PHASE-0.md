# 05 — WBS PHASE 0: Break-down tới mức file và lệnh

> Phase 0 = "Foundation": dựng khung để 3 phía không lệch ngay từ commit đầu, **chưa viết code sản phẩm**.
> Mỗi task lớn (P0-xx) trong `00-MASTER-PLAN.md` §5 được chẻ thành subtask `P0-xx.n`.
> Format subtask: **Input → Hành động (lệnh/file) → Output → DoD (kiểm được bằng lệnh hoặc mắt)**.
> Thứ tự trong file này là thứ tự thực hiện. Task có dấu `⟂` có thể chạy song song với task liền trước.

## Bảng phụ thuộc

```
P0-02 naming ──► P0-03 accounts ──► P0-08 landing live
P0-04 repo layout ──► P0-05 openapi ──► P0-11 fixtures
                 └──► P0-06 tokens  ──► P0-08 landing
                 └──► P0-07 ERD + migration 0001 (cần Neon từ P0-03)
                 └──► P0-09 ADR (có thể viết bất cứ lúc nào, đã có sẵn bản đầu)
P0-01 PRD ──► P0-08 nội dung landing, P0-10 threat model
```

Điểm dừng cứng: **không bắt đầu Phase 1 khi waitlist chưa đạt ngưỡng** (P0-08.6).

---

## P0-04 — Repo layout monorepo (làm đầu tiên, vì mọi path phía sau phụ thuộc)

| ID | Input | Hành động | Output | DoD |
|---|---|---|---|---|
| P0-04.1 | Repo hiện tại | Chạy khối lệnh `git mv` trong `04-CURRENT-STATE.md` §3 | `web/` chứa toàn bộ Next.js | `ls web/app/page.tsx` tồn tại; root không còn `app/` |
| P0-04.2 | — | Tạo root `package.json` (mẫu 04 §3), `pnpm-workspace.yaml` với `packages: ['web', 'contracts/tokens']` | 2 file | `pnpm install` 0 lỗi; `pnpm dev` mở được trang |
| P0-04.3 | — | `web/package.json`: `name` → `@shadow/web`; gỡ `@vercel/analytics`; thêm scripts `lint: next lint`, `typecheck: tsc --noEmit`, `test: vitest run` | file | `pnpm -F @shadow/web typecheck` chạy (được phép đỏ lúc này) |
| P0-04.4 | — | `web/next.config.mjs` → `next.config.ts`: xoá `ignoreBuildErrors`; thêm `output: 'export'`, `trailingSlash: false`, `reactStrictMode: true`; giữ `images.unoptimized` | file | `pnpm -F @shadow/web build` sinh `web/out/index.html` |
| P0-04.5 | — | `web/app/layout.tsx`: bỏ import Analytics; `lang="vi"`; `<html className="bg-background">`; title tạm "Shadow" | file | Build 0 warning |
| P0-04.6 | — | Tạo thư mục rỗng có `.gitkeep`: `contracts/tokens`, `contracts/fixtures`, `backend`, `ios`, `infra/compose`, `infra/scripts`, `infra/grafana`, `.github/workflows`, `docs/{adr,evidence,product,security,ops/runbooks,db,legal,growth,reviews}` | cây thư mục | `tree -L 2 -d` khớp `00-MASTER-PLAN` §3 |
| P0-04.7 | — | `.editorconfig` (utf-8, lf, 2 space; `*.go` tab; `*.swift` 4 space), `CODEOWNERS` (`* @JustineUIT`), `.gitignore` bổ sung theo 04 §1 | 3 file | `git check-ignore backend/bin/x` in ra path |
| P0-04.8 | — | Root `Makefile` với target rỗng có `echo TODO`: `gen`, `lint`, `test`, `evidence`; mỗi target gọi `$(MAKE) -C backend ...`, `pnpm -F ...` | file | `make gen` chạy không lỗi (in TODO) |
| P0-04.9 | GitHub | Branch protection `main`: require PR, require status check `ci`, no force push; conventional commits qua `commitlint` trong `.github/workflows/ci.yml` (job đầu tiên, chỉ lint commit + `pnpm -F @shadow/web typecheck build`) | settings + workflow | Push thẳng lên `main` bị từ chối; PR có check xanh |
| P0-04.10 | — | Commit `chore(repo): monorepo layout (D29)`; ADR `docs/adr/0007-repo-layout.md` | commit | `git log` có commit; preview v0 vẫn chạy nhờ root script `dev` |

**Exit P0-04**: `pnpm dev` ở root mở trang; CI xanh trên PR rỗng; cây thư mục khớp kế hoạch.

---

## P0-01 — Niche + PRD 1 trang ⟂

| ID | Input | Hành động | Output | DoD |
|---|---|---|---|---|
| P0-01.1 | Kinh nghiệm bản thân, quan sát | Điền `docs/product/PRD.md` (template đã tạo): persona, JTBD, 3 core loop, non-goals, 1 success metric | file | Không ô nào còn `TODO` |
| P0-01.2 | PRD | Viết 5 câu hỏi phỏng vấn (không dẫn dắt): thói quen học hiện tại, đau nhất ở đâu, đã trả tiền cho app nào, tần suất, thiết bị | `docs/product/interviews/guide.md` | 5 câu, không câu nào hỏi "bạn có dùng app X không" |
| P0-01.3 | Guide | Phỏng vấn 10 người (15 phút/ người, ghi âm có xin phép); mỗi người 1 file | `docs/product/interviews/NN-<alias>.md` (10 file) | 10 file, mỗi file có mục "Sẽ thử? Y/N + lý do" |
| P0-01.4 | 10 file | Tổng hợp: bảng đau × tần suất; cập nhật PRD nếu lệch | PRD v0.2 | ≥ 7/10 "sẽ thử"; nếu < 7 → sửa niche, lặp P0-01.2 |

**Output must-have** cho phần sau: `PRD.md` §"3 core loop" (dùng cho P0-05 để chọn endpoint `M`), §"persona" (dùng cho P0-08 copy landing), §"non-goals" (dùng để từ chối scope creep).

---

## P0-02 — Tên, trademark, domain, handle ⟂

| ID | Input | Hành động | Output | DoD |
|---|---|---|---|---|
| P0-02.1 | Tên đề xuất "Shadow" + 4 tên dự phòng | Search App Store (VN + US), Google Play, `namecheckr.com` cho handle | `docs/product/naming.md` bảng 5 tên × 6 cột (App Store / Play / .com / .app / IG / X) | Bảng đủ 30 ô |
| P0-02.2 | Bảng | Check trademark: WIPO Global Brand DB, USPTO TESS, IP Vietnam (`ipvietnam.gov.vn`) nhóm 9 + 41 | cột "Trademark" | Không trùng nhóm 9/41 ở VN + US với tên chọn |
| P0-02.3 | Tên chọn | Mua domain tại Cloudflare Registrar: `.com` (ưu tiên) hoặc `.app` (bắt buộc HTTPS, phù hợp); bật auto-renew, lock, WHOIS privacy | domain | `whois <domain>` thấy `clientTransferProhibited` |
| P0-02.4 | Domain | Đăng ký handle IG/X/TikTok/YouTube cùng tên; tạo Apple App Store Connect **app record** giữ tên (cần Apple Dev account từ P0-03) | `naming.md` §"Đã giữ" | Mỗi handle có link |
| P0-02.5 | Tên | Thay `<domain>` trong docs bằng domain thật? **Không.** Giữ placeholder `<domain>` trong docs, chỉ ghi giá trị thật 1 chỗ: `docs/ops/accounts.md` §Domain | — | `grep -rn "<domain>" docs | wc -l` không đổi |

---

## P0-03 — Mở account (13 dịch vụ)

Quy tắc: 1 email quản trị `admin@<domain>` (tạo qua Cloudflare Email Routing → forward về Gmail cá nhân, miễn phí); 2FA bắt buộc; secret vào password manager (Bitwarden/1Password), **không** vào repo. `docs/ops/accounts.md` chỉ ghi: dịch vụ, plan, email dùng, region, ngày tạo, link dashboard, giới hạn free tier.

| ID | Dịch vụ | Việc cụ thể | Ghi vào `accounts.md` | DoD |
|---|---|---|---|---|
| P0-03.1 | Cloudflare | Đã có từ P0-02.3. Bật Email Routing (`admin@`, `support@`, `privacy@`, `dmarc@`); tạo R2 bucket `shadow-cdn` + `shadow-backups` (Singapore/APAC); tạo Pages project `shadow-web` nối GitHub `web/` | Account ID, bucket names, Pages project | Gửi mail tới `admin@` nhận được |
| P0-03.2 | GitHub | Repo đã có. Bật Dependabot, secret scanning, branch protection (P0-04.9). Tạo PAT scope tối thiểu cho Pages deploy nếu cần | — | Settings → Code security tất cả "Enabled" |
| P0-03.3 | Apple Developer | Enroll Individual 99 USD; tạo App ID `com.<org>.shadow`, Sign in with Apple capability, Merchant ID không cần; tạo API key App Store Connect (cho Xcode Cloud/RevenueCat) | Team ID, Bundle ID | App ID xuất hiện trong Certificates, IDs & Profiles |
| P0-03.4 | Neon | Project `shadow`, region **ap-southeast-1**, Postgres 17; branch `main` + `staging`; bật pooled connection; ghi storage limit hiện tại của free tier | Project ID, region, limit | `psql "$DATABASE_URL" -c 'select 1'` |
| P0-03.5 | Sentry | Org + 3 project: `shadow-api` (Go), `shadow-ios` (Apple), `shadow-web` (Next.js) | DSN **không** ghi vào docs | 3 project tồn tại |
| P0-03.6 | PostHog | Project `shadow`, region EU hoặc US (chọn **US** — gần Singapore hơn về latency ingest không quan trọng, chọn theo giá); bật cookieless | Project host | — |
| P0-03.7 | Resend | Domain `<domain>` verified (DKIM/SPF/Return-Path); API key `waitlist` scope sending only | Records đã thêm | Dashboard "Verified"; `dig TXT resend._domainkey.<domain>` có |
| P0-03.8 | RevenueCat | Project `Shadow`, app iOS gắn Bundle ID, App Store Connect API key | Public SDK key **được** ghi (là public) | App hiện "Connected" |
| P0-03.9 | Grafana Cloud | Stack free; ghi Prometheus remote-write URL, Loki URL, Tempo URL | URLs (không token) | — |
| P0-03.10 | UptimeRobot + healthchecks.io | Tạo account, chưa tạo monitor (chưa có gì để theo dõi) | — | — |
| P0-03.11 | VPS | Thử Oracle Always Free ARM (Singapore) 3 lần trong 3 ngày; nếu fail → Hetzner CAX11 (Singapore không có → **Falkenstein** hoặc chấp nhận **Hetzner không có SG**; fallback thật: **Oracle SG** hoặc **Vultr/DigitalOcean SG ~6 USD**). Quyết định ghi ADR 0008 | Provider, IP, region | `ssh` được bằng key, password login tắt |
| P0-03.12 | Password manager | Vault `Shadow` chia folder theo dịch vụ; ghi chú recovery codes 2FA | — | Mọi entry có 2FA recovery |
| P0-03.13 | Kiểm tra chéo | Bảng `accounts.md` 13 dòng đủ cột; không dòng nào chứa chuỗi giống secret | file | `gitleaks detect --source docs` 0 finding |

---

## P0-05 — Contract v0.1 (`contracts/openapi.yaml`)

Input duy nhất: `01-CONTRACTS.md` §2 (quy ước), §3 (domain model), §4 (endpoint, chỉ lấy cột MVP = `M`).

| ID | Hành động | Output | DoD |
|---|---|---|---|
| P0-05.1 | Cài tool: `npm i -g @redocly/cli`; `go install github.com/oapi-codegen/oapi-codegen/v2/cmd/oapi-codegen@latest`; Swift: thêm plugin `swift-openapi-generator` vào package `APIClient` (Phase 2, hiện chỉ cần lint + Go + TS) | tool | `redocly --version` |
| P0-05.2 | Skeleton `openapi.yaml`: `openapi: 3.1.0`, `info`, `servers` (3 env), `security` (bearer), `tags` 8 nhóm theo §4.1–4.8 | file | `redocly lint` 0 error |
| P0-05.3 | `components/schemas`: 8 model từ §3 (User, DictionaryEntry, Deck, Card, CardState, Review, Entitlement, StatsDaily) + `Problem` (§2.3) + `Page<T>` (§2.4 cursor) + kiểu nguyên tử §2.1 (`UUIDv7` pattern, `RFC3339`, `Locale`, `Timezone`) | schemas | Mỗi schema có `required`, `example`, `description`; không `additionalProperties` mặc định (khai báo tường minh `false`) |
| P0-05.4 | `components/parameters`: `Cursor`, `Limit`, `IdempotencyKey`, `XTimezone`, `XClientPlatform`, `XRequestId` (§2.2) | parameters | Reuse qua `$ref`, không copy |
| P0-05.5 | `components/responses`: `Problem400/401/403/404/409/422/429/500` + header `RateLimit-*`, `Retry-After` | responses | Mọi 4xx/5xx `$ref` tới đây |
| P0-05.6 | Paths nhóm System (5) + Auth (5) — copy y nguyên §4.1, §4.2 | paths | `redocly lint` 0 error |
| P0-05.7 | Paths Me (4 M) + Dictionary (2 M) | paths | |
| P0-05.8 | Paths Decks & Cards (10 M) | paths | `operationId` dạng `verbNoun` (`listDecks`, `createDeck`) — oapi-codegen và Swift dùng làm tên hàm |
| P0-05.9 | Paths Study (3 M: queue, reviews, stats) + Billing + Admin (chỉ M) | paths | Đếm `grep -c "operationId" openapi.yaml` = số dòng `M` trong §4 |
| P0-05.10 | `contracts/README.md`: cách `make gen`, quy tắc §9 breaking change, versioning `info.version` semver | file | — |
| P0-05.11 | Codegen Go thử: `backend/oapi-codegen.yaml` (`generate: {std-http-server: true, strict-server: true, models: true}`), `go mod init github.com/JustineUIT/shadow/backend`, `make -C backend gen` | `backend/internal/gen/api/*.go` | `go build ./...` 0 lỗi (server chưa implement, chỉ compile types + interface) |
| P0-05.12 | Codegen TS thử: `pnpm -F @shadow/web add -D openapi-typescript`, script `gen:api: openapi-typescript ../contracts/openapi.yaml -o src/api/schema.d.ts` | `web/src/api/schema.d.ts` | `tsc --noEmit` 0 lỗi |
| P0-05.13 | CI `.github/workflows/contract.yml`: `redocly lint` → `make gen` → `git diff --exit-code` | workflow | PR sửa yaml mà quên gen → đỏ |
| P0-05.14 | Ví dụ `examples` cho mọi request/response 2xx (Swift decode test và schemathesis dùng) | trong yaml | `redocly lint --extends recommended` không warn `no-unused-components` |

**Output must-have**: `openapi.yaml` lint 0 error; Go types compile; TS types compile; CI drift job. **Optional (Phase 2)**: Swift gen.

---

## P0-06 — Tokens v0.1 + Style Dictionary

Input: `02-DESIGN-SYSTEM.md` §2 (cấu trúc 3 lớp), §3 (bảng màu light đã đo contrast), §4 (typography 10 vai), §6.3 (config).

| ID | Hành động | Output | DoD |
|---|---|---|---|
| P0-06.1 | `contracts/tokens/package.json` (`@shadow/tokens`, devDep `style-dictionary@^4`), script `build` | file | `pnpm install` |
| P0-06.2 | `tokens.json` lớp 1 **primitive**: color scale (theo §3, ghi hex), space (4-pt: 0,4,8,12,16,24,32,48,64), radius, font size/line-height/weight (§4), duration, shadow | file DTCG (`$type`, `$value`) | `jq . tokens.json` hợp lệ; số token = con số trong §2.1 |
| P0-06.3 | Lớp 2 **semantic** light: `color.bg.*`, `color.fg.*`, `color.border.*`, `color.action.*`, `color.status.*` trỏ `{color.primitive.xxx}` | trong file | Không semantic nào chứa hex trực tiếp |
| P0-06.4 | Lớp 3 **component** (chỉ những cái §5 cần: button, card, input, rating-bar) | trong file | — |
| P0-06.5 | `style-dictionary.config.mjs`: 2 platform. `css` → `web/src/design-system/tokens.css` (`:root{--color-bg-canvas:...}` + `@theme inline` mapping sang Tailwind v4). `swift` → `ios/Packages/DesignSystem/Sources/Generated/Tokens.swift` (`enum Tokens { enum Color { static let bgCanvas = Color(hex:...) } }`) | config | `pnpm -F @shadow/tokens build` ra 2 file |
| P0-06.6 | `scripts/contrast-check.mjs`: đọc semantic pairs (`fg.*` trên `bg.*`) tính WCAG ratio | script | Mọi pair text ≥ 4.5, large ≥ 3.0; output `docs/evidence/design/<date>-contrast.txt` |
| P0-06.7 | Nối vào web: `web/app/globals.css` chỉ `@import 'tailwindcss'; @import '../src/design-system/tokens.css';` + phần `@theme inline` trỏ biến. Xoá toàn bộ giá trị oklch cũ | file | Trang preview đổi màu theo token; `grep -c oklch globals.css` = 0 |
| P0-06.8 | `Makefile` target `gen-tokens`; thêm vào `contract.yml` bước build tokens + `git diff --exit-code` | CI | Sửa tokens.json quên build → đỏ |

**Output must-have**: `tokens.json`, `tokens.css`, `Tokens.swift` (committed), contrast evidence. **Optional**: dark mode values (§3.1) đã có sẵn trong file dưới `color.semantic.dark`, chưa wire.

---

## P0-07 — ERD v0.1 + migration 0001

Input: `01-CONTRACTS.md` §3, `checklists/04-database-postgres.md` §1 (cấu trúc) + mục A (schema rules).

| ID | Hành động | Output | DoD |
|---|---|---|---|
| P0-07.1 | Cài `goose`, `sqlc`, `squawk` (`npm i -g squawk-cli`), `psql` | tool | version in ra |
| P0-07.2 | `docs/db/ERD.md`: Mermaid `erDiagram` 12 bảng: `users, auth_identities, refresh_tokens, email_otps, dictionary_entries, example_sentences, decks, cards, card_states, review_logs, entitlements, billing_events, stats_daily, admin_audit_logs` (+ River tự tạo bảng riêng) | file | Mỗi bảng có PK UUIDv7, `created_at/updated_at timestamptz`, cột `user_id` cho mọi bảng dữ liệu user |
| P0-07.3 | `backend/db/migrations/0001_init.sql` (goose `-- +goose Up/Down`): extensions `pg_trgm`, bảng theo ERD, FK `on delete cascade` cho dữ liệu user, `on delete restrict` cho content | file | `squawk 0001_init.sql` 0 error |
| P0-07.4 | Index hot path (04 checklist mục I): `card_states(user_id, due_at)`, `review_logs(user_id, reviewed_at desc)`, `dictionary_entries using gin (headword gin_trgm_ops)`, unique `(user_id, client_review_id)`, `refresh_tokens(family_id)` | trong 0001 | Mỗi index có comment "-- phục vụ endpoint X" |
| P0-07.5 | Chạy thật trên Neon branch `dev-p0`: `goose -dir backend/db/migrations postgres "$DATABASE_URL" up` rồi `down` rồi `up` | Neon | 3 lệnh 0 lỗi; `\dt` đủ bảng |
| P0-07.6 | `backend/sqlc.yaml` + 1 query mẫu `db/queries/users.sql` (`GetUserByID`) → `sqlc generate` | `internal/gen/db/` | `go build ./...` |
| P0-07.7 | Ghi `docs/evidence/db/<date>-migration-0001.txt` (output goose + squawk + `\d+` 3 bảng chính) | evidence | file tồn tại, có commit hash |

---

## P0-08 — Landing "coming soon" + waitlist (gate thị trường)

Input: PRD (persona, 1 câu value prop), tokens (P0-06), domain (P0-02), Resend + Pages (P0-03).

| ID | Hành động | Output | DoD |
|---|---|---|---|
| P0-08.1 | `web/app/page.tsx` thay placeholder: hero (1 câu value prop + 1 câu phụ), 3 bullet core loop, form email, footer (privacy link tạm, `support@`) | page | Dùng **chỉ** token semantic; 0 màu hard-code |
| P0-08.2 | Form gửi POST tới **Cloudflare Pages Function** `web/functions/api/waitlist.ts` (static export không có server) → Resend Contacts API (audience `waitlist`) + gửi mail xác nhận | function | `curl -X POST .../api/waitlist -d '{"email":"x@y.z"}'` → 202 |
| P0-08.3 | Chống spam: Cloudflare Turnstile (free) + rate limit rule 5 req/phút/IP trên `/api/waitlist` | CF settings | Gửi 6 lần → 429 |
| P0-08.4 | SEO tối thiểu: `metadata` title/description/og image (GenerateImage 1200×630), `robots.txt`, `sitemap.xml`, `lang="vi"` | files | Lighthouse SEO 100 |
| P0-08.5 | Deploy: Pages project build `pnpm -F @shadow/web build`, output `web/out`; custom domain apex + `www` → apex redirect; `_headers` theo 02-web S01 | live | `curl -I https://<domain>` 200 + HSTS; Lighthouse mobile ≥ 95/95/100/100 → `docs/evidence/web/<date>-lhci-landing/` |
| P0-08.6 | PostHog snippet (cookieless) event `waitlist_submitted`; đặt **ngưỡng** trong `docs/product/PRD.md` §metric (gợi ý 30) | analytics | Dashboard đếm được |
| P0-08.7 | Phân phối: 3 kênh cụ thể (nhóm FB học IELTS, Reddit r/languagelearning, bạn bè) ghi `docs/growth/launch.md` §waitlist | file | Ghi ngày post + số đăng ký sau 7 ngày |

**Exit P0-08**: đạt ngưỡng waitlist. Chưa đạt → sửa copy/kênh, **không** sang Phase 1.

---

## P0-09 — ADR

Bản đầu đã viết sẵn trong `docs/adr/0001–0007` (lần này). Việc còn lại:

| ID | Hành động | DoD |
|---|---|---|
| P0-09.1 | Đọc lại 7 ADR, sửa chỗ nào thực tế khác (ví dụ VPS provider sau P0-03.11 → ADR 0008) | Không ADR nào ở trạng thái `Proposed` khi đóng Phase 0 |
| P0-09.2 | Quy tắc: đổi bất kỳ D-nào → ADR mới `Supersedes 000X`, không sửa ADR cũ | `docs/adr/README.md` ghi quy tắc + index |

---

## P0-10 — Threat model 1 trang

| ID | Hành động | Output | DoD |
|---|---|---|---|
| P0-10.1 | `docs/security/threat-model.md`: sơ đồ data flow (Mermaid) iOS/Web → Cloudflare → Caddy → Go → Neon/R2/Resend/RevenueCat | file | 1 sơ đồ |
| P0-10.2 | Bảng STRIDE rút gọn ≥ 10 threat: token theft, refresh reuse, OTP brute force, IDOR deck, webhook giả, SQLi, secret leak repo, VPS SSH brute, DDoS, supply chain (npm/go/SPM), backup lộ | bảng: Threat / Asset / Mitigation / **Checklist ID** | Mỗi dòng trỏ ≥ 1 ID trong checklist 01–06 (ví dụ refresh reuse → 03-S03) |
| P0-10.3 | Mục "Chấp nhận rủi ro" (không làm ở MVP): cert pinning, HSM, WAF trả tiền | mục | Có lý do từng dòng |

---

## P0-11 — FSRS fixtures

| ID | Hành động | Output | DoD |
|---|---|---|---|
| P0-11.1 | `backend/tools/fsrs-fixtures/main.go` dùng `github.com/open-spaced-repetition/go-fsrs/v3`: sinh 200 case ngẫu nhiên có seed cố định: input (state, stability, difficulty, elapsed_days, rating 1–4, now) → expected (new stability, difficulty, scheduled_days, due) | tool | `go run ./tools/fsrs-fixtures > ../contracts/fixtures/fsrs_cases.json` |
| P0-11.2 | File có header `{ "fsrs_version": "...", "go_fsrs_version": "v3.x", "seed": 42, "generated_at": ..., "cases": [...] }` | json | `jq '.cases | length'` = 200 |
| P0-11.3 | Test Go đọc lại file và so khớp (parity với chính nó — để khi nâng version go-fsrs, test đỏ báo thay đổi) | `internal/study/fsrs_fixtures_test.go` | `go test ./internal/study/...` xanh |
| P0-11.4 | Ghi vào `contracts/README.md` cách Swift test sẽ đọc cùng file (Phase 2, IOS-08) | doc | — |

---

## Exit Phase 0 — bảng kiểm

| Tiêu chí (từ 00 §5) | Lệnh/ bằng chứng | Trạng thái |
|---|---|---|
| Codegen 3 phía compile | `make gen && git diff --exit-code`; Go + TS compile (Swift để Phase 2, ghi rõ) | ☐ |
| Landing live | `curl -I https://<domain>` 200; LHCI evidence | ☐ |
| Waitlist đạt ngưỡng | PostHog/Resend audience count ≥ ngưỡng PRD | ☐ |
| ADR đủ | `ls docs/adr | wc -l` ≥ 8, không `Proposed` | ☐ |
| ERD đã review | `docs/db/ERD.md` + migration 0001 up/down OK trên Neon | ☐ |
| Tokens 2 platform | `tokens.css` + `Tokens.swift` committed, contrast evidence | ☐ |
| Threat model | ≥ 10 dòng trỏ checklist ID | ☐ |
| Fixtures | 200 case, test Go xanh | ☐ |
| SCORECARD Phase 0 | `docs/evidence/SCORECARD.md` cột Phase 0 điền (chỉ ô đo được: LHCI landing, contrast, squawk, redocly) | ☐ |
