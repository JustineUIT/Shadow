# 03 — MEASUREMENT TOOLS: Chứng minh bằng số liệu thật

> Nguyên tắc 2 của Master Plan: **không có số = chưa xong**. File này định nghĩa cho mỗi phần (iOS, Web, Backend, DB, Server, Domain) một **tool chính** (primary) và các tool phụ, lệnh chạy, file evidence phải sinh ra, và ngưỡng PASS.
> Mọi evidence lưu trong `docs/evidence/<phần>/<YYYY-MM-DD>-<tool>.<ext>` kèm **commit hash** đo. Scorecard tổng ở §8.

## 0. Quy tắc chung khi đo

| Quy tắc | Lý do |
|---|---|
| Đo trên **môi trường gần prod nhất**: iOS máy thật (không simulator) ở chế độ Release; web trên URL Cloudflare Pages preview/prod; backend trên staging VPS thật | Số trên máy dev vô nghĩa |
| Chạy **tối thiểu 5 lần**, lấy **median** (không lấy best) cho metric thời gian | Loại nhiễu |
| Ghi **commit hash + ngày + thiết bị/region** vào đầu mỗi file evidence | Tái lập được |
| Cùng một script đo cho mọi lần đo, script commit trong repo (`scripts/measure/*`) | So sánh trước/sau công bằng |
| Số **field** (user thật: MetricKit, CrUX, Grafana) thắng số **lab** (Instruments, Lighthouse, k6) nếu mâu thuẫn | Lab để debug, field để kết luận |
| Tool phải in ra **số** (ms, MB, %, điểm). Tool chỉ báo "ok/warn" không được dùng làm bằng chứng chính | Yêu cầu của bạn: số liệu thật |

Thư mục:

```
docs/evidence/
├── SCORECARD.md                 # §8, điền cuối mỗi phase
├── ios/       web/      backend/     db/      server/     domain/
scripts/measure/
├── ios-launch.sh  ios-size.sh
├── web-lhci.sh    web-bundle.sh
├── be-k6.sh       be-pprof.sh    be-bench.sh
├── db-explain.sh  db-stats.sql   db-restore-drill.sh
├── srv-audit.sh   srv-external-scan.sh
└── dom-audit.sh
```

---

## 1. iOS Swift — tool chính: **Xcode Instruments + XCTest Performance Metrics** (lab) và **MetricKit** (field)

| # | Tool | Đo cái gì | Cách chạy (tóm tắt) | Output → evidence | Target Bare | Target Perfect |
|---|---|---|---|---|---|---|
| I1 | **XCTest Performance** (`XCTApplicationLaunchMetric`, `XCTMemoryMetric`, `XCTClockMetric`, `XCTOSSignpostMetric`, `XCTStorageMetric`) với `.xcbaseline` | Launch time, memory peak, thời gian mở Study, thời gian sync 100 review | `xcodebuild test -scheme App -testPlan Performance -destination 'id=<UDID máy thật>' -resultBundlePath out.xcresult` rồi `xcrun xcresulttool get --format json --path out.xcresult` | `ios/<date>-perf-metrics.json` + baseline commit trong repo | Cold launch (Release) **≤ 400 ms**; mở Study ≤ 300 ms; peak memory Study ≤ 150 MB; baseline không regress > 10% | Cold launch ≤ 250 ms; peak ≤ 100 MB |
| I2 | **Instruments — App Launch template** | Pre-main (dyld) vs post-main, `didFinishLaunching` | Product → Profile → App Launch, máy thật, Release, 5 lần | Screenshot + export `.trace` nén; ghi bảng số vào `ios/<date>-launch.md` | Pre-main ≤ 100 ms (ít dynamic framework, static SPM); tổng ≤ 400 ms | ≤ 250 ms |
| I3 | **Instruments — Animation Hitches** | Hitch rate khi cuộn list 1000 deck/card, lật thẻ | Template Animation Hitches, cuộn 10 s | `ios/<date>-hitches.md`: hitch time ratio ms/s | **< 5 ms/s** (Apple "good") | < 1 ms/s |
| I4 | **Instruments — Allocations + Leaks** | Leak, abandoned memory sau 20 lần vào/ra Study | Template Leaks, generation mark trước/sau | `ios/<date>-leaks.md` | **0 leak**; abandoned growth < 1 MB / 20 vòng | 0 |
| I5 | **MetricKit** (`MXMetricManager` subscriber gửi payload lên Sentry hoặc endpoint riêng) | Field: launch time p50/p95, hang rate, crash, disk write, battery | Code trong `Packages/Core/Observability`; payload đến hằng ngày | `ios/<date>-metrickit-summary.md` (tổng hợp 7 ngày) | Launch p95 ≤ 600 ms field; hang rate ≤ 0.5%; crash-free ≥ 99.5% | p95 ≤ 400 ms; hang ≤ 0.1%; crash-free ≥ 99.8% |
| I6 | **Xcode Organizer** (Launch Time, Hang Rate, Disk Writes, Battery, Terminations) | Số Apple thu từ user opt-in | Window → Organizer → Metrics | Screenshot vào `ios/<date>-organizer/*.png` | Không có regression đỏ giữa 2 version | |
| I7 | **App Size Report** | Download size, install size theo thiết bị | `xcodebuild -exportArchive` với `-exportOptionsPlist` có `thinning: <thin-for-all-variants>` → `App Thinning Size Report.txt`; hoặc TestFlight → Build → App Size | `ios/<date>-size.txt` | Download **≤ 30 MB**, install ≤ 80 MB (audio không bundle, tải từ CDN) | ≤ 15 MB |
| I8 | **Accessibility Inspector — Audit** | Contrast, label thiếu, hit region, Dynamic Type clipping | Xcode → Open Developer Tool → Accessibility Inspector → Audit trên 5 màn | `ios/<date>-a11y-audit.md` + png | **0 issue** loại Error | 0 warning |
| I9 | **swiftlint --strict**, **swift-format lint**, **periphery scan** | Lint, style, dead code | `swiftlint lint --strict --reporter json > out.json`; `periphery scan --format json` | `ios/<date>-lint.json`, `<date>-periphery.json` | 0 vi phạm lint; dead code ≤ 5 item (ghi lý do) | 0 |
| I10 | **Build timing** | Tổng build clean, file chậm nhất | `xcodebuild -showBuildTimingSummary`; `-Xfrontend -warn-long-expression-type-checking=100` | `ios/<date>-build-timing.txt` | Clean build ≤ 120 s trên máy dev; 0 expression > 100 ms | Incremental ≤ 10 s |
| I11 | **Sentry (iOS)** | Crash-free sessions, top issues, dSYM có | Dashboard Releases | Screenshot vào `ios/<date>-sentry.png` | Crash-free ≥ 99.5% sau 7 ngày TestFlight | ≥ 99.8% |

Script `scripts/measure/ios-launch.sh` (mô tả): nhận UDID, build Release, chạy test plan Performance 5 lần, gộp JSON, in bảng median, ghi file evidence có commit hash.

---

## 2. Web React — tool chính: **Lighthouse CI (lhci) với performance budgets** (lab) và **CrUX / web-vitals** (field)

| # | Tool | Đo cái gì | Cách chạy | Output → evidence | Target Bare | Target Perfect |
|---|---|---|---|---|---|---|
| W1 | **@lhci/cli autorun** với `lighthouserc.json` (assertions + budgets) | Performance/A11y/Best Practices/SEO, LCP, CLS, TBT, bundle budgets | `lhci autorun --collect.url=<preview url> --collect.numberOfRuns=5` (mobile preset, throttling simulated) | `web/<date>-lhci/*.json` + `manifest.json`; link temporary-public-storage hoặc LHCI server | Mobile median: **Perf ≥ 95, A11y ≥ 95, BP = 100, SEO = 100**; LCP ≤ 2.0 s; CLS ≤ 0.05; TBT ≤ 150 ms | Perf ≥ 98; LCP ≤ 1.2 s; CLS = 0 |
| W2 | **Performance budgets** (`budget.json` dùng bởi lhci) | Tổng JS, CSS, image, font, third-party, số request | Cùng lệnh W1 | trong W1 | Landing: JS ≤ 90 KB gzip, CSS ≤ 20 KB, font ≤ 100 KB, requests ≤ 30, third-party ≤ 1 (PostHog) | JS ≤ 50 KB |
| W3 | **WebPageTest** (public hoặc API) | Waterfall thật từ Singapore/HCM, 4G, filmstrip, Speed Index | webpagetest.org: location Singapore, Moto G4 / 4G, 3 runs, First + Repeat view | `web/<date>-wpt.md` (link kết quả + số) | Speed Index ≤ 2.5 s; Start Render ≤ 1.5 s; TTFB ≤ 400 ms | SI ≤ 1.5 s |
| W4 | **CrUX / PageSpeed Insights API** (`https://www.googleapis.com/pagespeedonline/v5/runPagespeed?url=&strategy=mobile`) | Field p75 LCP/INP/CLS 28 ngày (khi đủ traffic) | `curl` → jq `.loadingExperience.metrics` | `web/<date>-psi.json` | Khi có dữ liệu: LCP p75 ≤ 2.5 s, INP ≤ 200 ms, CLS ≤ 0.1 ("Good") | LCP ≤ 1.8 s, INP ≤ 100 ms |
| W5 | **web-vitals** lib → PostHog event `web_vital` | Field vitals từ ngày đầu (không cần đợi CrUX) | `onLCP/onINP/onCLS` trong `src/lib/vitals.ts`, gửi `{name, value, rating, path}` | Dashboard PostHog + export CSV `web/<date>-vitals.csv` | Như W4 | |
| W6 | **Bundle analysis**: `@next/bundle-analyzer` + **size-limit** (CI) | Kích thước từng route, thư viện nặng | `ANALYZE=true next build`; `pnpm size-limit --json` | `web/<date>-size-limit.json`, `analyze/client.html` | First Load JS landing ≤ 90 KB; admin ≤ 250 KB; size-limit fail → PR đỏ | |
| W7 | **Playwright + @axe-core/playwright** | A11y violation mỗi route; keyboard nav; screenshot 3 viewport | `pnpm playwright test --reporter=json` | `web/<date>-playwright.json`, `<date>-axe.json` | **0 serious/critical** trên mọi route; test keyboard pass | 0 moderate |
| W8 | **unlighthouse** (scan toàn site) | Điểm mọi URL, tìm trang lọt lưới (legal, admin login) | `npx unlighthouse --site <url> --budget 95` | `web/<date>-unlighthouse/ci-result.json` | Mọi route public ≥ 95 perf | |
| W9 | **tsc --noEmit**, **eslint --max-warnings 0**, **knip** (dead exports/deps) | Type, lint, dependency thừa | lệnh tương ứng `--format json` | `web/<date>-lint.json`, `<date>-knip.json` | 0 error/warning; knip 0 unused dep | |
| W10 | **Cloudflare Pages Web Analytics / Cache analytics** | Cache hit ratio, request theo country | Dashboard | Screenshot `web/<date>-cf-cache.png` | Static asset cache hit ≥ 95% | |

Script `scripts/measure/web-lhci.sh`: nhận URL, chạy lhci 5 lần mobile, in bảng median 4 điểm + LCP/CLS/TBT, copy report vào evidence.

---

## 3. Backend Go — tool chính: **k6** (load, p95/p99) + **pprof** (CPU/heap) + **go test -race -cover**

| # | Tool | Đo cái gì | Cách chạy | Output → evidence | Target Bare | Target Perfect |
|---|---|---|---|---|---|---|
| B1 | **k6** scenario `study-flow.js` (login → queue → 20 reviews batch → stats), `dictionary-search.js`, `auth-refresh.js`, có `thresholds` | p50/p95/p99 latency, RPS, error rate, per-endpoint tags | `k6 run --out json=out.json -e BASE=https://api-staging.<domain> loadtest/k6/study-flow.js` với 200 VU × 5 phút, ramp 1 phút | `backend/<date>-k6-<scenario>.json` + `summary.txt` (handleSummary) | @200 VU: **p95 ≤ 120 ms, p99 ≤ 300 ms** (không tính network), error rate < 0.1%, RPS ≥ 500; `/dictionary/search` p95 ≤ 40 ms qua HTTP | p95 ≤ 60 ms @500 VU |
| B2 | **pprof** (`net/http/pprof` chỉ bind cổng nội bộ / qua SSH tunnel) trong lúc chạy B1 | CPU top, heap inuse, goroutine count, alloc/op hot path | `go tool pprof -top -seconds=30 http://localhost:6060/debug/pprof/profile`; `.../heap`; `.../goroutine?debug=1` | `backend/<date>-pprof-cpu.txt`, `-heap.txt`, `-goroutines.txt` | Không hàm nào của mình > 20% CPU ngoài JSON/pgx; heap inuse ổn định (không tăng tuyến tính 5 phút); goroutine ≤ 200 sau load | |
| B3 | **go test -race -covermode=atomic -coverprofile** + `go tool cover -func` | Race, coverage domain packages | `go test -race -covermode=atomic -coverprofile=cover.out ./... && go tool cover -func=cover.out \| tail -1` | `backend/<date>-cover.txt` | **0 race**; coverage `internal/{auth,study,deck,billing}` ≥ 80% dòng | ≥ 90% |
| B4 | **go test -bench + benchstat** | FSRS schedule/op, JWT verify/op, cursor encode | `go test -bench=. -benchmem -count=10 ./internal/study/... > new.txt; benchstat old.txt new.txt` | `backend/<date>-bench.txt` | FSRS `Schedule` ≤ 2 µs/op, 0 alloc; JWT verify ≤ 60 µs/op; benchstat không regress > 10% (p < 0.05) | |
| B5 | **schemathesis** (contract fuzz từ `openapi.yaml`) | Response đúng schema, 5xx, header đúng | `schemathesis run https://api-staging.<domain>/v1/openapi.yaml --checks all --hypothesis-max-examples=200 --header "Authorization: Bearer $T" --report` | `backend/<date>-schemathesis.txt` | **0 failure** trên endpoint MUST; 0 5xx | |
| B6 | **govulncheck** + **golangci-lint** (`.golangci.yml` bật `errcheck, govet, staticcheck, gosec, bodyclose, sqlclosecheck, noctx, errorlint, exhaustive, gocritic, revive`) | CVE trong call graph; lint | `govulncheck ./... ; golangci-lint run --out-format json` | `backend/<date>-govulncheck.txt`, `<date>-golangci.json` | 0 vuln reachable; 0 lint issue | |
| B7 | **Grafana Cloud (Tempo/Prometheus từ OTel)** | p95 thật theo endpoint 7 ngày, DB span share, error rate | Dashboard `infra/grafana/dashboards/api.json` | Screenshot + export CSV `backend/<date>-grafana-api.csv` | p95 field ≤ 200 ms (gồm network trong VN→SG); 5xx < 0.1% | |
| B8 | **Docker image**: `docker image inspect --format '{{.Size}}'` + **dive** + **trivy image** | Size, layer thừa, CVE | `dive --ci --lowestEfficiency=0.95 img`; `trivy image --severity HIGH,CRITICAL --exit-code 1 img` | `backend/<date>-image.txt`, `<date>-trivy.json` | Image **≤ 25 MB**, 0 HIGH/CRITICAL | ≤ 15 MB |
| B9 | **gocyclo / gocognit** (qua golangci) | Độ phức tạp hàm | trong B6 | trong B6 | Không hàm > 15 cyclomatic (trừ generated) | ≤ 10 |
| B10 | **hey** hoặc **vegeta** (kiểm tra nhanh 1 endpoint) | Sanity trước khi chạy k6 dài | `vegeta attack -rate=200 -duration=30s -targets=t.txt \| vegeta report -type=json` | `backend/<date>-vegeta-<ep>.json` | Cùng ngưỡng B1 | |

Script `scripts/measure/be-k6.sh`: lấy token test user, chạy 3 scenario tuần tự, gộp `summary.txt` có bảng p50/p95/p99 theo endpoint, gắn commit hash từ `GET /v1/version`.

---

## 4. Database Postgres (Neon) — tool chính: **pg_stat_statements + EXPLAIN (ANALYZE, BUFFERS)** và **pgbench**

| # | Tool | Đo cái gì | Cách chạy | Output → evidence | Target Bare | Target Perfect |
|---|---|---|---|---|---|---|
| D1 | **pg_stat_statements** (bật trên Neon: `CREATE EXTENSION`) | Top query theo `total_exec_time`, `mean_exec_time`, `calls`, `rows`, cache hit | `scripts/measure/db-stats.sql`: reset trước k6, chạy B1, rồi select top 20 | `db/<date>-pg-stat-statements.csv` | Không query app nào `mean_exec_time` > 20 ms; queue query ≤ 10 ms; search ≤ 15 ms | ≤ 5 ms |
| D2 | **EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON)** cho 8 query hot path (queue, reviews insert, stats range, search prefix, search trgm, decks list cursor, cards list, entitlement check) | Plan, seq scan, rows estimate vs actual, shared hit/read | `scripts/measure/db-explain.sh` chạy 8 file `.sql` với tham số user test có 10k review | `db/<date>-explain/<query>.json` + tóm tắt `.md` | **0 Seq Scan** trên bảng > 10k dòng; estimate/actual lệch < 10×; buffers read ≈ 0 lần 2 | |
| D3 | **pgbench** custom script (`-f review_insert.sql`) trên Neon branch | TPS insert review, p95 latency | `pgbench -c 20 -j 4 -T 60 -f loadtest/pgbench/review_insert.sql -r` | `db/<date>-pgbench.txt` | ≥ 500 TPS insert review; latency avg ≤ 20 ms (pooled, cùng region) | |
| D4 | **squawk** (lint migration) | Migration nguy hiểm: lock, không `CONCURRENTLY`, NOT NULL không default, đổi type | `squawk backend/db/migrations/*.sql --reporter json` | `db/<date>-squawk.json` | **0 error** | 0 warning |
| D5 | **Catalog queries** (`pg_stat_user_tables`, `pg_stat_user_indexes`, `pg_indexes`, `pg_stat_database`) | Seq scan count, unused index, index size, cache hit ratio, dead tuples | trong `db-stats.sql` | `db/<date>-catalog.md` | Cache hit ratio ≥ 99%; 0 index unused sau 7 ngày traffic (trừ unique/PK); dead tuple ratio < 10% | |
| D6 | **Kích thước**: `pg_database_size`, `pg_total_relation_size` top 10 | Fit free tier Neon | trong `db-stats.sql` | `db/<date>-size.md` | Tổng ≤ 400 MB (DATA-06) hoặc có ADR đổi plan | |
| D7 | **Restore drill có đồng hồ** (`scripts/measure/db-restore-drill.sh`) | RTO thật: tải dump từ R2 → giải mã age → `pg_restore` vào Neon branch → chạy smoke query → so `count(*)` | script in thời gian từng bước | `db/<date>-restore-drill.md` | Dump nightly tồn tại ≤ 24 h tuổi; **RTO ≤ 30 phút**; count khớp; drill mỗi tháng | RTO ≤ 10 phút; PITR test |
| D8 | **Neon console metrics** (CPU, connections, storage, branch) + `pg_stat_activity` | Connection peak so pool max, idle in transaction | screenshot + `select state, count(*)` trong load | `db/<date>-neon-metrics.png` | Peak conn ≤ 10 (pool) + 2 worker; 0 `idle in transaction` > 5 s | |
| D9 | **hypopg** (nếu có) hoặc so `EXPLAIN` trước/sau thêm index trên branch | Chứng minh index đáng thêm | tạo Neon branch, chạy D2 trước/sau | trong D2 | Mỗi index thêm có số trước/sau trong ADR | |
| D10 | **Import report** từ ETL (`tools/import-dictionary` in thống kê) | Số entry, % IPA, % ví dụ, duplicate, thời gian import | `app import --report` | `db/import-report.md` | Duplicate = 0; ≥ 95% top 5000 có IPA + ≥ 1 ví dụ + audio | |

---

## 5. Server / Infra — tool chính: **Lynis** (hardening score) + **Trivy** + **nmap từ ngoài** + **Grafana node metrics**

| # | Tool | Đo cái gì | Cách chạy | Output → evidence | Target Bare | Target Perfect |
|---|---|---|---|---|---|---|
| S1 | **Lynis** | Hardening index, warnings, suggestions | `sudo lynis audit system --quiet --report-file /tmp/lynis.dat; grep hardening_index /tmp/lynis.dat` | `server/<date>-lynis.txt` (report + top suggestions) | **Hardening index ≥ 75**, 0 warning nghiêm trọng (SSH, firewall, kernel) | ≥ 85 |
| S2 | **ssh-audit** (từ máy ngoài) | Cipher/KEX/MAC yếu, banner | `ssh-audit -p <port> <ip> --json` | `server/<date>-ssh-audit.json` | 0 fail; chỉ ed25519 host key; password auth off | |
| S3 | **nmap** từ máy ngoài | Port mở thật | `nmap -sS -sV -p- --open <ip>` (và `-sU --top-ports 50`) | `server/<date>-nmap.txt` | Chỉ **22 (hoặc port đổi), 80, 443** mở; không thấy 5432/6060/9100/2019 | Chỉ 443 + 80 (SSH qua Tailscale) |
| S4 | **testssl.sh** hoặc **SSL Labs API** cho `api.<domain>` origin và qua Cloudflare | Protocol, cipher, HSTS, OCSP, cert chain | `testssl.sh --jsonfile out.json https://api.<domain>` | `server/<date>-testssl.json` | SSL Labs **A+**; TLS 1.2+1.3 only; HSTS ≥ 1 năm | |
| S5 | **Trivy** (`image` + `fs` + `config` cho compose/Dockerfile/Caddyfile) + **dockle** | CVE, misconfig, image best practice | `trivy config infra/; trivy image ...; dockle img` | `server/<date>-trivy-config.json`, `<date>-dockle.json` | 0 HIGH/CRITICAL; dockle 0 FATAL/WARN | |
| S6 | **goss** (`infra/goss.yaml`: user, sshd config, ufw rules, fail2ban, docker rootless/hardened, unattended-upgrades, cron backup, thời gian) | Server đúng spec sau bootstrap, kiểm tra idempotent | `goss -g infra/goss.yaml validate --format json` | `server/<date>-goss.json` | **100% pass** sau `bootstrap-vps.sh` chạy 2 lần | |
| S7 | **Grafana Cloud + node_exporter/Alloy + cAdvisor** | CPU, RAM, disk, IO wait, container mem, restart count, trong lúc k6 | Dashboard `infra/grafana/dashboards/node.json` | `server/<date>-node-under-load.png` + CSV | @200 VU: CPU ≤ 70%, RAM api container ≤ 256 MB, IO wait < 5%, 0 restart, disk free ≥ 30% | |
| S8 | **UptimeRobot / healthchecks.io** | Uptime % 30 ngày, thời gian phát hiện sự cố, cron backup có chạy | Dashboard + export | `server/<date>-uptime.csv` | Uptime ≥ 99.5%; alert đến điện thoại ≤ 5 phút (drill OPS-04) | ≥ 99.9% |
| S9 | **Deploy drill có đồng hồ** (`deploy.sh` in timestamp) | Thời gian deploy, downtime quan sát bằng `hey -z 3m` chạy song song | chạy deploy khi hey đang bắn | `server/<date>-deploy-drill.md` | Deploy ≤ 3 phút; non-2xx trong deploy ≤ 30 request (Bare chấp nhận vài giây downtime); rollback tự động khi `/readyz` fail đã test | 0 non-2xx (blue/green) |
| S10 | **docker stats / docker inspect** | Container read-only, cap drop, no-new-privileges, mem limit | `docker inspect api --format '{{json .HostConfig}}' \| jq '{ReadonlyRootfs,CapDrop,SecurityOpt,Memory}'` | `server/<date>-container-hardening.json` | `ReadonlyRootfs=true`, `CapDrop=["ALL"]`, `no-new-privileges`, Memory ≤ 512 MB | |
| S11 | **Chi phí**: hoá đơn Hetzner/Oracle, Neon, R2, Cloudflare | Tiền/tháng thật | screenshot hoá đơn | `server/<date>-cost.md` | Theo bảng §8 Master Plan | |

Script `scripts/measure/srv-audit.sh` chạy trên VPS: Lynis + goss + docker inspect + trivy config, gom về 1 thư mục và `scp` xuống. `srv-external-scan.sh` chạy từ laptop: nmap + ssh-audit + testssl.

---

## 6. Domain / Ops / Legal / Growth — tool chính: **SSL Labs + securityheaders.com + DNSViz + mail-tester** (kỹ thuật) và **App Store Connect + PostHog + RevenueCat** (sản phẩm)

| # | Tool | Đo cái gì | Cách chạy | Output → evidence | Target Bare | Target Perfect |
|---|---|---|---|---|---|---|
| O1 | **securityheaders.com** (`https://securityheaders.com/?q=<domain>&followRedirects=on`) cho `<domain>`, `admin.`, `api.` | Điểm header bảo mật | curl hoặc trình duyệt; lưu ảnh | `domain/<date>-securityheaders.md` | **A** cho cả 3 host (CSP report-only vẫn A nếu có các header khác) | A+ (CSP enforcing) |
| O2 | **Mozilla Observatory** (`https://developer.mozilla.org/en-US/observatory`) | Điểm tổng hợp, CSP, cookie, redirect | web | `domain/<date>-observatory.md` | ≥ B+ | A+ |
| O3 | **SSL Labs** cho `<domain>` và `api.` (qua Cloudflare) | Grade, HSTS preload ready | web hoặc `ssllabs-scan` CLI | `domain/<date>-ssllabs.md` | A+ | HSTS preload list |
| O4 | **DNSViz** + `dig +dnssec`, `delv` | DNSSEC chain hợp lệ, CAA, TTL | `delv @1.1.1.1 <domain> A +rtrace`; `dig CAA <domain>` | `domain/<date>-dns.txt` | DNSSEC "secure"; CAA có `letsencrypt.org` + `pki.goog`/`digicert.com` (Cloudflare) | |
| O5 | **mail-tester.com** + **MXToolbox** (`SPF, DKIM, DMARC, blacklist`) + **dmarcian XML report** | Deliverability score email OTP/waitlist qua Resend | Gửi OTP tới địa chỉ mail-tester; `dig TXT _dmarc.<domain>` | `domain/<date>-mail-tester.md`, `<date>-dmarc-report.xml` | mail-tester **≥ 9/10**; SPF pass, DKIM pass, DMARC `p=quarantine` ít nhất; 0 blacklist | DMARC `p=reject`, BIMI |
| O6 | **Google Search Console + sitemap + `curl -I` robots** | Index status, coverage errors, Core Web Vitals field | Console; `curl -s <domain>/sitemap.xml \| xmllint --noout -` | `domain/<date>-gsc.png` | 0 coverage error; mọi URL public indexed; legal pages indexed | Rich result cho FAQ |
| O7 | **Lighthouse SEO + Best Practices** (đã có trong W1) và **Rich Results Test** | Structured data (Organization, SoftwareApplication, FAQPage) | `https://search.google.com/test/rich-results` | `domain/<date>-rich-results.png` | 0 error | |
| O8 | **App Store Connect — App Analytics** | Impressions → Product page view → Download (conversion), crash, retention D1/D7/D30 theo Apple | ASC → Analytics → Metrics, export CSV | `domain/<date>-asc-analytics.csv` | Conversion ≥ 25%; retention theo KPI §9 Master Plan | |
| O9 | **PostHog** (funnel `app_opened → signup_completed → study_session_completed`, retention, paywall funnel) từ `events.yaml` | KPI sản phẩm thật | Dashboard `Activation`, `Retention`, `Monetization` | export CSV `domain/<date>-posthog-*.csv` | Install→signup ≥ 50%; signup→first session ≥ 60%; D1/D7/D30 ≥ 35/15/8% | 45/25/12 |
| O10 | **RevenueCat Charts** (trial start, trial conversion, MRR, churn, refund) | Tiền thật | Charts → export | `domain/<date>-rc.csv` | Trial→paid ≥ 25%; refund < 5% | ≥ 40% |
| O11 | **App Review checklist run** (`checklists/06` mục A) + **Privacy Manifest validator** (`xcrun` build warning `ITMS-91053`) | Reject risk | Upload TestFlight, đọc email ITMS warnings | `domain/<date>-itms-warnings.txt` | 0 ITMS warning về privacy manifest / required reason API | |
| O12 | **License compliance**: `licensee` (web/backend) + SPM `license-plist` (iOS) | Mọi dependency có license tương thích, trang Licenses đủ | `licensee detect .`; `license-plist --output-path ...` | `domain/<date>-licenses.md` | 0 dependency GPL/AGPL trong app; trang licenses liệt kê 100% | |
| O13 | **Status page** (UptimeRobot public page hoặc Uptime Kuma) | Uptime công khai | | `domain/<date>-status.png` | Có URL `status.<domain>` (Perfect) | |

Script `scripts/measure/dom-audit.sh`: chạy `dig`/`delv`/`curl -I` cho 4 host, gọi securityheaders và PSI API, ghi 1 file markdown tổng.

---

## 7. Lịch đo

| Khi nào | Chạy gì | Ai |
|---|---|---|
| Mỗi PR (CI) | B3, B6, B8-trivy, W1 (preview URL), W6 size-limit, W7 axe, W9, D4 squawk, I9 lint, O12 | CI tự động; fail → PR đỏ |
| Nightly (GitHub Actions `nightly-loadtest.yml`) | B1 (100 VU rút gọn 2 phút trên staging), D1, B5 schemathesis | CI, đăng kết quả vào issue |
| Cuối mỗi phase | Toàn bộ mục của phần liên quan + điền §8 | Bạn |
| Trước submit App Store | Toàn bộ 6 phần | Bạn |
| Hằng tháng (Phase 5) | I5, I11, W4, B7, D5–D7, S1, S3, S8, O5, O8–O10 | Bạn; ghi `docs/reviews/YYYY-MM.md` |

---

## 8. SCORECARD template (`docs/evidence/SCORECARD.md`)

Mỗi ô: `số đo | PASS/FAIL | link evidence`. Ô trống = chưa đo = FAIL.

```
# SCORECARD — <date> — commit <hash> — phase <n>

| Phần | Metric | Bare target | Đo được | PASS? | Evidence |
|---|---|---|---|---|---|
| iOS | Cold launch median (Release, iPhone <model>) | ≤ 400 ms | | | ios/<date>-perf-metrics.json |
| iOS | Hitch ratio list 1000 | < 5 ms/s | | | ios/<date>-hitches.md |
| iOS | Leaks sau 20 vòng Study | 0 | | | ios/<date>-leaks.md |
| iOS | Download size | ≤ 30 MB | | | ios/<date>-size.txt |
| iOS | Crash-free 7 ngày | ≥ 99.5% | | | ios/<date>-sentry.png |
| iOS | A11y audit errors | 0 | | | ios/<date>-a11y-audit.md |
| Web | Lighthouse mobile Perf/A11y/BP/SEO | ≥95/≥95/100/100 | | | web/<date>-lhci/ |
| Web | LCP / CLS / TBT lab | ≤2.0 s / ≤0.05 / ≤150 ms | | | web/<date>-lhci/ |
| Web | First Load JS landing | ≤ 90 KB | | | web/<date>-size-limit.json |
| Web | axe serious+critical | 0 | | | web/<date>-axe.json |
| Backend | k6 p95 / p99 @200 VU | ≤120 / ≤300 ms | | | backend/<date>-k6-study-flow.json |
| Backend | Error rate under load | < 0.1% | | | (cùng file) |
| Backend | Race / coverage domain | 0 / ≥ 80% | | | backend/<date>-cover.txt |
| Backend | schemathesis failures | 0 | | | backend/<date>-schemathesis.txt |
| Backend | govulncheck / HIGH CVE image | 0 / 0 | | | backend/<date>-govulncheck.txt, -trivy.json |
| Backend | Image size | ≤ 25 MB | | | backend/<date>-image.txt |
| DB | Seq scan trên bảng > 10k | 0 | | | db/<date>-explain/ |
| DB | Max mean_exec_time query app | ≤ 20 ms | | | db/<date>-pg-stat-statements.csv |
| DB | squawk errors | 0 | | | db/<date>-squawk.json |
| DB | Restore drill RTO | ≤ 30 phút | | | db/<date>-restore-drill.md |
| DB | Tổng size | ≤ 400 MB | | | db/<date>-size.md |
| Server | Lynis hardening index | ≥ 75 | | | server/<date>-lynis.txt |
| Server | Port mở (nmap ngoài) | 22/80/443 | | | server/<date>-nmap.txt |
| Server | SSL Labs api. | A+ | | | server/<date>-testssl.json |
| Server | goss pass | 100% | | | server/<date>-goss.json |
| Server | Uptime 30 ngày | ≥ 99.5% | | | server/<date>-uptime.csv |
| Server | Deploy time / rollback tested | ≤ 3 phút / yes | | | server/<date>-deploy-drill.md |
| Domain | securityheaders 3 host | A | | | domain/<date>-securityheaders.md |
| Domain | DNSSEC / CAA | secure / có | | | domain/<date>-dns.txt |
| Domain | mail-tester | ≥ 9/10 | | | domain/<date>-mail-tester.md |
| Domain | ITMS warnings | 0 | | | domain/<date>-itms-warnings.txt |
| Domain | Licenses coverage | 100% | | | domain/<date>-licenses.md |
| Product | D1/D7/D30 | 35/15/8% | | | domain/<date>-posthog-retention.csv |
```

---

## 9. Bảng cài đặt tool (1 lần, ghi vào `docs/ops/tooling.md`)

| Tool | Cài | Chạy ở đâu |
|---|---|---|
| Xcode + Instruments, Accessibility Inspector | App Store | Mac |
| swiftlint, swift-format, periphery, license-plist | `brew install swiftlint swift-format peripheryapp/periphery/periphery mono0926/license-plist/license-plist` | Mac |
| @lhci/cli, unlighthouse, size-limit, playwright, @axe-core/playwright, knip | `pnpm add -D` trong `web/` | Mac + CI |
| k6 | `brew install k6` / GitHub Action `grafana/k6-action` | Mac + CI |
| vegeta, hey | `brew install vegeta hey` | Mac |
| schemathesis | `pipx install schemathesis` | Mac + CI |
| govulncheck, golangci-lint, benchstat | `go install golang.org/x/vuln/cmd/govulncheck@latest` … | Mac + CI |
| trivy, dive, dockle | `brew install trivy dive goodwithtech/r/dockle` | Mac + CI |
| squawk | `npm i -g squawk-cli` | Mac + CI |
| psql, pgbench | `brew install libpq` | Mac |
| lynis, goss, ssh-audit, nmap, testssl.sh | `apt install lynis nmap`; goss/testssl từ GitHub release; `pipx install ssh-audit` | VPS (lynis, goss) / laptop (nmap, ssh-audit, testssl) |
| licensee | `gem install licensee` | Mac |
