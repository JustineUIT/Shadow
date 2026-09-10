# Infrastructure, DNS, CDN & Network Architecture Specification

> **Mục tiêu**: Định nghĩa toàn diện cấu hình Mạng, Bản ghi DNS, CDN Caching, Tường lửa WAF và Reverse Proxy Caddy cho hệ thống Shadow.

---

## 1. Bản đồ Phân bổ Tên miền & Subdomains (DNS Topology)

| Hostname | Loại Bản Ghi | Giá Trị Đích (Target) | Proxy CF (Vàng/Xám) | Mục Đích Sử Dụng |
|---|---|---|---|---|
| `@` (apex) | CNAME | `<project>.pages.dev` | Proxied (Vàng) | Landing Page (Next.js 16 Static Export) |
| `www` | CNAME | `@` (apex) | Proxied (Vàng) | Redirect tự động về domain chính |
| `admin` | CNAME | `<project>.pages.dev` | Proxied (Vàng) | Cổng quản trị Admin Portal (SPA) |
| `api` | A / AAAA | `<VPS_IPV4>` / `<VPS_IPV6>` | Proxied (Vàng) | API Gateway Production (Go Modular Monolith) |
| `api-staging` | A / AAAA | `<VPS_IPV4>` / `<VPS_IPV6>` | Proxied (Vàng) | API Gateway Staging |
| `cdn` | CNAME | `public.r2.dev` (R2 Custom Domain) | Proxied (Vàng) | Phân phối Audio phát âm & Static Assets |
| `@` | MX | `feedback-smtp.us-east-1.amazonses.com` / `resend` | DNS Only (Xám) | Tiếp nhận & định tuyến Email |
| `@` | TXT | `v=spf1 include:amazonses.com ~all` (hoặc Resend) | DNS Only (Xám) | Xác thực gửi mail SPF |
| `_dmarc` | TXT | `v=DMARC1; p=reject; rua=mailto:dmarc@<domain>` | DNS Only (Xám) | Chính sách chống giả mạo DMARC |
| `resend._domainkey`| TXT | `<DKIM_PUBLIC_KEY>` | DNS Only (Xám) | Chữ ký số DKIM |
| `@` | CAA | `0 issue "letsencrypt.org"`, `0 issue "digicert.com"` | DNS Only (Xám) | Giới hạn Certificate Authority cấp SSL |

---

## 2. Cấu hình Cloudflare Edge & WAF Rules

### 2.1 Cài đặt SSL/TLS & Edge Security
- **SSL/TLS Mode**: `Full (Strict)` (Bắt buộc chứng chỉ Origin CA trên máy chủ VPS).
- **Minimum TLS Version**: `TLS 1.2` (Khuyến nghị TLS 1.3).
- **HSTS**: Bật `max-age=31536000` (1 năm), `includeSubDomains: true`, `preload: true`.
- **Automatic HTTPS Rewrites**: Bật (Chuyển toàn bộ tài nguyên HTTP sang HTTPS).
- **Brotli Compression & Early Hints**: Bật tối đa hóa tốc độ tải trang.

### 2.2 Quy tắc Tường lửa WAF (Cloudflare WAF Rules)
1. **Rule 1 — Chặn Bot độc hại (Bot Fight Mode)**: Tự động thách thức CAPTCHA hoặc chặn các request có `cf.client.bot` hoặc điểm bot score $< 30$.
2. **Rule 2 — Giới hạn Tần suất Truy cập API (Rate Limiting at Edge)**:
   - Endpoint: `api.<domain>/v1/auth/*` -> Giới hạn 10 requests / 1 phút / IP. Vượt ngưỡng -> Chặn tạm thời 10 phút.
   - Endpoint: `api.<domain>/v1/*` -> Giới hạn 300 requests / 1 phút / IP.
3. **Rule 3 — Bảo vệ Cổng Admin**: Giới hạn truy cập `admin.<domain>` chỉ cho phép từ dải IP VPN hoặc yêu cầu Cloudflare Zero Trust Access (OTP xác thực qua Email quản trị).

---

## 3. Cấu hình Caddy Reverse Proxy (`infra/compose/Caddyfile`)

Caddy hoạt động như reverse proxy đầu vào trên máy chủ VPS, tự động xử lý chứng chỉ Cloudflare Origin Certificate và nén dữ liệu:

```caddy
{
    admin off
    auto_https off
}

# Production API Endpoint
api.{$DOMAIN} {
    tls /etc/caddy/certs/origin.pem /etc/caddy/certs/origin.key

    # Giới hạn kích thước body tối đa 2MB (chống tràn bộ nhớ)
    request_body {
        max_size 2MB
    }

    # Bật nén zstd và gzip
    encode zstd gzip

    # Chuyển hướng traffic tới container Go API
    reverse_proxy api:8080 {
        header_up Host {host}
        header_up X-Real-IP {remote_host}
        header_up X-Forwarded-For {remote_host}
        header_up X-Forwarded-Proto {scheme}

        # Health check upstream tự động
        health_uri /healthz
        health_interval 10s
        health_timeout 2s
    }

    # Security Headers
    header {
        Strict-Transport-Security "max-age=31536000; includeSubDomains; preload"
        X-Content-Type-Options "nosniff"
        X-Frame-Options "DENY"
        Referrer-Policy "strict-origin-when-cross-origin"
        -Server
    }
}

# Staging API Endpoint (Chạy song song trên cùng VPS)
api-staging.{$DOMAIN} {
    tls /etc/caddy/certs/origin.pem /etc/caddy/certs/origin.key
    encode zstd gzip

    reverse_proxy api-staging:8080 {
        header_up Host {host}
        header_up X-Real-IP {remote_host}
        header_up X-Forwarded-For {remote_host}
        header_up X-Forwarded-Proto {scheme}
    }
}
```

---

## 4. Cấu hình Cloudflare R2 Object Storage & CDN Caching

- **R2 Bucket Name**: `shadow-assets-prod`
- **Tên miền Custom**: `cdn.<domain>`
- **Cấu hình Cache Rule trên Cloudflare Edge**:
  - Đường dẫn: `cdn.<domain>/audio/*`
  - Cache Level: `Cache Everything`
  - Edge Cache TTL: `30 days`
  - Browser Cache TTL: `7 days`
  - Header đính kèm: `Cache-Control: public, max-age=604800, immutable`
- **Chi phí**: Egress miễn phí 100% nhờ tích hợp Cloudflare R2 + Cloudflare CDN.
