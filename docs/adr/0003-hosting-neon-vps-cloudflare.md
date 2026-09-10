# ADR 0003 — Hosting: Neon Postgres + 1 VPS Docker Compose + Cloudflare toàn tuyến

- Status: **Accepted** (2026-09-10)
- Bao phủ: D07, D08, D09, D10, D17

## Context
Ngân sách năm đầu mục tiêu < 300 USD (00 §8). User chính ở Việt Nam → region Singapore. Sản phẩm dùng thương mại (có IAP) → **Vercel Hobby không được dùng** (điều khoản). Nhiều file audio tĩnh → egress là chi phí lớn nhất nếu chọn sai.

## Options
| Hạng mục | Options | Loại vì |
|---|---|---|
| Database | Supabase; self-host Postgres trên VPS; **Neon** | Self-host: tự lo backup/PITR, rủi ro mất dữ liệu người dùng. Supabase: tốt nhưng kéo theo auth/RLS không dùng (ADR 0005 tự làm auth). Neon: branch cho test, PITR, region SG |
| Compute | Cloud Run; Fly.io; Render; **1 VPS ARM (Oracle Free → fallback trả tiền)** | Cold start (Run/Render free), Fly free tier đã cắt; VPS dạy ops thật, không cold start |
| Object storage | S3; Backblaze B2; **Cloudflare R2** | R2 egress free; audio phát nhiều lần |
| DNS/Edge | Route53 + CloudFront; **Cloudflare** | Free WAF/DDoS/cache, Registrar giá gốc, Pages free cho thương mại |
| Web hosting | Vercel; Netlify; **Cloudflare Pages (Next.js static export)** | Vercel Hobby cấm thương mại; Pro 20 USD/tháng không cần cho site tĩnh |

## Decision
- Neon Postgres 17, `ap-southeast-1`, pooled connection cho `api`, direct cho `migrate`.
- 1 VPS ARM Singapore: Docker Compose (`caddy`, `api`, `worker`, staging song song), Caddy TLS, ufw, fail2ban, unattended-upgrades.
- R2 bucket `shadow-cdn` (custom domain `cdn.<domain>`) và `shadow-backups` (dump mã hoá `age`).
- Cloudflare: Registrar, DNS (DNSSEC), proxy `api.`, WAF free, rate limit `/v1/auth/*`, Pages cho `web/`.

## Consequences
- (+) Chi phí ~0–6 USD/tháng ngoài Apple 99 USD.
- (−) VPS là single point of failure: chấp nhận; stateless API + Neon managed → dựng lại từ image < 10 phút (runbook 05).
- (−) Neon free tier storage nhỏ: DATA-06 đo sớm; fallback Launch plan 19 USD hoặc tách content sang SQLite pack trên R2.
- (−) Static export: không có server component động trên web → admin là SPA gọi `api.`; landing form dùng Pages Function.
- Provider VPS thực tế (Oracle có đăng ký được không) → ADR 0008.

## Checklist liên quan
05-server-infra toàn bộ. 04-database mục B (backup), S07. 06-domain D01–D05.
