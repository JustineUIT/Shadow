# Checklist 06 — Domain, DNS, Email, Legal, App Store, Growth ("những vấn đề còn lại")

> Dùng để AI check `web/app/legal/*`, `web/public/*`, `docs/legal/*`, `docs/product/*`, `docs/growth/*`, cấu hình App Store Connect (bạn dán export/screenshot); bạn re-check `[M]`. Tham chiếu D10, D16, D22, D23; `01-CONTRACTS.md` §7 events, §8 subdomain; số đo `03-MEASUREMENT-TOOLS.md` §6.

## 0. Biên giới: phần này nhận gì, phải trả gì

| Hướng | MUST | OPT | Nguồn |
|---|---|---|---|
| **Input** | Tên sản phẩm + domain đã mua (P0-02); danh sách subdomain cố định (Contracts §8); danh sách dữ liệu thu thập (từ schema 04 + events 07) để viết Privacy + App Privacy labels; danh sách dependency + license (O12) | | |
| **Output** cho Infra/Web/Backend | DNS records đúng: `A/AAAA api.` (proxied), `CNAME <domain>/www/admin` → Pages, `CNAME cdn.` → R2, `MX/TXT` Resend, `TXT _dmarc`, `CAA`, DNSSEC bật; email `admin@`, `support@`, `privacy@`, `noreply@` hoạt động | `status.` | Checklist 05 A06 |
| **Output** cho iOS | URL công khai: Privacy, Terms, Support, Account deletion, Licenses (đưa vào App Store Connect + Settings trong app); reviewer account (`REVIEWER_EMAIL` + OTP cố định); App Privacy labels; screenshots; review notes | Universal links `apple-app-site-association` | Checklist 01 S06, 03 §8 env |
| **Output** đo | Dashboards PostHog (Activation, Retention, Monetization), RevenueCat, ASC Analytics; `docs/growth/launch.md`; `docs/reviews/YYYY-MM.md` | | Master Plan §9 KPI |
| **Output** evidence | `docs/evidence/domain/*` | | |

## 1. Cấu trúc file kỳ vọng

```
docs/product/{PRD.md, naming.md, pricing.md}
docs/legal/{privacy.md, terms.md, licenses.md (generated + manual), support.md, account-deletion.md, data-inventory.md, app-privacy-labels.md, review-notes.md}
docs/growth/{aso.md, launch.md, channels.md}
docs/ops/{accounts.md, dns-records.md, email-setup.md}
web/app/legal/{privacy,terms,licenses,support,account-deletion}/page.tsx  (render từ docs/legal/*.md hoặc web/src/content/legal)
web/public/{robots.txt (gen), .well-known/apple-app-site-association (OPT), .well-known/security.txt}
scripts/measure/dom-audit.sh
```

## 2. D — Domain & DNS (D01–D08)

| ID | Mục | Target Bare | Target Perfect | Cách check | M |
|---|---|---|---|---|---|
| D01 | Domain tại Cloudflare Registrar (hoặc transfer), auto-renew, registrar lock, WHOIS privacy, 2FA account; email đăng ký là `admin@` domain khác (không phụ thuộc chính domain này) | | | Screenshot registrar; `whois <domain>` thấy lock | [M] |
| D02 | DNSSEC bật, chain "secure" (DNSViz / `delv`) | | | O4 evidence `dns.txt` | |
| D03 | Records đúng bảng Contracts §8: `<domain>`, `www` → Pages (proxied); `admin` → Pages; `api`, `api-staging` → VPS (proxied); `cdn` → R2 custom domain; không record thừa trỏ IP cũ; TTL auto | | | `docs/ops/dns-records.md` = export Cloudflare (`cloudflare dns export`) đối chiếu; `dig +short` từng host | |
| D04 | CAA: `0 issue "letsencrypt.org"`, `0 issue "pki.goog"`, `0 issue "digicert.com"` (Cloudflare Universal SSL), `0 issuewild ";"` nếu không wildcard, `iodef mailto:admin@` | | | `dig CAA <domain>` | |
| D05 | Redirect: `http://` → `https://` (Always HTTPS), `www` → apex (hoặc ngược lại, chọn 1: **apex**), `/index.html` → `/`; không chuỗi redirect > 1 bước | | | `curl -sIL http://www.<domain> \| grep -c "HTTP/"` = 2 (1 redirect + 200) | |
| D06 | `security.txt` (`/.well-known/security.txt`: `Contact: mailto:security@`, `Expires`, `Preferred-Languages: vi, en`) | | Bug bounty policy | `curl` 200 | |
| D07 | Cloudflare zone: SSL Full strict, min TLS 1.2, HSTS bật (max-age 1 năm, includeSubDomains — chỉ khi mọi subdomain HTTPS ✓), TLS 1.3, HTTP/3, Brotli, Early Hints, Bot Fight Mode, WAF managed | HSTS preload submit | | Screenshot; O3 SSL Labs A+ | |
| D08 | Domain phụ chống typo/brand (`.vn`, `.app`) redirect về chính — chỉ nếu rẻ | | | | |

## 3. E — Email (E01–E06)

| ID | Mục | Target Bare | Target Perfect | Cách check | M |
|---|---|---|---|---|---|
| E01 | Resend domain verified: DKIM (3 CNAME hoặc TXT), SPF `v=spf1 include:amazonses.com ~all` (Resend dùng SES — kiểm tra record Resend cấp), Return-Path CNAME; gửi từ `noreply@<domain>`; reply-to `support@` | Subdomain gửi riêng `mail.<domain>` để bảo vệ reputation | | Resend dashboard "Verified"; `dig TXT <domain>` | |
| E02 | DMARC: `_dmarc TXT "v=DMARC1; p=quarantine; rua=mailto:dmarc@<domain>; ruf=...; fo=1; pct=100"`; đọc báo cáo (dmarcian/Postmark DMARC free) | `p=reject` sau 30 ngày sạch; BIMI | | `dig TXT _dmarc.<domain>`; O5 report XML | |
| E03 | Nhận email: `admin@ support@ privacy@ security@ dmarc@` — Cloudflare Email Routing → hộp thư thật; test gửi/nhận từ Gmail | Helpdesk (Freshdesk free) | | Gửi 5 email test, nhận đủ | [M] |
| E04 | mail-tester ≥ 9/10 cho email OTP thật; không blacklist (MXToolbox) | 10/10 | | O5 evidence | |
| E05 | Template OTP: plain text + HTML tối giản, có tên app, mã 6 số, hết hạn 10 phút, "không phải bạn? bỏ qua", không link đăng nhập (chống phishing); ngôn ngữ theo `Accept-Language` | | | Screenshot email; template trong `internal/platform/mail/templates` | |
| E06 | Email giao dịch khác (welcome, delete confirm, export ready) có unsubscribe **không** cần (transactional); marketing (nếu có) qua broadcast có unsubscribe | | | Review danh sách email trong code | |

## 4. L — Legal & compliance (L01–L08)

| ID | Mục | Target Bare | Target Perfect | Cách check | M |
|---|---|---|---|---|---|
| L01 | **Privacy Policy** `/legal/privacy` (vi + en): controller (tên bạn/công ty + email `privacy@`), dữ liệu thu thập (map 1-1 với `docs/legal/data-inventory.md`: email, Apple sub, tên hiển thị, dữ liệu học, thiết bị, IP 90 ngày, purchase qua Apple/RevenueCat, analytics PostHog không cookie), mục đích, cơ sở, bên thứ ba (Neon SG, Cloudflare, R2, Sentry, PostHog, RevenueCat, Resend, Apple), lưu trữ Singapore, thời hạn (30 ngày sau xoá), quyền (truy cập, xoá, export), trẻ em (≥ 13 hoặc 4+ không thu thập), cập nhật ngày; phù hợp Nghị định 13/2023 (VN) + GDPR cơ bản nếu có user EU | Luật sư review; DPA với vendor | | Trang 200; mỗi mục có; `data-inventory.md` khớp schema 04 | [M] |
| L02 | **Terms of Service** `/legal/terms`: tài khoản, hành vi cấm, IAP/subscription (auto-renew, huỷ qua Apple, refund theo Apple), nội dung user, IP, miễn trừ, luật áp dụng (Việt Nam), liên hệ | | | Trang 200 | |
| L03 | **Licenses** `/legal/licenses` + màn Settings → Licenses trong app: 100% dependency (SPM `license-plist`, web/backend `licensee`) + **content**: Wiktionary (CC BY-SA 4.0, link), Tatoeba (CC BY 2.0 FR), CMUdict (BSD), NGSL (CC BY-SA 4.0), Piper voices (license theo model); nêu rõ dữ liệu từ điển derived giữ CC BY-SA | | | O12 evidence; trang liệt kê; grep GPL/AGPL = 0 | |
| L04 | **Support** `/legal/support`: email `support@`, FAQ 10 câu, cách huỷ subscription (link Apple), cách xoá tài khoản, thời gian phản hồi (48 h) | Helpdesk | | Trang 200; URL nhập vào ASC Support URL | |
| L05 | **Account deletion** `/legal/account-deletion`: hướng dẫn xoá trong app (Settings → Delete) + cách yêu cầu qua email nếu không vào được app; nói rõ 30 ngày hard delete; **bắt buộc theo Apple 5.1.1(v)** | Data export self-serve | | Trang 200; iOS Settings có nút; 01-S04 | |
| L06 | Cookie/consent: web dùng PostHog cookieless + không cookie marketing → không cần banner; ghi trong Privacy; nếu thêm cookie → banner | | | `document.cookie` trên landing = rỗng | |
| L07 | Pricing hiển thị đúng luật: giá VND có VAT, auto-renew rõ, trial rõ "sau X ngày sẽ tính phí", trên paywall và landing | | | Screenshot paywall + pricing | |
| L08 | Thuế/pháp nhân: hộ kinh doanh hoặc cá nhân kê khai thu nhập từ Apple; Apple Paid Apps Agreement + tax form (W-8BEN) + bank; ghi quyết định trong `docs/product/pricing.md` | Công ty | | ASC Agreements "Active" | [M] |

## 5. A — App Store Connect & App Review (A01–A12)

| ID | Mục | Target Bare | Target Perfect | Cách check | M |
|---|---|---|---|---|---|
| A01 | App record: bundle id, tên (≤ 30), subtitle (≤ 30), category Education, age rating 4+ (không UGC public → nếu public decks có UGC thì cần moderation + report + block → **quyết định**: public decks **chỉ system** ở MVP để tránh UGC), primary language vi + en | | | Screenshot ASC | |
| A02 | **App Privacy labels** (`docs/legal/app-privacy-labels.md` → nhập ASC): Contact Info (email) linked, Identifiers (user id) linked, Usage Data (product interaction) linked, Purchases linked, Diagnostics (crash) not linked; **không** Tracking; khớp `PrivacyInfo.xcprivacy` (01-S06) và Privacy Policy | | | So 3 nguồn khớp 100% | |
| A03 | **IAP**: 2 sản phẩm auto-renewable `pro_monthly`, `pro_yearly` cùng subscription group, trial 7 ngày yearly; localized description; review screenshot; giá VN tier; RevenueCat entitlement `pro` map 2 product; **restore purchases** nút có; paywall có link Terms + Privacy + "huỷ bất cứ lúc nào" | Intro offer, promo codes, win-back | | ASC IAP "Ready to Submit"; RevenueCat dashboard; sandbox test 01 re-check 7 | [M] |
| A04 | StoreKit: dùng StoreKit 2 qua RevenueCat; **không** link ra web mua ngoài (trừ được phép theo region — không dùng ở MVP); không nhắc "rẻ hơn trên web" | | | grep `openURL` tới trang pricing = 0 trong Paywall | |
| A05 | **Sign in with Apple** có mặt vì có login bên thứ ba khác? — MVP: SIWA + Email OTP (Email OTP là first-party → SIWA không bắt buộc nhưng **giữ**, giảm friction) ; nút SIWA đúng HIG (kích cỡ, màu, text) | | | Screenshot nút | |
| A06 | **Account deletion trong app** (5.1.1(v)): Settings → Delete account → confirm → xoá thật (không chỉ email yêu cầu) | | | 01-S04 | |
| A07 | **Review notes + reviewer account**: `REVIEWER_EMAIL` với OTP cố định (server config, chỉ prod, 1 email); notes giải thích OTP, offline mode, cách kích hoạt Pro sandbox; video demo link nếu offline flow khó | | | `docs/legal/review-notes.md`; server env | |
| A08 | Screenshots 6.9" + 6.5" (bắt buộc), iPad nếu hỗ trợ (MVP: **iPhone only**, tắt iPad trong target để không phải nộp iPad screenshots), 5–8 ảnh, không hiện notch fake, text đúng ngôn ngữ; App Preview video OPT | Localized screenshots vi/en | | Bộ ảnh trong `docs/growth/assets/` | |
| A09 | Metadata ASO (`docs/growth/aso.md`): keyword field 100 ký tự, không lặp tên, không tên đối thủ; description 3 dòng đầu có value prop; What's New mỗi bản | | | File; ASC | |
| A10 | Privacy manifest & required reason APIs: 0 ITMS-91053/91061 email sau upload; third-party SDK (Sentry, RevenueCat, PostHog, GRDB) đều có `PrivacyInfo.xcprivacy` + signature nếu trong danh sách Apple | | | O11 evidence | |
| A11 | Export compliance: dùng HTTPS chuẩn → `ITSAppUsesNonExemptEncryption = NO` trong Info.plist | | | plist | |
| A12 | Reject runbook `docs/ops/runbooks/app-review-reject.md`: mã guideline hay gặp (2.1 crash, 2.3 metadata, 3.1.1 IAP, 4.0 design, 5.1.1 privacy) → hành động; dùng Reply/Appeal | | | File | |

## 6. G — Growth & analytics (G01–G08)

| ID | Mục | Target Bare | Target Perfect | Cách check | M |
|---|---|---|---|---|---|
| G01 | PostHog project: events đúng `events.yaml` 100% (tên + props M); không PII (kiểm tra live events 24 h); `distinct_id` = user uuid sau login, alias từ anonymous | | | Export event definitions so `events.yaml`; grep email trong live = 0 | |
| G02 | Dashboards: **Activation** (funnel `app_opened → signup_completed → study_session_completed` theo cohort tuần), **Retention** (D1/D7/D30 theo `study_session_completed`), **Monetization** (`paywall_viewed → purchase_started → purchase_completed` theo `trigger`), **Quality** (`error_shown` theo `code`, `sync_completed` result) | Alerts trên KPI drop | | 4 dashboard có dữ liệu TestFlight; O9 evidence | |
| G03 | RevenueCat: entitlement `pro`, offering default, charts trial conversion, churn; webhook → backend 03-A16; sandbox events thấy | | | O10 | |
| G04 | ASC Analytics + Search Ads (chưa chi tiền): theo dõi impressions → page view → download; benchmark conversion ≥ 25% | Search Ads Basic khi có LTV | | O8 | |
| G05 | `docs/growth/launch.md`: ngày, kênh (cộng đồng học tiếng Anh VN, Threads/TikTok clip 30 s, Product Hunt OPT, bạn bè waitlist), thông điệp, KPI 30 ngày, ai làm gì | | | File | |
| G06 | Waitlist → launch email (Resend broadcast, có unsubscribe) gửi ngày launch; đo open/click | | | Resend broadcast stats | |
| G07 | Feedback loop: trong app "Gửi phản hồi" → email `support@` với device info; TestFlight feedback đọc hằng tuần; bảng `docs/reviews/feedback-log.md` | In-app survey (PostHog surveys) | | File có ≥ 10 dòng sau TestFlight | |
| G08 | Nhịp review tháng `docs/reviews/YYYY-MM.md`: KPI (§9 Master Plan), chi phí, incident, quyết định, top 3 việc tháng sau; chạy lại `dom-audit.sh` + scorecard | | | File tháng đầu tiên tồn tại | |

## 7. Bare Minimum vs Perfect

| Bare | Perfect |
|---|---|
| D01–D07, E01–E05, L01–L08 (L01 tự viết theo template + data inventory), A01–A12, G01–G08 (G05–G08 tối giản) | DMARC `p=reject` + BIMI, HSTS preload, subdomain gửi mail riêng, helpdesk, luật sư review + DPA, data export self-serve, iPad + localized screenshots, App Preview video, promo/intro offers, Search Ads, status page, PostHog surveys, alerts KPI |

## 8. G — Prompt cho AI (copy nguyên văn)

```
Bạn là reviewer Legal/Ops/Growth. Đầu vào: docs/checklists/06-domain-ops.md, docs/01-CONTRACTS.md §7 và §8,
docs/legal/**, docs/product/**, docs/growth/**, docs/ops/{dns-records.md,email-setup.md,accounts.md}, web/app/legal/**,
web/public/{robots.txt,.well-known/**}, ios/App/App/{PrivacyInfo.xcprivacy,Info.plist}, backend/db/migrations (để đối chiếu data inventory),
và evidence docs/evidence/domain/ nếu có (dns.txt, securityheaders.md, ssllabs.md, mail-tester.md, dmarc-report.xml, itms-warnings.txt,
licenses.md, gsc.png, posthog-*.csv, rc.csv, asc-analytics.csv) cùng screenshot ASC tôi đính kèm.
Kiểm tra từng mục D01→D08, E01→E06, L01→L08, A01→A12, G01→G08.
Với mỗi mục trả về đúng 1 dòng bảng:
ID | PASS/FAIL/N-A | bằng chứng (file:dòng, output dig/curl, hoặc mô tả screenshot) | việc cần sửa (nếu FAIL)
Quy tắc:
- Không PASS nếu không có bằng chứng. Với L01: lập bảng đối chiếu 3 chiều: cột trong schema DB ↔ data-inventory.md ↔ đoạn trong privacy.md ↔ App Privacy label; thiếu 1 chiều = FAIL L01 hoặc A02.
- Với L03: liệt kê dependency trong Package.resolved, web/package.json, backend/go.mod chưa có trong trang licenses.
- Mục [M] chỉ ghi "CẦN MANUAL".
- Cuối cùng liệt kê: mọi DNS record không nằm trong bảng Contracts §8, mọi event trong code không có trong events.yaml (và ngược lại),
  mọi URL legal chưa trả 200.
- Kết thúc bằng bảng tổng PASS/FAIL/MANUAL theo nhóm.
```

## 9. Re-check thủ công (bạn làm, ~60 phút)

1. **Registrar (D01)**: đăng nhập Cloudflare, xác nhận auto-renew + lock + 2FA; thử `whois`.
2. **Email (E03)**: từ Gmail cá nhân gửi tới 5 địa chỉ; nhận đủ trong ≤ 2 phút; reply từ hộp thư đó về Gmail; header `SPF=pass DKIM=pass DMARC=pass` (Gmail → Show original).
3. **Legal đọc thật (L01–L05)**: đọc 5 trang trên điện thoại như user; mọi email/link bấm được; ngày cập nhật đúng; tên pháp lý/tên app nhất quán với ASC.
4. **IAP sandbox (A03)**: mua monthly → huỷ trong Settings sandbox → mở app thấy hết Pro sau kỳ (sandbox 5 phút) → restore trên máy thứ 2 → có Pro lại; xem RevenueCat event + webhook tới backend (`billing_events` có dòng).
5. **Reviewer account (A07)**: đăng nhập bằng `REVIEWER_EMAIL` trên build TestFlight với OTP cố định → vào được, có deck mẫu, có Pro sandbox.
6. **Privacy khớp (A02)**: mở ASC App Privacy, đối chiếu từng mục với `app-privacy-labels.md` và trang Privacy.
7. **PostHog PII (G01)**: Live events 10 phút dùng app thật; không có email/tên trong props.

## 10. Evidence phải có khi đóng Phase 4 (trước submit)

`docs/evidence/domain/`: `<date>-dns.txt`, `-securityheaders.md`, `-observatory.md`, `-ssllabs.md`, `-mail-tester.md`, `-dmarc-report.xml`, `-gsc.png`, `-rich-results.png`, `-itms-warnings.txt`, `-licenses.md`, `-asc-privacy.png`, `-asc-iap.png`, `-rc-sandbox.png`, `-posthog-{activation,retention,monetization}.csv` (sau TestFlight), `checklist-06-run-<date>.md`.
