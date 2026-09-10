# Security Architecture & STRIDE Threat Model

> **Mục tiêu**: Nhận diện, phân tích và thiết lập cơ chế kiểm soát phòng thủ đa lớp (Defense in Depth) cho toàn bộ hệ thống Shadow dựa trên mô hình **STRIDE** của Microsoft.

---

## 1. Bảng Phân loại Mối đe dọa STRIDE

| Danh mục STRIDE | Mối đe dọa tiềm ẩn | Vùng bị ảnh hưởng | Mức độ rủi ro | Cơ chế Kiểm soát Phòng thủ Triển khai |
|---|---|---|---|---|
| **S**poofing (Giả mạo danh tính) | - Giả mạo Identity Token của Apple Sign In.<br>- Giả mạo người dùng khác để nộp kết quả học.<br>- Tái sử dụng Refresh Token bị đánh cắp. | Auth Service, API Gateway | **HIGH** | - Xác thực chữ ký số Apple JWKS với caching 24h.<br>- Token Ed25519 gắn UUID người dùng, xác thực ở Auth Middleware.<br>- **Refresh Token Rotation (RTR)** với Token Family (`sid`): tự động hủy toàn bộ phiên nếu phát hiện tái sử dụng. |
| **T**ampering (Xâm phạm toàn vẹn dữ liệu) | - Thay đổi kết quả review trên đường truyền mạng.<br>- Sửa đổi trực tiếp dữ liệu nhật ký học `review_logs` hoặc `admin_audit_logs`.<br>- Làm giả webhook thanh toán từ RevenueCat. | Network, Database, Webhook Handler | **HIGH** | - Ép buộc HTTPS/TLS 1.3 qua Cloudflare HSTS.<br>- Áp dụng quyền `REVOKE UPDATE, DELETE` trên các bảng append-only cho user `app_rw`.<br>- Xác thực chữ ký Bearer Authorization Secret trên Webhook của RevenueCat. |
| **R**epudiation (Chối bỏ trách nhiệm) | - Admin từ chối việc đã xóa người dùng hoặc sửa dữ liệu từ điển.<br>- Người dùng khiếu nại về trạng thái thẻ không khớp. | Admin Portal, Study Engine | **MEDIUM** | - Toàn bộ thao tác quản trị được ghi log bất biến vào `admin_audit_logs` kèm `actor_id`, `ip_address`, `diff jsonb`, `request_id`.<br>- Nhật ký `review_logs` lưu trữ `client_review_id` và timestamp do client sinh. |
| **I**nformation Disclosure (Rò rỉ thông tin) | - Lộ cơ sở dữ liệu hoặc thông tin thẻ tín dụng.<br>- Rò rỉ thông tin nhạy cảm qua lỗi hệ thống (Stack traces).<br>- Quét trộm toàn bộ dữ liệu từ điển (Data scraping). | Database, Public API, Error Handler | **HIGH** | - Không bao giờ lưu trữ thẻ tín dụng (Ủy quyền hoàn toàn cho Apple IAP / RevenueCat).<br>- Chuẩn hóa lỗi theo RFC 7807 `Problem Details`, giấu sạch internal stack trace trong môi trường production.<br>- Rate Limiting thông minh (Token Bucket theo IP + User ID), Cloudflare Bot Fight Mode chống bot cào dữ liệu. |
| **D**enial of Service (Từ chối dịch vụ) | - Tấn công DDoS hạ tầng VPS.<br>- Tấn công làm cạn kiệt Connection Pool Postgres qua truy vấn nặng.<br>- Spam liên tục request gửi OTP Email làm cạn quota Resend. | Infrastructure, VPS, Resend API | **CRITICAL** | - Cloudflare Anycast DDoS Protection + WAF Managed Rules.<br>- Thiết lập Connection Pool giới hạn (`max_conns=25`), `statement_timeout = 3000ms` cho mọi truy vấn.<br>- Giới hạn OTP: Tối đa 3 OTP/15 phút/email, chống brute-force 5 lần nhập sai. |
| **E**levation of Privilege (Leo thang đặc quyền) | - Người dùng thông thường tự cấp quyền `admin` hoặc gói trả phí `pro`.<br>- Đọc trộm file cấu hình server qua lỗ hổng Local File Inclusion (LFI). | API Authorization, Docker Container | **CRITICAL** | - Quyền hạn (`role`, `entitlements`) được kiểm tra nghiêm ngặt tại Service Layer dựa trên token JWT và Database state, không tin cậy client state.<br>- Docker Container chạy dưới dạng **Distroless Non-Root** (UID 65532), hệ thống file root là read-only (`read_only: true`). |

---

## 2. Chiến lược Bảo mật 6 Lớp (Defense-in-Depth)

```
[1. Cloudflare Edge] ──► DDoS Protection, WAF Rules, HSTS, Bot Management
        │
[2. Host VPS Level] ──► UFW Firewall (Chỉ mở 22, 80, 443), Fail2ban, SSH Key-only
        │
[3. Caddy Reverse Proxy] ─► TLS 1.3, Rate Limiting, Request Header Filtering
        │
[4. Container Isolation] ─► Distroless Base, Non-Root User, Read-Only Root Filesystem
        │
[5. Go Application] ────► RFC 7807 Errors, Zero Leaks, Strict Context Auth, Idempotency
        │
[6. PostgreSQL Database] ─► Least Privilege Roles (app_rw, app_migrate), Revoke DDL/Deletes
```

---

## 3. Quản lý Khóa Bí mật & Quy trình Xoay vòng (Secrets Management & Rotation)

1. **Nguyên tắc Không Commit Secret**: Không bao giờ đưa `.env`, chứng chỉ SSL, private keys vào Git. Mọi file cấu hình có chứa secret phải nằm trong `.gitignore`.
2. **Môi trường Production**: File `/opt/app/.env` trên VPS được bảo vệ bằng quyền truy cập nghiêm ngặt (`chmod 600`, chỉ thuộc sở hữu của user chạy dịch vụ).
3. **Quy trình Xoay vòng Khóa Bí mật (Rotation Cadence)**:
   - **JWT Signing Key**: Xoay vòng 90 ngày/lần. Hệ thống hỗ trợ 2 keys song song (Key rollover) trong 24 giờ để tránh ngắt quãng phiên đăng nhập của người dùng.
   - **Database Password**: Đổi định kỳ 180 ngày hoặc ngay lập tức khi nghi ngờ có rò rỉ.
   - **Third-party API Keys (Resend, RevenueCat, Sentry)**: Tạo API Key mới trên Dashboard -> Cập nhật `/opt/app/.env` -> Khởi động lại service -> Xóa API Key cũ sau 1 giờ.
