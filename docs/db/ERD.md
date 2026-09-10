# Database Architecture & Entity Relationship Specification (PostgreSQL 17)

> **Mục tiêu**: Định nghĩa toàn diện cấu trúc cơ sở dữ liệu PostgreSQL 17 (Neon, region Singapore) cho hệ thống Shadow.
> **Nguyên tắc vàng**: SQL Migration (`backend/db/migrations/0001_init.sql`) là nguồn sự thật duy nhất (Single Source of Truth). Mọi kiểu dữ liệu, ràng buộc (Constraints) và chỉ mục (Indexes) phải được khai báo tường minh tại đây.

---

## 1. Tổng quan Sơ đồ Thực thể (Entity Relationship Diagram)

```
       +--------------------+          1:N         +-----------------------+
       |       users        |--------------------->|     refresh_tokens    |
       +--------------------+                      +-----------------------+
         | 1            | 1
         |              |
         | 1:N          | 1:N          1:N         +-----------------------+
         |              +------------------------->|     billing_events    |
         v                                         +-----------------------+
       +--------------------+          1:N         +-----------------------+
       |       decks        |--------------------->|     entitlements      |
       +--------------------+                      +-----------------------+
         | 1
         | 1:N
         v
       +--------------------+          1:1 (ref)   +-----------------------+
       |       cards        |--------------------->|  dictionary_entries   |
       +--------------------+                      +-----------------------+
         | 1            | 1                            | 1
         |              |                              | 1:N
         | 1:N          | 1:N                          v
         v              v                          +-----------------------+
  +-------------+  +---------------+               |   example_sentences   |
  | card_states |  |  review_logs  |               +-----------------------+
  +-------------+  +---------------+
```

---

## 2. Danh mục Bảng & Định nghĩa Chi tiết (Detailed Table Definitions)

### 2.1 Bảng `users` (Tài khoản & Hồ sơ người dùng)

Lưu trữ thông tin định danh và tùy chọn học tập của người dùng.

```sql
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email CITEXT UNIQUE,
    apple_sub TEXT UNIQUE,
    display_name TEXT NOT NULL,
    avatar_url TEXT,
    locale VARCHAR(10) NOT NULL DEFAULT 'vi',
    timezone VARCHAR(50) NOT NULL DEFAULT 'Asia/Ho_Chi_Minh',
    level VARCHAR(10) NOT NULL DEFAULT 'A1' CHECK (level IN ('A1', 'A2', 'B1', 'B2', 'C1', 'C2')),
    daily_goal INT NOT NULL DEFAULT 20 CHECK (daily_goal BETWEEN 5 AND 100),
    role VARCHAR(20) NOT NULL DEFAULT 'user' CHECK (role IN ('user', 'admin')),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at TIMESTAMPTZ,
    CONSTRAINT ck_users_identity CHECK (email IS NOT NULL OR apple_sub IS NOT NULL)
);

CREATE UNIQUE INDEX ux_users_email ON users (email) WHERE deleted_at IS NULL;
CREATE UNIQUE INDEX ux_users_apple_sub ON users (apple_sub) WHERE deleted_at IS NULL;
CREATE INDEX ix_users_created_at ON users (created_at DESC);
CREATE INDEX ix_users_deleted_at ON users (deleted_at) WHERE deleted_at IS NOT NULL;
```

---

### 2.2 Bảng `refresh_tokens` (Phiên đăng nhập & Bảo mật Token Family)

Quản lý Refresh Token theo mô hình **Refresh Token Rotation (RTR)** kèm Token Family (`sid`).

```sql
CREATE TABLE refresh_tokens (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    sid UUID NOT NULL, -- Session ID / Family ID
    token_hash BYTEA NOT NULL UNIQUE, -- SHA-256 hash của opaque refresh token
    platform VARCHAR(20) NOT NULL CHECK (platform IN ('ios', 'web', 'admin')),
    user_agent TEXT,
    ip_address INET,
    expires_at TIMESTAMPTZ NOT NULL,
    used_at TIMESTAMPTZ,
    revoked_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX ix_refresh_tokens_sid ON refresh_tokens (sid);
CREATE INDEX ix_refresh_tokens_user_id ON refresh_tokens (user_id);
CREATE INDEX ix_refresh_tokens_active ON refresh_tokens (user_id, expires_at) WHERE revoked_at IS NULL AND used_at IS NULL;
```

---

### 2.3 Bảng `email_otps` (Mã xác thực đăng nhập qua Email)

```sql
CREATE TABLE email_otps (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email CITEXT NOT NULL,
    code_hash BYTEA NOT NULL, -- SHA-256 hash của OTP 6 số
    attempts INT NOT NULL DEFAULT 0,
    expires_at TIMESTAMPTZ NOT NULL,
    consumed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX ix_email_otps_lookup ON email_otps (email, created_at DESC) WHERE consumed_at IS NULL;
```

---

### 2.4 Bảng `idempotency_keys` (Đảm bảo tính Idempotent của Mutation API)

```sql
CREATE TABLE idempotency_keys (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    key UUID NOT NULL,
    route VARCHAR(100) NOT NULL,
    request_hash BYTEA NOT NULL,
    response_status INT NOT NULL,
    response_headers JSONB NOT NULL DEFAULT '{}'::jsonb,
    response_body JSONB NOT NULL DEFAULT '{}'::jsonb,
    expires_at TIMESTAMPTZ NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    CONSTRAINT ux_idempotency_user_key_route UNIQUE (user_id, key, route)
);

CREATE INDEX ix_idempotency_cleanup ON idempotency_keys (expires_at);
```

---

### 2.5 Bảng `dictionary_entries` & `example_sentences` (Kho Từ điển & Câu ví dụ)

```sql
CREATE TABLE dictionary_entries (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    headword TEXT NOT NULL,
    headword_norm TEXT NOT NULL, -- Lowercase, unaccented, trimmed
    pos VARCHAR(30) NOT NULL, -- Part of Speech: noun, verb, adjective, etc.
    ipa TEXT NOT NULL,
    cefr VARCHAR(10) CHECK (cefr IN ('A1', 'A2', 'B1', 'B2', 'C1', 'C2')),
    freq_rank INT, -- Độ phổ biến (1 = từ phổ biến nhất)
    audio_url TEXT,
    senses JSONB NOT NULL, -- Mảng các định nghĩa: [{"gloss_vi": "...", "gloss_en": "...", "examples": [...]}]
    source VARCHAR(50) NOT NULL DEFAULT 'kaikki',
    source_id TEXT,
    license VARCHAR(50) NOT NULL DEFAULT 'CC-BY-SA 4.0',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    CONSTRAINT ux_dict_headword_pos_source UNIQUE (headword, pos, source_id)
);

-- Trigram index cho Full-Text Search và Fuzzy matching
CREATE INDEX ix_dict_headword_norm_trgm ON dictionary_entries USING gin (headword_norm gin_trgm_ops);
-- Prefix index cho Autocomplete gõ phím
CREATE INDEX ix_dict_headword_norm_prefix ON dictionary_entries (headword_norm text_pattern_ops);
CREATE INDEX ix_dict_freq_rank ON dictionary_entries (freq_rank ASC NULLS LAST);

CREATE TABLE example_sentences (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    entry_id UUID REFERENCES dictionary_entries(id) ON DELETE CASCADE,
    en TEXT NOT NULL,
    vi TEXT NOT NULL,
    audio_url TEXT,
    source VARCHAR(50) NOT NULL DEFAULT 'tatoeba',
    source_id TEXT,
    license VARCHAR(50) NOT NULL DEFAULT 'CC-BY 2.0 FR',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    CONSTRAINT ux_examples_source_id UNIQUE (source, source_id)
);

CREATE INDEX ix_examples_entry_id ON example_sentences (entry_id);
```

---

### 2.6 Bảng `decks` & `cards` (Bộ thẻ & Thẻ học cá nhân / hệ thống)

```sql
CREATE TABLE decks (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    owner_id UUID REFERENCES users(id) ON DELETE CASCADE, -- NULL = Deck hệ thống (System Deck)
    title VARCHAR(150) NOT NULL,
    description TEXT,
    visibility VARCHAR(20) NOT NULL DEFAULT 'private' CHECK (visibility IN ('private', 'public', 'system')),
    cefr VARCHAR(10) CHECK (cefr IN ('A1', 'A2', 'B1', 'B2', 'C1', 'C2')),
    card_count INT NOT NULL DEFAULT 0,
    version INT NOT NULL DEFAULT 1,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at TIMESTAMPTZ
);

CREATE INDEX ix_decks_owner ON decks (owner_id, created_at DESC) WHERE deleted_at IS NULL;
CREATE INDEX ix_decks_public ON decks (visibility, created_at DESC) WHERE visibility IN ('public', 'system') AND deleted_at IS NULL;

CREATE TABLE cards (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    deck_id UUID NOT NULL REFERENCES decks(id) ON DELETE CASCADE,
    entry_id UUID REFERENCES dictionary_entries(id) ON DELETE SET NULL,
    front TEXT NOT NULL,
    back TEXT NOT NULL,
    example TEXT,
    audio_url TEXT,
    position INT NOT NULL DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at TIMESTAMPTZ
);

CREATE INDEX ix_cards_deck_position ON cards (deck_id, position) WHERE deleted_at IS NULL;
CREATE INDEX ix_cards_deck_created ON cards (deck_id, created_at DESC) WHERE deleted_at IS NULL;
```

---

### 2.7 Bảng `card_states` & `review_logs` (Trạng thái FSRS & Nhật ký học tập Append-Only)

```sql
CREATE TABLE card_states (
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    card_id UUID NOT NULL REFERENCES cards(id) ON DELETE CASCADE,
    state VARCHAR(20) NOT NULL DEFAULT 'new' CHECK (state IN ('new', 'learning', 'review', 'relearning')),
    due TIMESTAMPTZ NOT NULL DEFAULT now(),
    stability DOUBLE PRECISION NOT NULL DEFAULT 0.0 CHECK (stability >= 0.0),
    difficulty DOUBLE PRECISION NOT NULL DEFAULT 0.0 CHECK (difficulty BETWEEN 0.0 AND 10.0),
    reps INT NOT NULL DEFAULT 0 CHECK (reps >= 0),
    lapses INT NOT NULL DEFAULT 0 CHECK (lapses >= 0),
    last_review TIMESTAMPTZ,
    fsrs_version VARCHAR(10) NOT NULL DEFAULT 'v4.5',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (user_id, card_id)
);

-- Chỉ mục cực kỳ quan trọng cho truy vấn lấy hàng đợi học tập hàng ngày (Study Queue)
CREATE INDEX ix_card_states_user_due ON card_states (user_id, due) WHERE state <> 'new';
CREATE INDEX ix_card_states_user_state ON card_states (user_id, state);

CREATE TABLE review_logs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    card_id UUID NOT NULL REFERENCES cards(id) ON DELETE CASCADE,
    client_review_id UUID NOT NULL, -- Khóa sinh từ Client để đảm bảo Idempotent
    rating SMALLINT NOT NULL CHECK (rating BETWEEN 1 AND 4), -- 1: Again, 2: Hard, 3: Good, 4: Easy
    state_before VARCHAR(20) NOT NULL,
    state_after VARCHAR(20) NOT NULL,
    scheduled_days DOUBLE PRECISION NOT NULL,
    elapsed_days DOUBLE PRECISION NOT NULL,
    reviewed_at TIMESTAMPTZ NOT NULL,
    duration_ms INT NOT NULL DEFAULT 0,
    fsrs_version VARCHAR(10) NOT NULL DEFAULT 'v4.5',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    CONSTRAINT ux_review_logs_user_client_review UNIQUE (user_id, client_review_id)
);

CREATE INDEX ix_review_logs_user_reviewed ON review_logs (user_id, reviewed_at DESC);
CREATE INDEX ix_review_logs_card ON review_logs (card_id, reviewed_at DESC);
```

---

### 2.8 Bảng `stats_daily` (Thống kê Tiến độ Ngày)

```sql
CREATE TABLE stats_daily (
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    day DATE NOT NULL,
    reviews INT NOT NULL DEFAULT 0,
    new_cards INT NOT NULL DEFAULT 0,
    minutes NUMERIC(5, 2) NOT NULL DEFAULT 0.0,
    retention NUMERIC(5, 2) NOT NULL DEFAULT 0.0,
    goal_reached BOOLEAN NOT NULL DEFAULT FALSE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (user_id, day)
);
```

---

### 2.9 Bảng `entitlements`, `billing_events` & `admin_audit_logs`

```sql
CREATE TABLE entitlements (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    product VARCHAR(50) NOT NULL, -- 'pro_monthly', 'pro_yearly', 'lifetime'
    source VARCHAR(30) NOT NULL CHECK (source IN ('revenuecat', 'admin', 'promo')),
    starts_at TIMESTAMPTZ NOT NULL,
    expires_at TIMESTAMPTZ NOT NULL,
    will_renew BOOLEAN NOT NULL DEFAULT FALSE,
    rc_original_transaction_id TEXT,
    note TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    CONSTRAINT ux_entitlements_user_product_source UNIQUE (user_id, product, source)
);

CREATE INDEX ix_entitlements_user_active ON entitlements (user_id, expires_at);

CREATE TABLE billing_events (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_id TEXT NOT NULL UNIQUE, -- ID từ RevenueCat Webhook
    type VARCHAR(50) NOT NULL,
    app_user_id TEXT NOT NULL,
    raw JSONB NOT NULL,
    received_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    processed_at TIMESTAMPTZ,
    error TEXT
);

CREATE INDEX ix_billing_events_pending ON billing_events (received_at) WHERE processed_at IS NULL;

CREATE TABLE admin_audit_logs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    actor_id UUID NOT NULL REFERENCES users(id),
    action VARCHAR(50) NOT NULL,
    target_type VARCHAR(50) NOT NULL,
    target_id TEXT NOT NULL,
    diff JSONB NOT NULL DEFAULT '{}'::jsonb,
    ip_address INET,
    request_id UUID,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX ix_audit_created ON admin_audit_logs (created_at DESC);
CREATE INDEX ix_audit_target ON admin_audit_logs (target_type, target_id);
```

---

## 3. Phân quyền Người dùng Cơ sở Dữ liệu (Database Roles & Security)

Để bảo đảm an toàn dữ liệu, 3 vai trò được tách biệt hoàn toàn:

1. **`app_migrate`** (Owner): Có toàn quyền DDL (tạo/sửa bảng, chỉ mục) dùng khi chạy goose migration.
2. **`app_rw`** (Application Runtime): Chỉ có quyền DML (`SELECT`, `INSERT`, `UPDATE`, `DELETE`) trên các bảng nghiệp vụ. Bị chặn tuyệt đối quyền `UPDATE` và `DELETE` trên bảng bất biến `review_logs` và `admin_audit_logs`.
3. **`app_ro`** (Read-Only Analytics): Chỉ có quyền `SELECT` dùng cho báo cáo hoặc đọc dữ liệu phân tích.

```sql
-- Thiết lập quyền hạn cho app_rw
GRANT CONNECT ON DATABASE shadow TO app_rw;
GRANT USAGE ON SCHEMA public TO app_rw;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO app_rw;
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA public TO app_rw;

-- Khóa quyền xóa sửa trên các bảng append-only
REVOKE UPDATE, DELETE ON review_logs FROM app_rw;
REVOKE UPDATE, DELETE ON admin_audit_logs FROM app_rw;
```
