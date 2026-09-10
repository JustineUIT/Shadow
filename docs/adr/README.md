# ADR — Architecture Decision Records

Quy tắc:
1. Mỗi ADR bất biến sau khi `Accepted`. Muốn đổi → viết ADR mới có dòng `Supersedes: 000X`, ADR cũ thêm `Superseded by: 000Y`.
2. Format: Status / Context / Options / Decision / Consequences / Checklist liên quan. Dưới 80 dòng.
3. Mọi dòng D-xx trong `00-MASTER-PLAN.md` §1 phải được một ADR ở đây bao phủ.

| # | Tiêu đề | Bao phủ | Status |
|---|---|---|---|
| [0001](0001-backend-modular-monolith-go.md) | Backend: modular monolith Go, stdlib, pgx+sqlc, goose, River | D01, D03, D04, D05, D06 | Accepted |
| [0002](0002-contract-first-codegen.md) | Contract-first: OpenAPI + tokens.json + events.yaml sinh 3 phía | D02, D14, D19, D21, D24 | Accepted |
| [0003](0003-hosting-neon-vps-cloudflare.md) | Hosting: Neon + 1 VPS Docker Compose + Cloudflare (DNS/WAF/Pages/R2) | D07, D08, D09, D10, D17 | Accepted |
| [0004](0004-ios-swiftui-packages-grdb.md) | iOS: SwiftUI iOS 17+, Swift 6, MV + SPM packages, GRDB, Xcode Cloud | D11, D12, D13, D27 | Accepted |
| [0005](0005-auth-and-billing.md) | Auth tự làm (SIWA + OTP, JWT EdDSA, refresh rotation) + StoreKit 2 / RevenueCat | D15, D16, D28 | Accepted |
| [0006](0006-learning-core-data-observability.md) | FSRS server-side, dữ liệu mở Wiktionary/Tatoeba, Piper TTS, Sentry/Grafana/PostHog, Resend, admin shadcn | D18, D20, D22, D23, D25, D26 | Accepted |
| [0007](0007-repo-layout.md) | Repo layout: Next.js chuyển vào `web/`, root mỏng | D29 | Accepted |
| 0008 | VPS provider thực tế (sau P0-03.11) | — | Chưa viết |
