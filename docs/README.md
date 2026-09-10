# docs/ — Bản đồ Tài liệu Toàn diện Dự án Shadow

Bảng ánh xạ 10 yêu cầu kỹ thuật gốc tới các tài liệu chuyên biệt:

| # | Yêu cầu Kỹ thuật | Tài liệu Quy chuẩn & Kế hoạch Chi tiết | Checklist & Verification |
|---|---|---|---|
| 1 | **iOS Swift Best Practice** | `00-MASTER-PLAN.md` (D11–D14, D27), `07-WBS-PHASE-2-IOS.md`, `adr/0004` | `checklists/01-ios-swift.md` |
| 2 | **Frontend ReactJS Best Practice** | `00-MASTER-PLAN.md` (D17–D19, D28), `08-WBS-PHASE-3-WEB.md`, `adr/0003`, `adr/0007` | `checklists/02-web-react.md` |
| 3 | **Backend Golang Best Practice** | `00-MASTER-PLAN.md` (D01–D06, D15), `06-WBS-PHASE-1-BACKEND-DB.md`, `adr/0001`, `adr/0005` | `checklists/03-backend-go.md` |
| 4 | **Database Best Practice** | `db/ERD.md`, `06-WBS-PHASE-1-BACKEND-DB.md`, `adr/0003` | `checklists/04-database-postgres.md` |
| 5 | **Server & Infra Best Practice** | `06-WBS-PHASE-1-BACKEND-DB.md`, `ops/dns-and-network.md`, `adr/0003` | `checklists/05-server-infra.md` |
| 6 | **Domain, Ops & Security Best Practice** | `ops/dns-and-network.md`, `security/threat-model.md`, `product/pricing.md`, `adr/0006` | `checklists/06-domain-ops.md` |
| 7 | **≥6 Tools Đo Hiệu Năng Chuyên Dụng** | `03-MEASUREMENT-TOOLS.md` (I/W/B/D/S/O Series), `09-WBS-PHASE-4-INTEGRATION-TESTING.md` | Điền vào `evidence/SCORECARD.md` |
| 8 | **6 File .md Check Code & Setup (Target Bare vs Perfect, AI Prompts, Recheck)** | `checklists/01-ios-swift.md` đến `06-domain-ops.md` (Mục 0, Bare/Perfect, Prompt AI, Recheck `[M]`) | 6 Checklists chuyên biệt |
| 9 | **Input/Output Specs (Must vs Optional) & Design System 2 Cấp độ** | `01-CONTRACTS.md` (API/DB/Tokens/Events/Fixtures), `02-DESIGN-SYSTEM.md` (Tokens 3 lớp, Bare vs Perfect) | `contracts/*` SoT skeletons |
| 10 | **Breakdown Phòng ban IT A-Z, Quyết định có Rationale, WBS Chi tiết** | `IT-DEPARTMENT-ROLES.md`, `00-MASTER-PLAN.md`, `05-WBS-PHASE-0.md` đến `10-WBS-PHASE-5-LAUNCH-OPERATIONS.md` | Bảng phân rã WBS toàn diện |

---

## Cấu trúc Thư mục Tài liệu & Hợp đồng (Complete Docs Map)

```
docs/
├── README.md                                 # Bản đồ tài liệu (file này)
├── 00-MASTER-PLAN.md                         # Quyết định D01-D29, 6 phases, exit criteria
├── 01-CONTRACTS.md                           # 4 Single Sources of Truth, API & Env specs
├── 02-DESIGN-SYSTEM.md                       # Token architecture (DTCG), Typography, Components
├── 03-MEASUREMENT-TOOLS.md                   # Bộ 6 tools đo hiệu năng chuyên biệt & ngưỡng
├── 04-CURRENT-STATE.md                       # Audit thực tế repo, gap analysis, D29
├── 05-WBS-PHASE-0.md                         # Kế hoạch Foundation (Repository, Setup, PRD)
├── 06-WBS-PHASE-1-BACKEND-DB.md              # Kế hoạch Backend Go, PostgreSQL 17, River, Infra
├── 07-WBS-PHASE-2-IOS.md                     # Kế hoạch iOS Native SwiftUI, GRDB SQLite, FSRS
├── 08-WBS-PHASE-3-WEB.md                     # Kế hoạch Next.js 16 Static Export, Landing, Admin
├── 09-WBS-PHASE-4-INTEGRATION-TESTING.md     # Kế hoạch Tích hợp, Load test k6, Parity, Security
├── 10-WBS-PHASE-5-LAUNCH-OPERATIONS.md       # Kế hoạch Production Launch, App Store, Runbooks
├── IT-DEPARTMENT-ROLES.md                    # Cơ cấu Phòng ban IT & Phân rã Trách nhiệm
├── checklists/
│   ├── 01-ios-swift.md                       # Checklist iOS Swift (AI + Manual [M])
│   ├── 02-web-react.md                       # Checklist React Web (AI + Manual [M])
│   ├── 03-backend-go.md                      # Checklist Backend Go (AI + Manual [M])
│   ├── 04-database-postgres.md               # Checklist PostgreSQL DB (AI + Manual [M])
│   ├── 05-server-infra.md                    # Checklist Server / Infra (AI + Manual [M])
│   └── 06-domain-ops.md                      # Checklist Domain, Legal & Ops (AI + Manual [M])
├── adr/                                      # 0001–0007 Architecture Decision Records
├── db/
│   └── ERD.md                                # Kiến trúc CSDL & ERD PostgreSQL 17 chi tiết
├── security/
│   └── threat-model.md                       # Mô hình đe dọa STRIDE & Hardening đa lớp
├── ops/
│   ├── accounts.md                           # Danh mục tài khoản & dịch vụ hạ tầng
│   └── dns-and-network.md                    # Cấu hình DNS, CDN R2, WAF, Caddy Reverse Proxy
├── product/
│   ├── PRD.md                                # Product Requirement Document 1 trang
│   ├── naming.md                             # Khảo sát tên gọi, Trademark & Domain
│   └── pricing.md                            # Chiến lược giá & In-App Purchases StoreKit 2
└── evidence/
    └── SCORECARD.md                          # Bảng ghi nhận số liệu đo thực tế qua các Phase

contracts/
├── openapi.yaml                              # OpenAPI 3.1 Contract (Client ↔ API)
├── tokens/
│   └── tokens.json                           # DTCG Design Tokens (UI ↔ UI)
├── events.yaml                               # Analytics Events Spec (Client ↔ Analytics)
└── fixtures/
    └── fsrs_cases.json                       # FSRS v4.5 Test Vectors (Go ↔ Swift Parity)
```
