# 09 — WBS PHASE 4: Tích hợp Toàn diện, Kiểm thử Parity, Tải k6 & Thắt chặt Bảo mật

> Phase 4 = "Integration, Parity & Hardening": Kết nối toàn bộ các thành phần hệ thống (iOS Swift + Backend Go + Web React + Postgres DB + Infra), tiến hành đo đạc hiệu năng thực tế bằng các công cụ chuyên dụng, chứng minh bằng số liệu thật, và kiểm tra toàn diện trước khi Release.
> Format chuẩn: **Input → Hành động (lệnh/file) → Output → DoD (Kiểm tra tự động & thủ công)**.
> Nguyên tắc: Không phát hành bất kỳ tính năng nào nếu số đo thực tế không đạt ngưỡng Target Bare trong `03-MEASUREMENT-TOOLS.md`.

---

## Bảng phụ thuộc Phase 4

```
P1-20 Staging Backend ──┐
P2-13 TestFlight iOS ───┼──► P4-01 E2E Integration ──► P4-03 k6 Load Testing ──► P4-06 Security Audit
P3-12 Cloudflare Pages ─┘    P4-02 FSRS Parity       P4-04 Schemathesis Fuzz    P4-07 Disaster Recovery
                                                     P4-05 Profile & Allocations
```

---

## Chi tiết từng gói công việc (Work Packages)

### P4-01 — End-to-End Integration & Synchronization Verification

| ID | Input | Hành động (Lệnh / File) | Output | DoD (Definition of Done) |
|---|---|---|---|---|
| P4-01.1 | iOS + Backend Staging | Thực hiện kiểm thử kịch bản đồng bộ hoàn chỉnh: Mở app iOS -> Học 20 thẻ Flashcard ở chế độ Máy bay (Offline) -> Bật lại mạng Wi-Fi -> Chờ `SyncEngine` tự động đồng bộ ngầm -> Kiểm tra database Postgres trên server. | Báo cáo kiểm thử đồng bộ | Dữ liệu `card_states` trên server khớp 100% với client; 20 bản ghi mới được ghi nhận vào `review_logs`; không phát sinh bản ghi trùng lặp hoặc mất mát dữ liệu. |
| P4-01.2 | Multi-Device Conflict | Giả lập học song song trên 2 thiết bị khác nhau với cùng 1 tài khoản khi mạng chập chờn. | Báo cáo kiểm thử xung đột | Cơ chế Last-Write-Wins (dựa trên timestamp UTC) xử lý chính xác, trạng thái thẻ cuối cùng hội tụ đồng nhất (eventual consistency). |
| P4-01.3 | Apple IAP & Webhook | Kích hoạt mua gói Subscription thử nghiệm trên iOS qua TestFlight Sandbox -> RevenueCat nhận transaction -> RevenueCat gửi Webhook tới Backend Go `POST /v1/billing/webhook` -> Backend cấp quyền `entitlements` trong Postgres. | Log webhook & database record | Quyền hạn người dùng chuyển sang `active_pro` trong thời gian $\le 3$ giây kể từ khi hoàn tất thanh toán trên iPhone. |

---

### P4-02 — FSRS 4.5 Algorithm Cross-Platform Parity Verification

| ID | Input | Hành động (Lệnh / File) | Output | DoD (Definition of Done) |
|---|---|---|---|---|
| P4-02.1 | `contracts/fixtures/fsrs_cases.json` | Chạy bộ test suite gồm hơn 100 kịch bản học tập phức tạp (nhiều chuỗi rating khác nhau: Again -> Good -> Hard -> Good -> Easy) đồng thời trên 2 nền tảng: Go Backend (`go test ./internal/study -run TestFSRS_Parity`) và Swift iOS Client (`swift test --filter FSRSParityTests`). | Báo cáo so sánh Parity | Kết quả tính toán Stability ($S$), Difficulty ($D$), Scheduled Days ($Interval$) giữa Go và Swift khớp nhau 100% với độ lệch tuyệt đối $\le 10^{-6}$. |
| P4-02.2 | Edge Cases | Kiểm thử các trường hợp đặc biệt: ôn tập thẻ quá hạn 365 ngày (extreme overdue), ôn tập lại ngay sau 10 giây (immediate review), chuỗi 10 lần Again liên tiếp (memory lapse). | Edge case test results | Không xảy ra hiện tượng tràn số (integer/float overflow), không có giá trị NaN hoặc số âm cho Stability/Interval. |

---

### P4-03 — Tải trọng Chuyên biệt với k6 (Load & Stress Testing)

| ID | Input | Hành động (Lệnh / File) | Output | DoD (Definition of Done) |
|---|---|---|---|---|
| P4-03.1 | `loadtest/k6/study-flow.js` | Viết kịch bản kiểm thử tải k6 mô phỏng hành vi 200 người dùng đồng thời (200 Virtual Users) thực hiện vòng lặp học tập liên tục trong 10 phút: Đăng nhập -> Lấy queue học (`GET /v1/study/queue`) -> Nộp kết quả 10 reviews (`POST /v1/study/reviews`) -> Xem biểu đồ thống kê (`GET /v1/study/stats`). | Kịch bản k6 | Lệnh thực thi: `k6 run --vus 200 --duration 10m loadtest/k6/study-flow.js`. |
| P4-03.2 | Measurement Evidence | Chạy k6 trên VPS Staging thật (không chạy trên localhost), thu thập báo cáo metrics chi tiết và lưu vào file `docs/evidence/backend/YYYY-MM-DD-k6-study-flow.json`. | File evidence k6 | Kết quả thực tế phải đạt ngưỡng Target Bare: **Latency p95 $\le 120$ ms, Latency p99 $\le 300$ ms, Tỷ lệ lỗi (Error rate) $< 0.1\%$, Throughput $\ge 500$ RPS**. |
| P4-03.3 | Stress & Spike Test | Chạy k6 với 500 VUs đột ngột trong 1 phút (Spike testing) để kiểm tra cơ chế Rate Limiting và Connection Pool saturation. | Spike test report | Server không bị crash (0 panics), trả về HTTP 429 đúng quy định khi vượt quá quota, tự động hồi phục trạng thái bình thường ngay sau khi hạ tải. |

---

### P4-04 — API Fuzzing & Contract Testing (Schemathesis)

| ID | Input | Hành động (Lệnh / File) | Output | DoD (Definition of Done) |
|---|---|---|---|---|
| P4-04.1 | `contracts/openapi.yaml` & Staging API | Chạy công cụ Schemathesis tự động sinh hàng nghìn payloads bất thường (chuỗi unicode độc hại, số âm cực lớn, body khuyết thiếu, SQL injection payloads) để fuzzing toàn bộ 22 endpoints trên Staging: `schemathesis run contracts/openapi.yaml --base-url https://api-staging.example.com --checks all --validate-schema=true --workers 4`. | Log kiểm thử Schemathesis | Lưu kết quả vào `docs/evidence/backend/YYYY-MM-DD-schemathesis.txt`. |
| P4-04.2 | Schema Compliance | Kiểm tra tính tuân thủ tuyệt đối: Mọi request hợp lệ trả về response khớp 100% schema đã định nghĩa; Mọi request không hợp lệ trả về HTTP 4xx kèm RFC 7807 Problem Details. | Báo cáo kiểm định | **0 failures, 0 server 500 errors** trên toàn bộ 22 endpoints. |

---

### P4-05 — iOS Memory Leaks, Frame Hitches & Allocations Profiling

| ID | Input | Hành động (Lệnh / File) | Output | DoD (Definition of Done) |
|---|---|---|---|---|
| P4-05.1 | Xcode Instruments (Leaks & Allocations) | Chạy Instruments trên iPhone thật ở chế độ Release build: thực hiện kịch bản tự động lặp lại 20 lần chuỗi thao tác (Vào màn hình Study -> Học 10 thẻ -> Thoát ra Home -> Vào lại Study). | File trace `.trace` nén | Lưu số liệu vào `docs/evidence/ios/YYYY-MM-DD-leaks.md`: **0 memory leak**, mức tăng bộ nhớ lưu cữu (abandoned memory growth) $< 1$ MB sau 20 chu kỳ. |
| P4-05.2 | Instruments Animation Hitches | Đo tỷ lệ giật khung hình khi cuộn danh sách 1,000 từ vựng và khi thực hiện hiệu ứng lật thẻ 3D. | File trace Animation Hitches | Lưu số liệu vào `docs/evidence/ios/YYYY-MM-DD-hitches.md`: **Hitch Time Ratio $< 5$ ms/s** (đạt chuẩn "Good" của Apple). |
| P4-05.3 | XCTest Performance Suite | Chạy test plan đo tốc độ Cold Launch trên iPhone thật: `xcodebuild test -scheme App -testPlan Performance -destination 'id=<Device_UDID>'`. | JSON kết quả đo | Lưu vào `docs/evidence/ios/YYYY-MM-DD-perf-metrics.json`: Thời gian khởi động **Cold Launch median $\le 400$ ms**. |

---

### P4-06 — An toàn Thông tin & Hardening Toàn Diện (Security Audit)

| ID | Input | Hành động (Lệnh / File) | Output | DoD (Definition of Done) |
|---|---|---|---|---|
| P4-06.1 | Go Dependency & CVE Scanner | Quét mã nguồn và thư viện phụ thuộc bằng `govulncheck ./...` và quét Docker container image bằng Trivy (`trivy image ghcr.io/org/app:tag`). | Báo cáo quét bảo mật | Lưu vào `docs/evidence/backend/YYYY-MM-DD-trivy.json`: **0 lỗ hổng bảo mật mức HIGH hoặc CRITICAL**. |
| P4-06.2 | Linux Server Hardening (Lynis) | Chạy công cụ kiểm tra bảo mật máy chủ chuyên dụng `lynis audit system` trên VPS production. | Báo cáo kiểm định Lynis | Lưu vào `docs/evidence/server/YYYY-MM-DD-lynis.txt`: **Chỉ số Hardening Index $\ge 75/100$**. |
| P4-06.3 | Port Scan & Exposure Check | Chạy quét cổng mạng từ máy bên ngoài bằng `nmap -sS -p- <VPS_IP>` để xác minh tường lửa UFW và Cloudflare Proxy. | Báo cáo quét cổng nmap | Lưu vào `docs/evidence/server/YYYY-MM-DD-nmap.txt`: **Chỉ mở duy nhất 3 cổng: 22 (SSH key-only), 80 (HTTP redirect), 443 (HTTPS Caddy)**. |
| P4-06.4 | SSL/TLS Configuration Audit | Kiểm tra cấu hình chứng chỉ và mã hóa SSL/TLS qua SSL Labs và testssl.sh: `testssl.sh --jsonfile out.json https://api.example.com`. | Báo cáo chứng chỉ SSL | Đạt điểm **A+ trên SSL Labs**, chỉ hỗ trợ TLS 1.2 và TLS 1.3, vô hiệu hóa hoàn toàn các cipher suites yếu. |

---

### P4-07 — Diễn tập Phục hồi Thảm họa Database (DR & Restore Drill)

| ID | Input | Hành động (Lệnh / File) | Output | DoD (Definition of Done) |
|---|---|---|---|---|
| P4-07.1 | Database Backup Script | Chạy kịch bản sao lưu tự động `infra/scripts/backup.sh`: tạo bản dump mã hóa AES-256 qua GPG và tải lên Cloudflare R2 bucket lưu trữ bảo mật. | File backup mã hóa | Backup hoàn tất trong $\le 2$ phút cho cơ sở dữ liệu dung lượng 1 GB. |
| P4-07.2 | Restore Drill | Thực hiện diễn tập khôi phục dữ liệu thực tế trên một VPS thử nghiệm độc lập: tải bản backup từ R2, giải mã GPG, chạy `pg_restore`, kiểm tra tính toàn vẹn của dữ liệu (Row count của tất cả các bảng và checksum của bảng `review_logs`). | Biên bản diễn tập khôi phục | Lưu vào `docs/evidence/db/YYYY-MM-DD-restore-drill.md`: Thời gian phục hồi thực tế (**RTO**) $\le 30$ phút, thất thoát dữ liệu tối đa (**RPO**) $\le 24$ giờ (hoặc 5 phút nếu dùng Neon PITR). |

---

## Tiêu chí hoàn thành Phase 4 (Exit Criteria)

1. [ ] 100% các ô trong bảng `docs/evidence/SCORECARD.md` thuộc Phase 0 và Phase 1 đã được điền số đo thực tế và đạt trạng thái **PASS**.
2. [ ] Kịch bản tải k6 đạt chuẩn với 200 VUs: Latency p95 $\le 120$ ms, Error rate $< 0.1\%$.
3. [ ] Schemathesis kiểm thử tự động 22 endpoints đạt 0 failure.
4. [ ] Khởi động iOS Cold Launch $\le 400$ ms, Hitch Ratio $< 5$ ms/s, 0 memory leaks.
5. [ ] Đạt điểm A+ trên SSL Labs, Lynis Hardening Index $\ge 75/100$, 0 open ports trái phép.
6. [ ] Diễn tập khôi phục dữ liệu Database thành công với RTO $\le 30$ phút.
