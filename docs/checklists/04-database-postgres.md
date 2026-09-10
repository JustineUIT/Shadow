# Checklist 04 — Database Postgres 17 (Neon, region Singapore)

> Dùng để AI check `backend/db/**`, `docs/db/ERD.md`, `infra/scripts/backup.sh`; bạn re-check `[M]`. Tham chiếu D04–D07, D25; `01-CONTRACTS.md` §3 domain model; số đo `03-MEASUREMENT-TOOLS.md` §4.

## 0. Biên giới: DB nhận gì, phải trả gì

| Hướng | MUST | OPT | Nguồn |
|---|---|---|---|
| **Input** schema | `backend/db/migrations/NNNN_*.sql` (goose) là **nguồn duy nhất** của schema; ERD trong `docs/db/ERD.md` sinh/đồng bộ từ migration | | |
| **Input** dữ liệu content | ETL `tools/import-dictionary` từ kaikki (Wiktionary), Tatoeba, NGSL/COCA, CMUdict → bảng `dictionary_*`, `example_sentences`, seed `decks` system | | D25 |
| **Input** kết nối | API/worker qua **pooled** endpoint user `app_rw`; migrate qua **direct** endpoint user `app_migrate`; admin analytics đọc qua `app_ro` (Perfect) | | Contracts §5 |
| **Output** cho backend | Bảng/cột đúng tên snake_case khớp field JSON trong Contracts §3 (không cần map tên); constraint bảo vệ invariant (unique `client_review_id`, `version`, enum bằng `CHECK`/domain) để backend không phải "nhớ" | | |
| **Output** vận hành | Nightly dump encrypted → R2; restore drill hàng tháng có số RTO; `pg_stat_statements` bật; size báo cáo | PITR Neon (paid) | Checklist 05 |
| **Output** evidence | `docs/evidence/db/*` | | |

Quy ước đặt tên (ép bằng test tĩnh `TestSchemaNaming` đọc `information_schema`): bảng số nhiều snake_case; PK `id uuid`; FK `<table_singular>_id`; index `ix_<table>_<cols>`; unique `ux_<table>_<cols>`; FK constraint `fk_<table>_<ref>`; check `ck_<table>_<rule>`; timestamps `created_at timestamptz NOT NULL DEFAULT now()`, `updated_at` có trigger hoặc set từ app (chọn: **app set**, không trigger, để sqlc rõ ràng).

## 1. Schema mục tiêu (v0.1) — bảng, khoá, index bắt buộc

| Bảng | Cột chính (ngoài `id uuid PK`, `created_at`, `updated_at`) | Constraint | Index bắt buộc (hot path) |
|---|---|---|---|
| `users` | `email citext UNIQUE NULL`, `apple_sub text UNIQUE NULL`, `display_name`, `locale`, `timezone`, `level`, `daily_goal int`, `role text CHECK IN ('user','admin')`, `deleted_at timestamptz NULL` | `CHECK (email IS NOT NULL OR apple_sub IS NOT NULL)` | `ux_users_email`, `ux_users_apple_sub`, partial `ix_users_deleted_at WHERE deleted_at IS NOT NULL` |
| `refresh_tokens` | `user_id FK`, `sid uuid` (family), `token_hash bytea UNIQUE`, `expires_at`, `used_at NULL`, `revoked_at NULL`, `platform`, `ua`, `ip inet` | | `ix_refresh_tokens_sid`, `ix_refresh_tokens_user_id`, partial `WHERE revoked_at IS NULL` |
| `email_otps` | `email citext`, `code_hash`, `expires_at`, `attempts int`, `consumed_at NULL` | | `ix_email_otps_email_created` |
| `idempotency_keys` | `user_id`, `key uuid`, `route`, `request_hash`, `response_status`, `response_body jsonb`, `expires_at` | `UNIQUE (user_id, key, route)` | `ix_idempotency_expires` (job xoá) |
| `dictionary_entries` | `headword text`, `headword_norm text` (lower, unaccent), `pos`, `ipa`, `cefr`, `freq_rank int`, `audio_url`, `senses jsonb` (gloss vi/en, examples), `source`, `source_id`, `license` | `UNIQUE (headword, pos, source_id)` | `ix_dict_headword_norm_trgm USING gin (headword_norm gin_trgm_ops)`, `ix_dict_headword_norm_prefix (headword_norm text_pattern_ops)`, `ix_dict_freq_rank` |
| `example_sentences` | `entry_id FK NULL`, `en text`, `vi text`, `source`, `source_id`, `license` | `UNIQUE (source, source_id)` | `ix_examples_entry_id` |
| `decks` | `owner_id FK NULL` (NULL = system), `title`, `description`, `visibility text CHECK IN ('private','public','system')`, `cefr`, `card_count int`, `version int NOT NULL DEFAULT 1`, `deleted_at NULL` | | `ix_decks_owner_created (owner_id, created_at DESC, id DESC)`, partial `ix_decks_public WHERE visibility IN ('public','system') AND deleted_at IS NULL` |
| `cards` | `deck_id FK`, `entry_id FK NULL`, `front`, `back`, `example`, `audio_url`, `position int`, `deleted_at NULL` | | `ix_cards_deck_position (deck_id, position)`, `ix_cards_deck_created (deck_id, created_at DESC, id DESC)` |
| `card_states` | `user_id FK`, `card_id FK`, `state text CHECK IN ('new','learning','review','relearning')`, `due timestamptz`, `stability float8`, `difficulty float8`, `reps int`, `lapses int`, `last_review timestamptz NULL`, `fsrs_version text` | `PRIMARY KEY (user_id, card_id)` | **`ix_card_states_user_due (user_id, due) WHERE state <> 'new'`** (queue), `ix_card_states_user_state (user_id, state)` |
| `review_logs` | `user_id`, `card_id`, `client_review_id uuid`, `rating smallint CHECK 1..4`, `state_before`, `state_after`, `scheduled_days`, `elapsed_days`, `reviewed_at timestamptz`, `duration_ms int`, `fsrs_version` | `UNIQUE (user_id, client_review_id)`; **append-only**: `REVOKE UPDATE, DELETE ON review_logs FROM app_rw` | `ix_review_logs_user_reviewed (user_id, reviewed_at DESC)`, `ix_review_logs_card (card_id, reviewed_at DESC)` |
| `stats_daily` | `user_id`, `day date`, `reviews int`, `new_cards int`, `minutes numeric`, `retention numeric`, `goal_reached bool` | `PRIMARY KEY (user_id, day)` | (PK đủ) |
| `entitlements` | `user_id`, `product text`, `source text CHECK IN ('revenuecat','admin','promo')`, `starts_at`, `expires_at`, `will_renew bool`, `rc_original_transaction_id NULL`, `note` | `UNIQUE (user_id, product, source)` | `ix_entitlements_user_active (user_id) WHERE expires_at > now()` — lưu ý `now()` không dùng được trong partial index → dùng `(user_id, expires_at)` |
| `billing_events` | `event_id text UNIQUE`, `type`, `app_user_id`, `raw jsonb`, `received_at`, `processed_at NULL`, `error text NULL` | | `ix_billing_events_unprocessed WHERE processed_at IS NULL` |
| `admin_audit_logs` | `actor_id`, `action`, `target_type`, `target_id`, `diff jsonb`, `ip inet`, `request_id`, `created_at` | append-only (REVOKE) | `ix_audit_created (created_at DESC, id DESC)`, `ix_audit_target (target_type, target_id)` |
| `devices` (OPT) | `user_id`, `apns_token UNIQUE`, `platform`, `app_version`, `last_seen_at` | | |
| River tables | do `river migrate-up` tạo | | |

Extensions: `pg_trgm`, `citext`, `unaccent`, `pg_stat_statements`. Không `uuid-ossp` (UUIDv7 sinh ở Go). Roles: `app_migrate` (owner), `app_rw` (DML, không DDL, không UPDATE/DELETE trên 2 bảng append-only), `app_ro` (Perfect).

## 2. A — Schema & migration (A01–A12)

| ID | Mục | Target Bare | Target Perfect | Cách check | M |
|---|---|---|---|---|---|
| A01 | Mọi bảng §1 tồn tại với PK, FK, CHECK, UNIQUE như bảng; kiểu đúng (`timestamptz`, `uuid`, `citext` cho email, `jsonb` không `json`) | | | `psql -c "\d+ <table>"` từng bảng vào evidence; `TestSchemaMatchesSpec` đọc `information_schema` so file `docs/db/schema-spec.yaml` | |
| A02 | Naming convention §0 100% | | | `TestSchemaNaming`; query `pg_indexes` regex `^(ix\|ux)_` | |
| A03 | Migration goose `NNNN_<slug>.sql`, có `Down` thật (không `-- noop` trừ data migration ghi rõ), `-- +goose StatementBegin/End` cho function; mỗi migration ≤ 1 mục đích | | | Đọc từng file; `TestMigrateUpDownUp` | |
| A04 | Expand/contract: không `DROP COLUMN`/`RENAME` trong cùng release với code dùng cột; `NOT NULL` thêm cột phải có `DEFAULT` hoặc backfill riêng; `CREATE INDEX CONCURRENTLY` cho bảng > 10k (goose `-- +goose NO TRANSACTION`) | | | `squawk` 0 error (D4 evidence); grep `CONCURRENTLY` cho index thêm sau 0001 | |
| A05 | Enum bằng `text + CHECK` (không `CREATE TYPE enum`) để thêm giá trị không lock; client chịu giá trị lạ | | | grep `CREATE TYPE .* AS ENUM` = 0 | |
| A06 | FK có `ON DELETE` rõ: user → `CASCADE` cho `refresh_tokens, card_states, stats_daily, idempotency_keys, devices`; `decks.owner_id` `SET NULL`? — **quyết định**: user hard-delete → job xoá deck private của user trước, `review_logs` anonymize (`user_id` → sentinel) để giữ thống kê tổng; FK `review_logs.user_id` `ON DELETE SET NULL`? Chọn: **`RESTRICT` + job xử lý tường minh** để không xoá nhầm | | | `\d` mỗi FK có action; job `hard_delete_user` test | |
| A07 | Append-only: `review_logs`, `admin_audit_logs` — `REVOKE UPDATE, DELETE ON ... FROM app_rw` trong migration; test thử UPDATE bằng `app_rw` → permission denied | | Trigger chặn thêm cho superuser | `TestReviewLogsAppendOnly` | |
| A08 | `card_states` PK `(user_id, card_id)`; queue index partial `WHERE state <> 'new'`; EXPLAIN queue = Index Scan | | | D2 evidence `queue.json` | |
| A09 | Search: `gin (headword_norm gin_trgm_ops)` + `text_pattern_ops` prefix; query dùng `headword_norm LIKE $1 \|\| '%'` cho prefix, `%` trgm khi ≥ 3 ký tự; `headword_norm` = `lower(unaccent(headword))` tính lúc import (cột thật, không expression index) | | | D2 `search_prefix.json`, `search_trgm.json` Index Scan/Bitmap; p95 ≤ 15 ms | |
| A10 | Cursor pagination index khớp sort `(…, created_at DESC, id DESC)` cho `decks`, `cards`, `admin users`, `audit` | | | D2 `decks_list.json` không Sort node | |
| A11 | `senses jsonb` có `CHECK (jsonb_typeof(senses) = 'array')`; không index GIN trên jsonb trừ khi có query (không có ở Bare) | | | grep | |
| A12 | ERD `docs/db/ERD.md` (Mermaid `erDiagram`) khớp schema; cập nhật cùng PR migration | Sinh tự động (`schemaspy`/`tbls`) | | So bảng/cột ERD vs `\d`; CI `tbls diff` (Perfect) | |

## 3. Q — Query & performance (Q01–Q08) — số từ `03-MEASUREMENT-TOOLS.md` §4

| ID | Mục | Target Bare | Target Perfect | Cách check | M |
|---|---|---|---|---|---|
| Q01 | 8 query hot path EXPLAIN ANALYZE: 0 Seq Scan trên bảng > 10k dòng; estimate/actual < 10× | | | D2 evidence | |
| Q02 | `pg_stat_statements` sau k6: không query app `mean_exec_time` > 20 ms; top 20 lưu CSV | ≤ 5 ms | | D1 evidence | |
| Q03 | Không N+1: queue, cards list (+state join), admin user detail đều ≤ 2 query | | | pgx tracer count trong test (03-A13) | |
| Q04 | Batch review insert: `INSERT ... SELECT unnest($1::uuid[], ...)` hoặc `pgx.CopyFrom` cho ≥ 20 review; `FOR UPDATE` theo thứ tự `card_id` để tránh deadlock | | | Query file; `TestBatch100ReviewsNoDeadlock` chạy 10 goroutine | |
| Q05 | Stats range query dùng `stats_daily` (không tính lại từ `review_logs`); rollup job idempotent `ON CONFLICT (user_id, day) DO UPDATE` | | | Query file; `TestRollupIdempotent` | |
| Q06 | Cache hit ratio ≥ 99%; 0 unused index sau 7 ngày (trừ PK/unique); dead tuple < 10% | | | D5 evidence | |
| Q07 | pgbench review insert ≥ 500 TPS, avg ≤ 20 ms (pooled, cùng region VPS ↔ Neon SG) | | | D3 evidence | |
| Q08 | Connection: peak ≤ pool max + worker; 0 `idle in transaction` > 5 s; `statement_timeout` 5 s trên role `app_rw` (`ALTER ROLE ... SET statement_timeout`) và `idle_in_transaction_session_timeout` 10 s | | | D8; `\drds` | |

## 4. S — Bảo mật & dữ liệu (S01–S08)

| ID | Mục | Target Bare | Target Perfect | Cách check | M |
|---|---|---|---|---|---|
| S01 | 2 role tách: `app_migrate` (owner DDL) và `app_rw` (DML only, `NOSUPERUSER NOCREATEDB NOCREATEROLE`); API không bao giờ dùng `app_migrate`; password ≥ 32 ký tự random; rotate 90 ngày (lịch trong ops) | `app_ro` cho analytics | | `\du`; `docker inspect api` env `DATABASE_URL` user = `app_rw` | |
| S02 | TLS `sslmode=verify-full` với root CA Neon; không `sslmode=require` | | | Connection string trong compose env (che pass) | |
| S03 | Mọi query đụng user data có `user_id` — kiểm tra ở 03-A10; DB bổ sung: bảng user-owned **không** có query nào trong `db/queries` thiếu `user_id` (cùng test) | RLS nếu chuyển Supabase/self-host | | 03-A10 | |
| S04 | PII tối thiểu: chỉ `email`, `display_name`, `ip` (refresh tokens, audit — giữ 90 ngày rồi job xoá), không lưu tên đầy đủ Apple trừ khi user cho; không lưu OTP/token plaintext (hash) | | Mã hoá cột email (pgcrypto) | `\d users`; grep `code_hash`, `token_hash` | |
| S05 | Soft delete `users.deleted_at` → hard delete job 30 ngày; kiểm tra sau job: 0 dòng `users, refresh_tokens, card_states, stats_daily, decks(owner)` của user; `review_logs.user_id` anonymized | | | `TestHardDeleteRemovesEverything` | |
| S06 | Neon: IP allow (paid) hoặc ít nhất password mạnh + chỉ VPS IP biết; console 2FA; không dùng branch `main` cho test | Neon IP allowlist | | Screenshot Neon settings | [M] |
| S07 | Backup: nightly `pg_dump -Fc` từ VPS (cron/systemd timer) → `age` encrypt (public key trong infra, private key **chỉ** trong password manager) → R2 bucket `backups/` với Object Lock/retention 30 ngày; log ping `healthchecks.io` | Neon PITR 7 ngày (paid); cross-provider copy (B2) | | `infra/scripts/backup.sh`; R2 list có file ≤ 24 h; healthchecks dashboard | |
| S08 | Restore drill hàng tháng: tải dump → giải mã → restore vào Neon branch mới → smoke query (`count users, decks, review_logs`; 1 user login được trên staging trỏ branch) → ghi RTO thật | RTO ≤ 10 phút | | D7 evidence `restore-drill.md` có 5 timestamp | [M] |

## 5. C — Content data (ETL) (C01–C06)

| ID | Mục | Target Bare | Target Perfect | Cách check | M |
|---|---|---|---|---|---|
| C01 | Import idempotent: `ON CONFLICT (headword, pos, source_id) DO UPDATE`; chạy 2 lần không nhân đôi | | | `import --report` lần 2: inserted = 0 | |
| C02 | Chất lượng: top 5000 theo `freq_rank`: ≥ 95% có `ipa`, ≥ 95% có ≥ 1 example, 100% có `audio_url` sau DATA-04; duplicate `headword_norm+pos` = 0 | | Top 10000 | `db/import-report.md` số cụ thể | |
| C03 | Attribution: mỗi dòng có `source`, `source_id`, `license` (`CC BY-SA 4.0` Wiktionary, `CC BY 2.0 FR` Tatoeba, public domain CMUdict); trang Licenses (06-L03) liệt kê | | | `select source, license, count(*) group by 1,2` | |
| C04 | Trim: bỏ entry không có gloss vi/en, không trong top N tần suất (N quyết định bởi DATA-06 size); giữ file dump gốc trên R2 để import lại | | | Size evidence D6 ≤ 400 MB | |
| C05 | Audio: `cdn.<domain>/audio/<entry_id>.opus`, ≤ 20 KB trung bình, `Cache-Control: public, max-age=31536000, immutable`; `audio_url` NULL khi chưa có (client fallback TTS) | mp3 fallback cho Safari cũ | | `rclone ls` size stats; `curl -I` header | |
| C06 | Seed system decks (NGSL 1000, IELTS core, A1–C1 theo CEFR) bằng script import (không migration data lớn); `decks.visibility='system'`, `owner_id NULL`, `card_count` đúng | | | `select title, card_count from decks where visibility='system'` khớp `select count(*) from cards group by deck_id` | |

## 6. Bare Minimum vs Perfect

| Bare | Perfect |
|---|---|
| A01–A12, Q01–Q08, S01–S05, S07–S08 (drill 1 lần trước launch), C01–C06 | Partition `review_logs` theo tháng khi > 10M dòng; Neon branch per PR trong CI; `app_ro` + read replica cho admin analytics; PITR; Neon IP allowlist; `tbls` auto ERD + diff; pgcrypto email; cross-provider backup |

## 7. G — Prompt cho AI (copy nguyên văn)

```
Bạn là DBA reviewer (Postgres 17 / Neon). Đầu vào: docs/checklists/04-database-postgres.md, docs/01-CONTRACTS.md §3,
backend/db/migrations/*.sql, backend/db/queries/*.sql, backend/sqlc.yaml, docs/db/ERD.md, infra/scripts/backup.sh,
infra/scripts/restore-drill.sh, và evidence docs/evidence/db/ nếu có (explain/*.json, pg-stat-statements.csv, squawk.json, size.md, restore-drill.md).
Kiểm tra từng mục A01→A12, Q01→Q08, S01→S08, C01→C06.
Với mỗi mục trả về đúng 1 dòng bảng:
ID | PASS/FAIL/N-A | bằng chứng (file:dòng hoặc lệnh + output rút gọn) | việc cần sửa (nếu FAIL)
Quy tắc:
- Không PASS nếu không có bằng chứng. Với Q01 phải trích node type từ file EXPLAIN JSON (tìm "Node Type": "Seq Scan").
- Mục [M] chỉ ghi "CẦN MANUAL".
- Với A01: lập bảng đối chiếu từng bảng §1: cột thiếu / kiểu sai / constraint thiếu / index thiếu.
- Với A04: liệt kê mọi câu ALTER/DROP/CREATE INDEX trong migration ≥ 0002 và đánh giá lock risk.
- Cuối cùng liệt kê: mọi CREATE TYPE ENUM, mọi SELECT *, mọi query trên bảng user-owned thiếu user_id, mọi FK không có ON DELETE.
- Kết thúc bằng bảng tổng PASS/FAIL/MANUAL theo nhóm.
```

## 8. Re-check thủ công (bạn làm, ~60 phút, lần đầu; 20 phút các lần sau)

1. **Restore drill (S08)**: theo `restore-drill.sh`, bấm đồng hồ. Kết quả phải là: bạn login được vào staging trỏ branch vừa restore, thấy deck của user test. Ghi 5 mốc thời gian.
2. **Append-only (A07)**: `psql` bằng `app_rw` → `UPDATE review_logs SET rating=1 WHERE ...` → phải `permission denied`.
3. **Neon console (S06)**: 2FA bật; role list chỉ 2–3 role; branch `main` không phải nơi chạy test; storage usage < 400 MB.
4. **Search cảm nhận (A09)**: trên app thật, gõ "beau" → kết quả < 100 ms cảm nhận; kiểm tra lại số `pg_stat_statements`.
5. **Backup tồn tại (S07)**: mở R2 bucket, file mới nhất ≤ 24 h, tải về, `age -d` bằng key trong password manager → mở được.

## 9. Evidence phải có khi đóng Phase 1 (và hàng tháng)

`docs/evidence/db/`: `<date>-explain/{queue,reviews_insert,stats_range,search_prefix,search_trgm,decks_list,cards_list,entitlement_check}.json` + `summary.md`, `-pg-stat-statements.csv`, `-pgbench.txt`, `-squawk.json`, `-catalog.md`, `-size.md`, `-restore-drill.md`, `-neon-metrics.png`, `import-report.md`, `schema-dump-<date>.sql` (`pg_dump --schema-only`), `checklist-04-run-<date>.md`.
