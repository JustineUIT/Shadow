# 01 — CONTRACTS: Biên giới giữa các hệ thống

> Mục tiêu duy nhất của file này: không bao giờ có chuyện "iOS gửi field A, Go chờ field B, web hiển thị field C".
> Một nguồn sự thật cho mỗi biên giới → code sinh tự động → CI chặn lệch.

## 1. Bốn nguồn sự thật (Source of Truth)

| Biên giới | File SoT | Sinh ra (KHÔNG sửa tay) | CI gate |
|---|---|---|---|
| Client ↔ API | `contracts/openapi.yaml` (OpenAPI 3.1) | Go: `backend/internal/gen/api/*.go` (oapi-codegen, strict server + types). Swift: `ios/Packages/APIClient` (swift-openapi-generator, build plugin). TS: `web/src/api/schema.d.ts` (openapi-typescript) | `redocly lint`; `make gen && git diff --exit-code`; schemathesis chạy trên staging |
| API ↔ DB | `backend/db/migrations/*.sql` + `backend/db/queries/*.sql` | `backend/internal/gen/db/*.go` (sqlc) | `sqlc vet`; `squawk`; migrate up/down trên CI |
| UI ↔ UI (iOS ↔ web) | `contracts/tokens/tokens.json` (DTCG) | `ios/Packages/DesignSystem/Sources/Generated/Tokens.swift`; `web/src/design-system/tokens.css` | Style Dictionary build + `git diff --exit-code` |
| Client/Web ↔ Analytics | `contracts/events.yaml` | `Events.swift` (enum + typed props); `events.ts` | Script validate: mọi event dùng trong code phải có trong yaml |
| Client ↔ Server FSRS | `contracts/fixtures/fsrs_cases.json` | — (test fixture) | Go test và Swift test cùng đọc file, 100% match |

Lệnh ở root: `make gen` (tất cả), `make gen-go`, `make gen-swift`, `make gen-ts`, `make gen-tokens`, `make check-drift`.

---

## 2. Quy ước chung (áp dụng cho MỌI endpoint, không ngoại lệ)

### 2.1 Kiểu dữ liệu nguyên tử

| Thứ | Quy ước | Ví dụ |
|---|---|---|
| ID | UUID v7, string, lowercase. Sinh ở server (Go `uuid.NewV7()`); client chỉ sinh `client_review_id` và `Idempotency-Key` | `019263b1-7f5e-7c3a-9d2e-1a2b3c4d5e6f` |
| Thời điểm | RFC 3339, UTC, suffix `Z`, độ chính xác ms | `2026-09-10T03:04:05.123Z` |
| Ngày (không giờ) | `YYYY-MM-DD` theo timezone user (client gửi `X-Timezone`) | `2026-09-10` |
| Tiền | integer minor unit + `currency` ISO 4217 | `{"amount": 99000, "currency": "VND"}` |
| Ngôn ngữ | BCP 47 | `vi`, `en` |
| Enum | snake_case string; **client phải chịu được giá trị lạ** (Swift: case `unknown`; TS: không exhaustive switch) | `"state": "relearning"` |
| JSON field | snake_case | `created_at` |
| PATCH semantics | field vắng = không đổi; `null` = xoá giá trị | |
| Số thực | JSON number, không string | `"stability": 12.34` |
| Chuỗi | UTF-8, đã trim; `maxLength` khai báo trong openapi cho mọi string | |

### 2.2 Headers

Request (client → API):

| Header | MUST/OPT | Giá trị |
|---|---|---|
| `Authorization` | MUST với endpoint có khoá | `Bearer <access_jwt>` |
| `X-Client-Platform` | MUST | `ios` \| `web` \| `admin` |
| `X-Client-Version` | MUST | `<semver>+<build>`, ví dụ `1.2.0+45` |
| `X-Request-Id` | OPT (server sinh nếu thiếu) | UUID |
| `Idempotency-Key` | MUST cho `POST /study/reviews`, `POST /decks`, `POST /decks/{id}/cards` | UUID; server giữ 24 h |
| `X-Timezone` | MUST cho `/study/*`, `/me` PATCH; OPT còn lại | IANA `Asia/Ho_Chi_Minh` |
| `Accept-Language` | OPT | `vi-VN,vi;q=0.9,en;q=0.8` |
| `Content-Type` | MUST khi có body | `application/json` |

Response (API → client):

| Header | Khi nào |
|---|---|
| `X-Request-Id` | Luôn (echo hoặc sinh) |
| `Cache-Control` | `no-store` cho dữ liệu user; `public, max-age=3600, stale-while-revalidate=86400` cho `/dictionary/*` và public decks |
| `ETag` | Trên `GET /dictionary/entries/{id}`, `GET /decks/{id}` |
| `RateLimit-Limit`, `RateLimit-Remaining`, `RateLimit-Reset` | Mọi endpoint có rate limit |
| `Retry-After` | 429 và 503 |
| `Deprecation`, `Sunset`, `Link: <...>; rel="successor-version"` | Endpoint sắp bỏ |

### 2.3 Lỗi — RFC 9457 Problem Details (format lỗi DUY NHẤT)

`Content-Type: application/problem+json`

```json
{
  "type": "https://api.<domain>/errors/validation_failed",
  "title": "Validation failed",
  "status": 422,
  "detail": "reviews[2].rating must be between 1 and 4",
  "instance": "/v1/study/reviews",
  "code": "validation_failed",
  "request_id": "019263b1-...",
  "errors": [
    { "field": "reviews[2].rating", "code": "out_of_range", "message": "must be 1..4" }
  ]
}
```

Bảng `code` (client switch trên `code`, KHÔNG parse `detail`):

| HTTP | `code` | Khi nào | Client MUST làm gì |
|---|---|---|---|
| 400 | `bad_request` | JSON hỏng, header thiếu | Báo lỗi, gửi Sentry |
| 401 | `unauthenticated` | Thiếu/hết hạn access token | Refresh → retry đúng 1 lần → nếu vẫn 401 thì logout |
| 401 | `refresh_invalid` | Refresh hết hạn/revoked | Logout, về màn login |
| 401 | `refresh_reused` | Phát hiện dùng lại refresh token | Logout mọi thiết bị, hiện cảnh báo bảo mật |
| 403 | `forbidden` | Không phải owner / thiếu role | Ẩn action, hiện thông báo |
| 403 | `entitlement_required` | Tính năng Pro | Mở paywall với `trigger` = endpoint |
| 404 | `not_found` | | Về màn trước |
| 409 | `conflict` | `version` lệch, unique vi phạm | Reload resource, thử lại |
| 409 | `fsrs_version_mismatch` | Client FSRS major ≠ server | Force update / tắt preview offline |
| 410 | `gone` | Resource đã xoá | |
| 413 | `payload_too_large` | > giới hạn | Chia nhỏ batch |
| 422 | `validation_failed` | | Hiển thị `errors[].field` |
| 426 | `upgrade_required` | Build < `min_ios_build` | Màn force update |
| 429 | `rate_limited` | | Đọc `RateLimit-Reset`, backoff, không retry ngay |
| 500 | `internal` | | Retry 3 lần exp backoff + jitter (chỉ với idempotent), Sentry |
| 503 | `unavailable` | Maintenance / DB down | Chờ `Retry-After` |

### 2.4 Phân trang — cursor-based (không offset)

- Request: `?limit=50&cursor=<opaque>`; `limit` default 50, max 200.
- Response: `{ "data": [...], "next_cursor": "<opaque>" | null, "has_more": true }`.
- Sort mặc định `(created_at DESC, id DESC)`; cursor = base64url(`created_at|id`), server ký HMAC để chống sửa.

### 2.5 Versioning và tương thích

- Prefix `/v1`. Breaking change → `/v2` chạy song song tối thiểu 90 ngày.
- Không bump khi: thêm field optional, thêm enum value, thêm endpoint. Client MUST bỏ qua field lạ.
- `GET /v1/version` → `{ "api_version": "1.4.0", "fsrs_version": "5.0", "min_ios_build": 45, "min_web_build": "2026.09.01" }`. iOS build < `min_ios_build` → màn force update (client tự check lúc launch; server cũng trả 426 với `X-Client-Version` cũ).

### 2.6 Auth token

| Thứ | Quy định |
|---|---|
| Access token | JWT, alg `EdDSA` (Ed25519), header `kid`, TTL 15 phút. Claims: `sub` (user id), `sid` (session/family id), `role` (`user`\|`admin`), `ent` (snapshot entitlements, chỉ để UI, server không tin), `iat`, `exp`, `iss=https://api.<domain>`, `aud=englishapp` |
| Refresh token | Opaque 32 bytes base64url, TTL 30 ngày, lưu SHA-256, **rotation mỗi lần dùng**, reuse → revoke cả family (`sid`) |
| iOS | Refresh token trong body response, lưu Keychain |
| Web/Admin | `X-Client-Platform: web|admin` → server set cookie `rt` httpOnly, Secure, SameSite=Strict, Domain=`api.<domain>`, Path=`/v1/auth/refresh`; body không chứa refresh. Endpoint refresh yêu cầu header `X-Client-Platform` (chống CSRF form) và check `Origin` thuộc allowlist |
| JWKS | `GET /.well-known/jwks.json` public, 2 key song song khi rotate |

---

## 3. Domain model (canonical)

`M` = luôn có trong response; `O` = có thể vắng/null.

### 3.1 User

| field | type | M/O | ghi chú |
|---|---|---|---|
| id | uuid | M | |
| email | string \| null | M | null khi Apple ẩn email và chưa bổ sung |
| display_name | string (1..60) | M | default từ Apple hoặc "Learner" |
| locale | `vi` \| `en` | M | |
| timezone | IANA string | M | |
| level | `a1` \| `a2` \| `b1` \| `b2` \| `c1` | M | từ onboarding |
| daily_goal | int 5..200 | M | card mới/ngày |
| role | `user` \| `admin` | M | |
| created_at | datetime | M | |
| avatar_url | string | O | |
| deleted_at | datetime | O | soft delete, hard delete sau 30 ngày |

### 3.2 DictionaryEntry (content, read-only)

| field | type | M/O |
|---|---|---|
| id | uuid | M |
| lemma | string | M |
| pos | `noun`\|`verb`\|`adj`\|`adv`\|`prep`\|`conj`\|`pron`\|`det`\|`interj`\|`phrase`\|`other` | M |
| ipa_us | string | O |
| ipa_uk | string | O |
| senses | array of `{ definition_en: string, definition_vi: string?, examples: [{ en: string, vi: string?, source: string }] }` | M (có thể rỗng) |
| freq_rank | int | O |
| cefr | `a1..c2` | O |
| audio_url | string (https, cdn.) | O |
| source | string (`wiktionary`) | M |
| license | string (`CC BY-SA 4.0`) | M |

### 3.3 Deck

| field | type | M/O |
|---|---|---|
| id | uuid | M |
| owner_id | uuid \| null | M (null = system deck) |
| title | string 1..80 | M |
| description | string ≤ 500 | O |
| visibility | `private`\|`public`\|`system` | M |
| card_count | int | M |
| cefr | string | O |
| cover_url | string | O |
| version | int | M (optimistic lock, gửi lại khi PATCH) |
| created_at, updated_at | datetime | M |
| progress | `{ new: int, learning: int, review: int, due_today: int }` | O (chỉ khi có auth) |

### 3.4 Card

| field | type | M/O |
|---|---|---|
| id | uuid | M |
| deck_id | uuid | M |
| entry_id | uuid \| null | M |
| front | string 1..200 | M |
| back | string 1..2000 | M |
| example | string ≤ 500 | O |
| audio_url | string | O |
| position | int | M |
| created_at, updated_at | datetime | M |

### 3.5 CardState (FSRS, per user)

| field | type | M/O |
|---|---|---|
| card_id | uuid | M |
| state | `new`\|`learning`\|`review`\|`relearning` | M |
| due | datetime | M |
| stability | number | M |
| difficulty | number | M |
| elapsed_days, scheduled_days | int | M |
| reps, lapses | int | M |
| last_review | datetime \| null | M |
| suspended | bool | M |

### 3.6 Review (write model)

| field | type | M/O | ghi chú |
|---|---|---|---|
| client_review_id | uuid | M | idempotency key per review, unique per user |
| card_id | uuid | M | |
| rating | int 1..4 | M | 1 again, 2 hard, 3 good, 4 easy |
| reviewed_at | datetime | M | thời điểm thật trên client (offline) |
| duration_ms | int | O | |
| fsrs_version | string | M | major phải khớp server |

### 3.7 Entitlement

| field | type | M/O |
|---|---|---|
| product | `pro` | M |
| active | bool | M |
| source | `apple`\|`promo`\|`admin` | M |
| expires_at | datetime \| null | M |
| will_renew | bool | M |
| in_grace_period | bool | M |

### 3.8 StatsDaily

`{ date: "YYYY-MM-DD", reviews: int, new_cards: int, minutes: number, retention_rate: number 0..1, streak: int }`

---

## 4. Endpoint catalogue

Cột **MVP**: `M` = phải có để ship; `O` = sau.

### 4.1 System

| Method Path | Auth | Input | Output | MVP |
|---|---|---|---|---|
| `GET /healthz` | none | — | `200 ok` (process) | M |
| `GET /readyz` | none | — | 200 nếu DB ping + migration head; 503 nếu không | M |
| `GET /v1/version` | none | — | §2.5 | M |
| `GET /.well-known/jwks.json` | none | — | JWKS | M |
| `GET /v1/openapi.yaml` | none | — | spec đang chạy | M |

### 4.2 Auth

| Method Path | Auth | Input (body) | Output | Lỗi đặc thù | MVP |
|---|---|---|---|---|---|
| `POST /v1/auth/apple` | none | `{ identity_token: string M, raw_nonce: string M, full_name: {given, family} O }` | `200 { access_token, expires_in, refresh_token (iOS only), user, is_new_user }` | 401 `unauthenticated` (token Apple sai), 422 | M |
| `POST /v1/auth/email/otp/request` | none | `{ email: string M }` | `202 { retry_after_seconds }` (luôn 202 kể cả email không tồn tại) | 429 | M |
| `POST /v1/auth/email/otp/verify` | none | `{ email M, code: string(6) M, display_name O }` | như apple | 401 `otp_invalid`, 429 | M |
| `POST /v1/auth/refresh` | refresh (body iOS / cookie web) | iOS: `{ refresh_token }`; web: cookie | `200 { access_token, expires_in, refresh_token (iOS) }` | 401 `refresh_invalid`, `refresh_reused` | M |
| `POST /v1/auth/logout` | bearer | `{ all_devices: bool O }` | 204 | | M |

### 4.3 Me

| Method Path | Auth | Input | Output | MVP |
|---|---|---|---|---|
| `GET /v1/me` | bearer | — | User + `entitlements[]` | M |
| `PATCH /v1/me` | bearer | `{ display_name?, locale?, timezone?, level?, daily_goal?, email? }` | User | M |
| `DELETE /v1/me` | bearer | `{ confirm: "DELETE" }` | 202 (soft delete, hard sau 30 ngày; revoke mọi token) | M |
| `GET /v1/me/entitlements` | bearer | — | `{ data: Entitlement[] }` | M |
| `GET /v1/me/export` | bearer | — | 202 + email link (job) | O |
| `POST /v1/me/devices` | bearer | `{ apns_token, platform: ios, app_version }` | 204 | O |

### 4.4 Dictionary (public, cache được)

| Method Path | Auth | Input | Output | MVP |
|---|---|---|---|---|
| `GET /v1/dictionary/search` | optional | `?q=string(1..64) M&limit=1..50` | `{ data: DictionaryEntry[] (senses rút gọn 1) }` | M |
| `GET /v1/dictionary/entries/{id}` | optional | — | DictionaryEntry đầy đủ | M |
| `GET /v1/dictionary/word-of-day` | optional | `?date=` | DictionaryEntry | O |

### 4.5 Decks & Cards

| Method Path | Auth | Input | Output | Lỗi | MVP |
|---|---|---|---|---|---|
| `GET /v1/decks` | bearer | `?scope=mine\|public\|system&cursor&limit` | page of Deck | | M |
| `POST /v1/decks` | bearer + Idempotency-Key | `{ title M, description O, visibility O=private, cefr O }` | 201 Deck | 422 | M |
| `GET /v1/decks/{id}` | bearer (public/system không cần) | — | Deck | 404, 403 | M |
| `PATCH /v1/decks/{id}` | bearer owner | `{ version M, title?, description?, visibility? }` | Deck | 409 `conflict` | M |
| `DELETE /v1/decks/{id}` | bearer owner | — | 204 | | M |
| `GET /v1/decks/{id}/cards` | như GET deck | `?cursor&limit` | page of Card (+ `state` nếu auth) | | M |
| `POST /v1/decks/{id}/cards` | owner + Idempotency-Key | `{ entry_id O, front M, back M, example O }` (nếu có `entry_id`, server tự điền front/back/audio khi thiếu) | 201 Card | | M |
| `PATCH /v1/cards/{id}` | owner | `{ front?, back?, example?, position? }` | Card | | M |
| `DELETE /v1/cards/{id}` | owner | — | 204 | | M |
| `POST /v1/decks/{id}/clone` | bearer | — | 201 Deck (copy system/public deck về mine) | | M |

### 4.6 Study

| Method Path | Auth | Input | Output | Lỗi | MVP |
|---|---|---|---|---|---|
| `GET /v1/study/queue` | bearer + X-Timezone | `?deck_id O&limit=1..200 (default 50)` | `{ data: [{ card: Card, state: CardState, intervals_preview: { again, hard, good, easy: ISO8601 duration } }], counts: { new, learning, review }, fsrs_version }` | | M |
| `POST /v1/study/reviews` | bearer + Idempotency-Key + X-Timezone | `{ reviews: Review[1..100] }` | `200 { results: [{ client_review_id, status: applied\|duplicate\|rejected, state: CardState?, error: Problem? }] }` | 409 `fsrs_version_mismatch`, 413 | M |
| `POST /v1/study/undo` | bearer | `{ client_review_id }` | CardState trước đó | 410 nếu > 5 phút | O |
| `GET /v1/study/stats` | bearer + X-Timezone | `?from=YYYY-MM-DD&to=` (max 366 ngày) | `{ data: StatsDaily[], totals: {...} }` | | M |
| `GET /v1/study/cards/{id}/history` | bearer | — | review logs của card | | O |

### 4.7 Billing

| Method Path | Auth | Input | Output | MVP |
|---|---|---|---|---|
| `POST /v1/billing/revenuecat/webhook` | `Authorization: Bearer <RC_WEBHOOK_SECRET>` | RevenueCat event JSON | 200 (luôn, sau khi enqueue job; dedupe theo `event.id`) | M |
| `POST /v1/billing/promo/redeem` | bearer | `{ code }` | Entitlement | O |

### 4.8 Admin (role `admin`, mọi call ghi `admin_audit_logs`)

| Method Path | Input | Output | MVP |
|---|---|---|---|
| `GET /v1/admin/users` | `?q&cursor&limit` | page of User + entitlement summary | M |
| `GET /v1/admin/users/{id}` | — | User + stats + entitlements + devices | M |
| `POST /v1/admin/users/{id}/entitlements` | `{ product, expires_at, note }` | Entitlement (source=admin) | M |
| `GET /v1/admin/decks` / `POST` / `PATCH` / `DELETE` | như user nhưng cho system decks | | M |
| `POST /v1/admin/imports/dictionary` | `{ source, dry_run }` | 202 job id | O (MVP chạy CLI) |
| `GET /v1/admin/jobs` | `?state` | River jobs summary | M |
| `GET /v1/admin/metrics/summary` | — | `{ users_total, dau, reviews_today, active_pro, mrr_estimate }` | M |
| `GET /v1/admin/audit-logs` | `?cursor` | page | M |

---

## 5. Ma trận tích hợp (ai gọi ai, bằng gì, hỏng thì sao)

| From → To | Giao thức | Auth | Timeout / Retry | Failure mode phải xử lý |
|---|---|---|---|---|
| iOS → API | HTTPS/2 JSON | Bearer | 10 s; retry 3× exp backoff + jitter chỉ với GET và POST có Idempotency-Key | Offline queue trong GRDB; UI vẫn dùng được |
| Web/Admin → API | HTTPS JSON | Bearer (memory) + cookie refresh | 10 s; retry như iOS | 401 → refresh → retry 1; hiện toast theo `code` |
| API → Postgres (Neon pooled) | pgx, TLS `verify-full` | user `app_rw` | `statement_timeout=5s`, pool max 10, `default_query_exec_mode=cache_describe` | `/readyz` 503, Caddy trả 503 + `Retry-After` |
| Migrate → Postgres (direct endpoint) | pgx | user `app_migrate` | 60 s | Deploy dừng, không switch traffic |
| API/Worker → R2 | S3 API (PutObject, presign) | access key (scoped bucket) | 30 s, retry 3 | Job retry River; `audio_url` null → client dùng TTS local (AVSpeechSynthesizer) fallback |
| Client → `cdn.<domain>` | HTTPS GET public | none (public read) | cache 1 năm immutable | Fallback như trên |
| RevenueCat → API | Webhook POST | shared secret header | RC retry tự động | Idempotent theo `event.id`; luôn 200 sau khi lưu raw |
| API → RevenueCat REST | HTTPS | secret API key | 10 s | Chỉ dùng để reconcile thủ công (admin) |
| API → Apple JWKS | HTTPS GET | none | cache 24 h + ETag; fallback cache cũ 7 ngày | Nếu không fetch được và cache rỗng → 503 cho `/auth/apple` |
| Worker → Resend | HTTPS | API key | River retry 5× | OTP: user thấy "thử lại sau"; log |
| Worker → Piper (local process) | exec/HTTP nội bộ | — | 60 s/file | Job fail → retry, không chặn API |
| Prometheus (Alloy) → API `/metrics` | HTTP nội bộ docker network | none, không expose ra Caddy | scrape 15 s | |
| GitHub Actions → VPS | SSH (deploy key, restricted command) | key | 10 phút | Rollback về `PREV_IMAGE` nếu `/readyz` không 200 trong 60 s |
| Cloudflare → Caddy | HTTPS (Full strict) | Origin cert | | Chỉ nhận từ CF IP ranges (perfect tier) |

---

## 6. Codegen pipeline

| Bước | Công cụ | Lệnh | Output | Commit? |
|---|---|---|---|---|
| Lint spec | `@redocly/cli` | `redocly lint contracts/openapi.yaml` | — | — |
| Go types + strict server | `oapi-codegen` v2 (`std-http`, `strict-server`, `models`, `embedded-spec`) | `go run github.com/oapi-codegen/oapi-codegen/v2/cmd/oapi-codegen -config oapi-codegen.yaml ../contracts/openapi.yaml` | `internal/gen/api/api.gen.go` | Có |
| Go DB | `sqlc` | `sqlc generate` | `internal/gen/db/*.go` | Có |
| Swift client | `swift-openapi-generator` (SPM plugin, `openapi-generator-config.yaml`: `generate: [types, client]`, `accessModifier: public`) | build tự chạy | trong `.build`, không commit | Không |
| TS types | `openapi-typescript` | `openapi-typescript ../contracts/openapi.yaml -o src/api/schema.d.ts` | `schema.d.ts` | Có |
| Tokens | `style-dictionary` v4 | `style-dictionary build -c contracts/tokens/style-dictionary.config.mjs` | `Tokens.swift`, `tokens.css` | Có |
| Events | script nhỏ (`node scripts/gen-events.mjs`) | | `Events.swift`, `events.ts` | Có |
| Drift | Makefile | `make gen && git diff --exit-code -- backend/internal/gen web/src/api ios/Packages/DesignSystem/Sources/Generated` | | CI |

Quy tắc PR: PR đụng `contracts/**` phải kèm generated files và **ít nhất 1 test** thể hiện thay đổi (Go integration test hoặc Swift decode test).

---

## 7. Analytics events (`contracts/events.yaml`)

Quy tắc: không PII trong props; `distinct_id` = user id (uuid), anonymous trước login; mọi event có props chung tự động: `platform`, `app_version`, `locale`.

| event | props (M) | props (O) |
|---|---|---|
| `app_opened` | `cold_start: bool` | `from_notification: bool` |
| `onboarding_step_viewed` | `step: level\|goal\|notifications\|signup` | |
| `signup_completed` | `method: apple\|email` | `is_new_user` |
| `deck_opened` | `deck_id`, `visibility` | |
| `study_session_started` | `deck_id?`, `due_count`, `new_count`, `offline: bool` | |
| `card_reviewed` | `rating 1..4`, `state_before`, `duration_ms`, `offline` | |
| `study_session_completed` | `reviews`, `minutes`, `goal_reached: bool` | |
| `sync_completed` | `pushed`, `pulled`, `duration_ms`, `result: ok\|partial\|failed` | |
| `paywall_viewed` | `trigger: onboarding\|deck_limit\|new_card_limit\|settings\|feature_x` | |
| `purchase_started` / `purchase_completed` / `purchase_failed` | `product: pro_monthly\|pro_yearly`, `trial: bool` | `error_code` |
| `notification_permission` | `granted: bool` | |
| `notification_tapped` | `kind: daily_reminder` | |
| `account_deleted` | `reason?` | |
| `error_shown` | `code` (từ bảng §2.3), `endpoint` | |
| `web_cta_clicked` (web) | `cta: appstore\|waitlist`, `section` | |

---

## 8. Env variables contract

### Backend (`backend/.env.example`, validate lúc boot, fail fast)

| Var | M/O | Mô tả |
|---|---|---|
| `APP_ENV` | M | `dev`\|`staging`\|`prod` |
| `HTTP_ADDR` | M | `:8080` |
| `DATABASE_URL` | M | pooled, `sslmode=verify-full` |
| `DATABASE_URL_DIRECT` | M (migrate) | direct endpoint |
| `JWT_PRIVATE_KEYS` | M | JSON `[{kid, ed25519_pem}]`, key đầu dùng ký |
| `JWT_ISSUER`, `JWT_AUDIENCE` | M | |
| `APPLE_BUNDLE_ID` | M | |
| `RESEND_API_KEY`, `EMAIL_FROM` | M | |
| `R2_ACCOUNT_ID`, `R2_ACCESS_KEY_ID`, `R2_SECRET_ACCESS_KEY`, `R2_BUCKET`, `CDN_BASE_URL` | M | |
| `REVENUECAT_WEBHOOK_SECRET` | M | |
| `REVENUECAT_API_KEY` | O | reconcile |
| `SENTRY_DSN` | O | |
| `OTEL_EXPORTER_OTLP_ENDPOINT`, `OTEL_EXPORTER_OTLP_HEADERS` | O | Grafana Cloud |
| `CORS_ALLOWED_ORIGINS` | M | csv |
| `RATE_LIMIT_*` | O | có default |
| `REVIEWER_EMAIL`, `REVIEWER_OTP` | O | tài khoản App Review (chỉ prod, OTP cố định cho 1 email) |

### iOS (`ios/App/Config/{Debug,Staging,Release}.xcconfig`, không secret thật)

`API_BASE_URL`, `CDN_BASE_URL`, `REVENUECAT_PUBLIC_KEY`, `SENTRY_DSN`, `POSTHOG_KEY`, `POSTHOG_HOST`, `BUNDLE_ID_SUFFIX`.

### Web (`web/.env.example`, chỉ `NEXT_PUBLIC_*`, validate bằng zod lúc build)

`NEXT_PUBLIC_API_BASE_URL`, `NEXT_PUBLIC_CDN_BASE_URL`, `NEXT_PUBLIC_POSTHOG_KEY`, `NEXT_PUBLIC_POSTHOG_HOST`, `NEXT_PUBLIC_SENTRY_DSN`, `NEXT_PUBLIC_APPSTORE_URL`, `NEXT_PUBLIC_SITE_URL`.

### Infra (`infra/compose/.env.example`)

`IMAGE`, `PREV_IMAGE`, `DOMAIN_API`, `ACME_EMAIL`, toàn bộ backend env, `GRAFANA_CLOUD_*`, `R2_BACKUP_*`, `BACKUP_AGE_PUBLIC_KEY`, `HEALTHCHECKS_PING_URL`.

### Subdomain cố định

| Host | Trỏ tới | Env |
|---|---|---|
| `<domain>`, `www.` | Cloudflare Pages (landing + legal) | prod |
| `admin.<domain>` | Cloudflare Pages (cùng project, route `/admin` hoặc project riêng) | prod |
| `api.<domain>` | VPS Caddy (proxied) | prod |
| `api-staging.<domain>` | VPS Caddy (proxied) | staging |
| `cdn.<domain>` | R2 custom domain | shared |
| `status.<domain>` | UptimeRobot/Uptime Kuma page | O |

---

## 9. Breaking change checklist (trước khi merge PR đổi contract)

- [ ] Thay đổi này có làm client cũ (build hiện tại trên App Store) fail decode không? Nếu có → phải là `/v2` hoặc field optional.
- [ ] Đã bump `api_version`; nếu cần, `min_ios_build`.
- [ ] Đã cập nhật examples trong openapi (client test decode dùng chúng).
- [ ] Đã chạy `make gen` cả 3 phía và commit.
- [ ] Đã thêm/ sửa integration test Go + Swift decode test.
- [ ] Đã cập nhật bảng trong file này (nếu thêm endpoint/field).
- [ ] Nếu đổi DB: migration là expand (thêm) trước, contract (xoá) ở release sau.
