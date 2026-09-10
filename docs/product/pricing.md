# Monetization Strategy & In-App Purchase Specification (RevenueCat + Apple StoreKit 2)

> **Mục tiêu**: Định nghĩa các gói cước, quyền hạn người dùng (Entitlements), bảng giá, và luồng thanh toán In-App Purchase bảo mật cho Shadow.

---

## 1. Cơ cấu Gói Dịch vụ & Quyền hạn (Feature Matrix & Tiers)

| Tính năng | Gói Miễn Phí (Free Tier) | Gói Shadow Pro (Subscription) |
|---|---|---|
| **Số thẻ học mới mỗi ngày** | Giới hạn tối đa 20 thẻ / ngày | Không giới hạn ($\infty$) |
| **Thuật toán FSRS v4.5 cá nhân hóa** | Tham số mặc định toàn hệ thống | Tự động tối ưu tham số trọng số riêng theo lịch sử học cá nhân |
| **Kho từ vựng & System Decks** | Truy cập 3 Decks cơ bản (A1 - A2) | Toàn quyền truy cập tất cả 20+ Decks chuyên sâu (IELTS 7.5+, Oxford 5000, Business English, C1-C2) |
| **Audio phát âm giọng đọc chuẩn Mỹ/Anh** | Giới hạn phát online | Tự động tải trước (Pre-download) toàn bộ Audio để học hoàn toàn Offline |
| **Biểu đồ Thống kê Chuyên sâu** | Xem lịch sử 7 ngày gần nhất | Phân tích chi tiết 365 ngày, ma trận ghi nhớ Retention Heatmap |
| **Đồng bộ hóa đám mây đa thiết bị** | 1 thiết bị | Không giới hạn thiết bị |
| **Quảng cáo & Waitlist** | Có banner giới thiệu Pro | Trải nghiệm 100% không quảng cáo |

---

## 2. Bảng Giá Sản phẩm (Pricing Tiers)

| Mã Sản Phẩm (Product Identifier) | Thời Hạn | Giá Niêm Yết (VND) | Giá Niêm Yết (USD) | Dùng Thử Miễn Phí (Free Trial) | Ghi Chú |
|---|---|---|---|---|---|
| `shadow_pro_monthly` | 1 Tháng | 99,000 ₫ | $4.99 | Không có | Phù hợp người dùng muốn học ngắn hạn |
| `shadow_pro_yearly` | 1 Năm | 699,000 ₫ (Tiết kiệm 41%) | $34.99 | 7 Ngày Dùng thử Miễn phí | Gói cước chiến lược khuyến khích người dùng đăng ký |
| `shadow_pro_lifetime` | Vĩnh viễn | 1,499,000 ₫ | $79.99 | Không có | Gói Early Bird dành cho 500 người dùng đầu tiên |

---

## 3. Kiến trúc Luồng Thanh toán In-App Purchase (RevenueCat Flow)

```
[Người dùng trên iOS App]
        │ 1. Chọn gói Pro & Thanh toán FaceID/TouchID
        ▼
[Apple StoreKit 2 Framework]
        │ 2. Xử lý giao dịch với Apple App Store
        ▼
[RevenueCat iOS SDK]
        │ 3. Nhận CustomerInfo & Cập nhật UI tức thì trên máy
        ▼
[RevenueCat Server Engine]
        │ 4. Gửi Server-to-Server Webhook (POST /v1/billing/webhook)
        ▼
[Shadow Backend Go API]
        │ 5. Ghi nhận `billing_events` & Cập nhật bảng `entitlements` trong Postgres
        ▼
[PostgreSQL Database (Neon)]
```

---

## 4. Xử lý Trạng thái Ngoại lệ & Khôi phục Mua hàng (Restore Purchases)

1. **Nút "Restore Purchases"**: Bắt buộc phải có trong màn hình Paywall và màn hình Cài đặt (Settings) theo quy định của Apple. Khi bấm -> gọi `Purchases.shared.restorePurchases()` -> đồng bộ lại quyền hạn ngay lập tức.
2. **Xử lý Hủy gia hạn (Subscription Cancellation)**: Người dùng hủy qua App Store -> Quyền Pro vẫn có hiệu lực đến hết `expires_at` -> Sau ngày hết hạn, backend tự động chuyển trạng thái người dùng về gói Free.
3. **Grace Period & Billing Retry**: Hỗ trợ thời gian ân hạn 16 ngày của Apple khi thẻ thanh toán của người dùng hết hạn/lỗi, không cắt quyền Pro ngay lập tức nhằm duy trì tỷ lệ giữ chân khách hàng (Retention).
