# Accounts — dịch vụ bên thứ ba (P0-03)

> **KHÔNG** ghi secret, API key, password, DSN, connection string vào file này. Secret ở password manager vault `Shadow`.
> File này trả lời: dùng dịch vụ nào, plan gì, email nào, region nào, giới hạn free tier là bao nhiêu, dashboard ở đâu.
> Gate: `gitleaks detect --source docs` phải 0 finding trước mỗi commit.

## 1. Email quản trị

| Địa chỉ | Route tới | Dùng cho |
|---|---|---|
| `admin@<domain>` | `TODO` Gmail cá nhân | Đăng ký mọi dịch vụ dưới |
| `support@<domain>` | như trên | App Store, footer web |
| `privacy@<domain>` | như trên | Privacy policy, yêu cầu xoá dữ liệu |
| `dmarc@<domain>` | như trên | Báo cáo DMARC |
| `noreply@<domain>` | không nhận | From của Resend |

## 2. Bảng dịch vụ

| # | Dịch vụ | Plan | Email | Region | Ngày tạo | 2FA | Dashboard | Giới hạn free cần nhớ | Dùng ở |
|---|---|---|---|---|---|---|---|---|---|
| 1 | Cloudflare (Registrar, DNS, R2, Pages, Turnstile) | Free | admin@ | — | `TODO` | ☐ | dash.cloudflare.com | R2 10 GB, Pages 500 build/tháng | 05, 06 |
| 2 | GitHub | Free | | — | có | ☐ | github.com/JustineUIT/Shadow | Actions 2000 phút (Linux) | tất cả |
| 3 | Apple Developer | Individual 99 USD/năm | | — | `TODO` | ☐ | developer.apple.com | Xcode Cloud 25 h/tháng | 01, 06 |
| 4 | Neon | Free | | ap-southeast-1 | `TODO` | ☐ | console.neon.tech | Storage `TODO` GB (ghi số lúc đăng ký), compute giờ/tháng | 03, 04 |
| 5 | Sentry | Developer free | | US | `TODO` | ☐ | sentry.io | 5k errors/tháng | 01, 02, 03 |
| 6 | PostHog | Free | | US | `TODO` | ☐ | us.posthog.com | 1M events/tháng | 02, 06 |
| 7 | Resend | Free | | — | `TODO` | ☐ | resend.com | 3k mail/tháng, 100/ngày | 03, 06 |
| 8 | RevenueCat | Free | | — | `TODO` | ☐ | app.revenuecat.com | tới 2.5k USD MTR | 01, 03 |
| 9 | Grafana Cloud | Free | | ap-southeast (nếu có) | `TODO` | ☐ | grafana.com | 10k series, 50 GB logs, 50 GB traces | 03, 05 |
| 10 | UptimeRobot | Free | | — | `TODO` | ☐ | uptimerobot.com | 50 monitor, 5 phút | 05 |
| 11 | healthchecks.io | Free | | — | `TODO` | ☐ | healthchecks.io | 20 checks | 05 |
| 12 | VPS: `TODO Oracle / Hetzner / Vultr` | `TODO` | | Singapore | `TODO` | ☐ | | ADR 0008 | 05 |
| 13 | Password manager | | | — | | ☐ | | | — |

## 3. Giá trị công khai (được phép ghi)

| Khoá | Giá trị | Ghi chú |
|---|---|---|
| Domain chính | `TODO` | Nguồn duy nhất cho `<domain>` trong docs |
| Cloudflare Account ID | `TODO` | Không phải secret |
| R2 bucket | `shadow-cdn`, `shadow-backups` | |
| Pages project | `shadow-web` | |
| Neon project ID | `TODO` | |
| Apple Team ID | `TODO` | |
| Bundle ID | `com.<org>.shadow` | |
| RevenueCat public SDK key (iOS) | `TODO` | Public by design |
| PostHog project API key | `TODO` | Public by design (write-only) |
| Sentry DSN | **không ghi** | DSN có thể spam; để trong xcconfig/env |

## 4. Lịch xoay secret (chi tiết `docs/security/secrets.md` — Phase 1)

| Secret | Chu kỳ | Cách |
|---|---|---|
| JWT signing key (Ed25519) | 90 ngày | 2 `kid` song song, JWKS |
| Neon password | 180 ngày | Neon console reset → cập nhật compose env |
| Resend / RC webhook secret | 365 ngày hoặc khi lộ | Dashboard |
| SSH key VPS | 365 ngày | `bootstrap-vps.sh` |
