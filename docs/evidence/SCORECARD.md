# SCORECARD — 2026-09-10 — commit 99b67e6 — phase 0 (chưa bắt đầu)

> Mỗi ô "Đo được": `số | PASS/FAIL | link`. Ô trống = **chưa đo** (khác FAIL: FAIL là đã đo và không đạt).
> Cột "Phase đo" cho biết sớm nhất khi nào ô này có thể có số. Ô có phase 0 phải điền trước khi đóng Phase 0.
> Cách lấy số: `03-MEASUREMENT-TOOLS.md` mục tương ứng (cột Tool ID).

| Phần | Metric | Bare target | Tool ID | Phase đo | Đo được | PASS? | Evidence |
|---|---|---|---|---|---|---|---|
| Design | Contrast mọi pair text semantic | ≥ 4.5 (text), ≥ 3.0 (large) | 02-DS 06 | 0 | | | design/<date>-contrast.txt |
| Contract | `redocly lint` errors | 0 | 01 §1 | 0 | | | (CI log contract.yml) |
| Contract | Drift `make gen && git diff` | 0 file | 01 §1 | 0 | | | (CI log contract.yml) |
| DB | squawk errors migration 0001 | 0 | D4 | 0 | | | db/<date>-migration-0001.txt |
| Web | Lighthouse mobile Perf/A11y/BP/SEO (landing) | ≥95/≥95/100/100 | W1 | 0 | | | web/<date>-lhci-landing/ |
| Web | LCP / CLS / TBT lab (landing) | ≤2.0 s / ≤0.05 / ≤150 ms | W1 | 0 | | | web/<date>-lhci-landing/ |
| Domain | DNSSEC / CAA | secure / có | O4 | 0 | | | domain/<date>-dns.txt |
| Domain | securityheaders apex | A | O1 | 0 | | | domain/<date>-securityheaders.md |
| Domain | mail-tester (mail waitlist qua Resend) | ≥ 9/10 | O5 | 0 | | | domain/<date>-mail-tester.md |
| Product | Waitlist đăng ký | ≥ ngưỡng PRD (gợi ý 30) | PostHog/Resend | 0 | | | domain/<date>-waitlist.csv |
| Backend | k6 p95 / p99 @200 VU | ≤120 / ≤300 ms | B1 | 1 | | | backend/<date>-k6-study-flow.json |
| Backend | Error rate under load | < 0.1% | B1 | 1 | | | (cùng file) |
| Backend | Race / coverage domain | 0 / ≥ 80% | B3 | 1 | | | backend/<date>-cover.txt |
| Backend | schemathesis failures | 0 | B5 | 1 | | | backend/<date>-schemathesis.txt |
| Backend | govulncheck / HIGH CVE image | 0 / 0 | B6, B8 | 1 | | | backend/<date>-govulncheck.txt, -trivy.json |
| Backend | Image size | ≤ 25 MB | B8 | 1 | | | backend/<date>-image.txt |
| DB | Seq scan trên bảng > 10k | 0 | D2, D5 | 1 | | | db/<date>-explain/ |
| DB | Max mean_exec_time query app | ≤ 20 ms | D1 | 1 | | | db/<date>-pg-stat-statements.csv |
| DB | Restore drill RTO | ≤ 30 phút | D7 | 1 | | | db/<date>-restore-drill.md |
| DB | Tổng size sau import | ≤ 400 MB | D6 | 1 | | | db/<date>-size.md |
| Server | Lynis hardening index | ≥ 75 | S1 | 1 | | | server/<date>-lynis.txt |
| Server | Port mở (nmap từ ngoài) | 22/80/443 | S3 | 1 | | | server/<date>-nmap.txt |
| Server | SSL Labs `api.` | A+ | S4 | 1 | | | server/<date>-testssl.json |
| Server | goss pass | 100% | S6 | 1 | | | server/<date>-goss.json |
| iOS | Cold launch median (Release, iPhone <model>) | ≤ 400 ms | I1, I2 | 2 | | | ios/<date>-perf-metrics.json |
| iOS | Hitch ratio list 1000 | < 5 ms/s | I3 | 2 | | | ios/<date>-hitches.md |
| iOS | Leaks sau 20 vòng Study | 0 | I4 | 2 | | | ios/<date>-leaks.md |
| iOS | Download size | ≤ 30 MB | I7 | 2 | | | ios/<date>-size.txt |
| iOS | A11y audit errors | 0 | I8 | 2 | | | ios/<date>-a11y-audit.md |
| iOS | Crash-free 7 ngày TestFlight | ≥ 99.5% | I5, I6 | 2 | | | ios/<date>-sentry.png |
| Web | First Load JS landing | ≤ 90 KB | W6 | 3 | | | web/<date>-size-limit.json |
| Web | axe serious+critical | 0 | W7 | 3 | | | web/<date>-axe.json |
| Web | securityheaders `admin.` | A | O1 | 3 | | | domain/<date>-securityheaders.md |
| Server | Uptime 30 ngày | ≥ 99.5% | S8 | 4 | | | server/<date>-uptime.csv |
| Server | Deploy time / rollback tested | ≤ 3 phút / yes | S9 | 4 | | | server/<date>-deploy-drill.md |
| Domain | ITMS warnings | 0 | O8 | 4 | | | domain/<date>-itms-warnings.txt |
| Domain | Licenses coverage | 100% | O10+ (licensee, 03 §9) | 4 | | | domain/<date>-licenses.md |
| Product | D1/D7/D30 retention | 35/15/8% | O9 | 5 | | | domain/<date>-posthog-retention.csv |

## Lịch sử

| Ngày | Commit | Phase | Ô đã điền | Ô FAIL | Ghi chú |
|---|---|---|---|---|---|
| 2026-09-10 | 99b67e6 | 0 | 0/38 | 0 | Khởi tạo file. Chưa có gì để đo |
