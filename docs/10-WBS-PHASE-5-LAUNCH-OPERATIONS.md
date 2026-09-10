# 10 — WBS PHASE 5: Phát hành Chính thức, Vận hành Giám sát & Tăng trưởng

> Phase 5 = "Production Launch & Operational Excellence": Nộp ứng dụng lên Apple App Store, phát hành Landing Page & Admin Portal chính thức, kích hoạt hệ thống Giám sát & Cảnh báo thời gian thực (Grafana Cloud + Sentry + PostHog), thiết lập quy trình trực chiến và theo dõi các chỉ số tăng trưởng sản phẩm (Product KPI).
> Format chuẩn: **Input → Hành động (lệnh/file) → Output → DoD (Kiểm tra tự động & thủ công)**.
> Nguyên tắc: Không phát hành khi chưa có Playbook xử lý sự cố (Runbooks) và kênh cảnh báo tự động về điện thoại.

---

## Bảng phụ thuộc Phase 5

```
P4 Exit Criteria PASS ───────► P5-01 App Store Submission ──► P5-04 Phê duyệt & Live
P1-20 / P3-12 Production Env ─► P5-02 Production Cutover ────► P5-05 Health Checks
P5-03 Monitoring & Alerts ───► P5-06 On-Call Runbooks ─────► P5-07 Growth Tracking
```

---

## Chi tiết từng gói công việc (Work Packages)

### P5-01 & P5-04 — Đóng gói & Nộp Ứng dụng App Store Connect (ASO & Compliance)

| ID | Input | Hành động (Lệnh / File) | Output | DoD (Definition of Done) |
|---|---|---|---|---|
| P5-01.1 | iOS Release Build | Chuẩn bị bộ tài liệu và asset hoàn chỉnh trên App Store Connect: App Name ("Shadow: Học Từ Vựng FSRS"), Subtitle, Từ khóa ASO (Keywords tối ưu 100 ký tự), Mô tả chi tiết (Description), Screenshots 6.7" & 6.1" chuẩn độ phân giải, Video App Preview (tùy chọn), URL Chính sách Quyền riêng tư (`https://<domain>/legal/privacy`), URL Hỗ trợ (`https://<domain>/legal/support`), URL Điều khoản (`https://<domain>/legal/terms`). | Bộ hồ sơ App Store Connect | Tuân thủ 100% App Store Review Guidelines, không vi phạm các từ khóa thương hiệu bị cấm. |
| P5-01.2 | Reviewer Demo Account | Thiết lập tài khoản demo chuyên dụng dành cho chuyên viên duyệt của Apple (Apple App Reviewer): Cung cấp email demo cố định và OTP bypass được định cấu hình an toàn trên backend (`REVIEWER_EMAIL` và `REVIEWER_OTP_FIXED`), kèm ghi chú hướng dẫn chi tiết (Review Notes) cách trải nghiệm toàn bộ tính năng và mua gói thử nghiệm In-App Purchase. | `docs/legal/review-notes.md` | Đăng nhập tài khoản Reviewer hoạt động 100% không bị chặn bởi cơ chế chống brute-force. |
| P5-01.3 | App Privacy Labels | Khai báo danh mục dữ liệu thu thập (Data Collection Labels) trên App Store Connect: Khai báo rõ ràng Contact Info (Email - dùng cho Account/Auth), Usage Data (Product Interaction - dùng cho Analytics không liên kết danh tính), Diagnostics (Crash Data - dùng cho Sentry), không sử dụng Dữ liệu để theo dõi người dùng (Tracking: NO). | Bản khai App Privacy | Khớp 100% với tài liệu kiểm kê dữ liệu trong `docs/legal/data-inventory.md`. |
| P5-04.1 | Nộp xét duyệt & Phê duyệt | Nhấn "Submit for Review". Theo dõi trạng thái duyệt của Apple. Sau khi nhận trạng thái "Approved" -> tiến hành Release theo cơ chế Phân phối theo từng giai đoạn (Phased Release: 1% -> 2% -> 5% -> 10% -> 20% -> 50% -> 100% trong 7 ngày) để kiểm soát rủi ro phát sinh lỗi. | App Store Live Status | Ứng dụng có mặt chính thức trên App Store; tỷ lệ Crash-free sessions $\ge 99.5\%$ trong suốt quá trình rollout. |

---

### P5-02 & P5-05 — Chuyển đổi Môi trường Production & Kiểm tra Hoạt động (Cutover)

| ID | Input | Hành động (Lệnh / File) | Output | DoD (Definition of Done) |
|---|---|---|---|---|
| P5-02.1 | Production DNS & VPS | Cấu hình DNS trên Cloudflare: Kích hoạt subdomain `api.<domain>` trỏ về IP VPS Production (bật Cloudflare Proxy vàng), kích hoạt domain chính `<domain>` và `www` trỏ về Cloudflare Pages Landing Page, kích hoạt `admin.<domain>` trỏ về Cloudflare Pages Admin SPA. | DNS Production Live | Kiểm tra lệnh `dig +short api.<domain>` trả về dãy IP Anycast của Cloudflare, kết nối SSL Full Strict hoạt động 100%. |
| P5-02.2 | Database Migration Prod | Thực hiện chạy `app migrate up` trên cơ sở dữ liệu Neon Production qua kết nối Direct connection string. Kiểm tra tính sẵn sàng của database và connection pool. | Database sẵn sàng | Lệnh kiểm tra trả về trạng thái schema version mới nhất, 0 migration pending. |
| P5-05.1 | Smoke Testing Prod | Chạy kịch bản kiểm tra tự động sau triển khai (Post-Deployment Smoke Tests): Gọi `/readyz` -> Trả về 200; Gọi `GET /v1/version` -> Trả về version hiện tại; Gọi tìm kiếm từ điển -> Trả về kết quả trong $\le 20$ ms. | Báo cáo Smoke Test | 100% các endpoint công khai phản hồi chính xác, 0 cảnh báo lỗi trên logs. |

---

### P5-03 — Thiết lập Hệ thống Giám sát & Cảnh báo Tự động (Observability & Alerting)

| ID | Input | Hành động (Lệnh / File) | Output | DoD (Definition of Done) |
|---|---|---|---|---|
| P5-03.1 | Grafana Alloy & Prometheus | Cài đặt container Grafana Alloy trên VPS thu thập metrics: Cào metrics từ Backend Go (`/metrics` nội bộ tại port 6060), System metrics qua `node_exporter` (CPU, RAM, Disk, Network I/O), Container metrics qua `cadvisor`. Đẩy toàn bộ dữ liệu an toàn về Grafana Cloud. | Grafana Dashboards | Dashboard hiển thị realtime độ trễ p50/p95/p99, tỷ lệ lỗi HTTP 5xx, số lượng kết nối DB pool, tải CPU/RAM máy chủ. |
| P5-03.2 | Realtime Alerting Rules | Thiết lập các luật cảnh báo tự động gửi tin nhắn tức thì qua Telegram / Pushover / Email trong $\le 5$ phút khi xảy ra các điều kiện: 1) Tỷ lệ lỗi HTTP 5xx $> 1\%$ trong 5 phút; 2) Dung lượng ổ đĩa Disk Usage $> 80\%$; 3) RAM máy chủ $> 85\%$ liên tục 10 phút; 4) Database Connection Pool cạn kiệt ($> 90\%$ capacity); 5) Không nhận được heartbeat `/readyz` sau 2 phút. | Cấu hình Alerting | Thử nghiệm kích hoạt cảnh báo giả lập (Alert Test) nhận được thông báo trên điện thoại trong $\le 60$ giây. |

---

### P5-06 — Quy trình Xử lý Sự cố Chuẩn (Operational Runbooks)

| ID | Input | Hành động (Lệnh / File) | Output | DoD (Definition of Done) |
|---|---|---|---|---|
| P5-06.1 | Sự cố Khẩn cấp | Hoàn thiện bộ 10 tài liệu xử lý sự cố trong thư mục `docs/ops/runbooks/`: `deploy.md` (Quy trình triển khai tiêu chuẩn), `rollback.md` (Quy trình khôi phục bản build trước trong 1 phút), `incident.md` (Quy trình ứng phó và phân loại sự cố Sev 1/Sev 2), `db-down.md` (Xử lý khi mất kết nối Database Neon), `disk-full.md` (Xử lý khi tràn ổ cứng VPS), `cert-expired.md` (Xử lý khi lỗi chứng chỉ SSL/TLS), `ddos.md` (Kích hoạt chế độ Under Attack trên Cloudflare), `restore.md` (Quy trình khôi phục database từ bản backup R2), `rotate-secrets.md` (Quy trình thay đổi token bí mật/chìa khóa định kỳ), `vps-rebuild.md` (Quy trình dựng lại toàn bộ VPS từ con số không trong 15 phút). | 10 Runbook files | Mọi thao tác trong Runbook đều có lệnh copy-paste chính xác, không ghi chung chung "kiểm tra lại cấu hình". |

---

### P5-07 — Đo lường Chỉ số Sản phẩm & Tăng trưởng (Product Analytics & Growth)

| ID | Input | Hành động (Lệnh / File) | Output | DoD (Definition of Done) |
|---|---|---|---|---|
| P5-07.1 | PostHog Dashboards | Thiết lập 3 bảng theo dõi chỉ số tăng trưởng cốt lõi trên PostHog: 1) **Activation Funnel**: Mở app -> Hoàn thành Onboarding -> Học xong thẻ đầu tiên -> Hoàn thành bài học ngày 1; 2) **Retention Matrix**: Tỷ lệ người dùng quay lại vào Ngày 1 (D1), Ngày 7 (D7), Ngày 30 (D30); 3) **Core Loop Engagement**: Số lượt reviews trung bình/ngày/user, thời gian học trung bình/ngày. | PostHog Dashboards | Dashboard tự động cập nhật số liệu không phụ thuộc cookies hay theo dõi nhạy cảm. |
| P5-07.2 | Doanh thu & Chuyển đổi | Theo dõi chỉ số In-App Purchase qua RevenueCat: Tỷ lệ chuyển đổi dùng thử (Trial Conversion Rate), Doanh thu định kỳ hàng tháng (MRR), Tỷ lệ hủy gói (Churn Rate), Doanh thu trung bình trên mỗi người dùng trả phí (ARPPU). | RevenueCat Charts | Báo cáo doanh thu đồng bộ theo thời gian thực. |

---

## Tiêu chí hoàn thành Phase 5 (Exit Criteria)

1. [ ] Ứng dụng iOS Shadow chính thức có mặt trên Apple App Store, đạt trạng thái Ready for Sale.
2. [ ] Hệ thống Production (API, Database, Web Landing, Admin Portal, CDN) hoạt động ổn định với thời gian Uptime đạt $\ge 99.5\%$.
3. [ ] Hệ thống Giám sát & Cảnh báo tự động kích hoạt, kênh cảnh báo khẩn cấp gửi thông báo về điện thoại trong $\le 5$ phút khi có lỗi.
4. [ ] Toàn bộ 10 Runbooks xử lý sự cố hoàn chỉnh, đã diễn tập thao tác thành công.
5. [ ] Bộ 3 Dashboards chỉ số sản phẩm (Activation, Retention, Monetization) ghi nhận dữ liệu người dùng thật đầy đủ và chính xác.
