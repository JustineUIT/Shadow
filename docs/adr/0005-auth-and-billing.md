# ADR 0005 — Auth tự làm (SIWA + Email OTP, JWT EdDSA, refresh rotation) và thanh toán StoreKit 2 + RevenueCat

- Status: **Accepted** (2026-09-10)
- Bao phủ: D15, D16, D28

## Context
Apple bắt buộc Sign in with Apple nếu có đăng nhập bên thứ ba; app cần ít nhất 1 cách đăng nhập không phụ thuộc Apple (email). Subscription iOS có nhiều edge case (grace period, billing retry, refund, family sharing, upgrade/downgrade). Web admin cần token an toàn trước XSS.

## Options
| Hạng mục | Options | Loại vì |
|---|---|---|
| Auth | Firebase Auth; Clerk; Supabase Auth; **Tự làm trong Go** | Lock-in + chi phí khi scale; tự làm là chỗ đáng học (JWT, rotation, OTP) và đã có checklist bảo mật 03-S |
| Token | Session cookie; JWT access dài; **JWT access 15 phút (EdDSA, `kid`) + refresh opaque rotation + reuse detection** | Access ngắn giới hạn thiệt hại; rotation + family revoke chặn replay |
| Web lưu token | localStorage; **memory + refresh trong httpOnly cookie scope `api.<domain>` path `/v1/auth/refresh`** | localStorage lộ cho XSS |
| Billing | StoreKit 2 tự verify JWS + tự xử lý App Store Server Notifications; **StoreKit 2 + RevenueCat entitlement + webhook về Go** | Tự làm ASSN v2 là bãi mìn edge case; RC free tới 2.5k USD MTR |

## Decision
- Endpoint theo `01-CONTRACTS` §4.2: `/v1/auth/apple`, `/v1/auth/email/otp/{request,verify}`, `/v1/auth/refresh`, `/v1/auth/logout`.
- OTP 6 số, TTL 10 phút, 5 lần sai → khoá 15 phút, luôn trả 202 (không lộ email tồn tại), rate limit per email + per IP.
- JWT: EdDSA (Ed25519), `kid` trong header, JWKS công khai `/.well-known/jwks.json`, rotate key mỗi 90 ngày (2 key song song).
- Refresh: opaque random 32 byte, lưu hash, `family_id`; dùng lại token cũ → revoke cả family.
- iOS: token trong Keychain (`kSecAttrAccessibleAfterFirstUnlockThisDeviceOnly`). Web: access trong memory, refresh cookie `HttpOnly; Secure; SameSite=Strict`.
- Billing: client StoreKit 2 qua RevenueCat SDK; server nhận webhook RC (verify `Authorization` header), ghi `billing_events` (idempotent theo `event_id`), cập nhật `entitlements`; middleware `entitlement_required` đọc DB, không tin client.
- Xoá tài khoản: `DELETE /v1/me` soft delete + revoke; job hard delete sau 30 ngày (Apple yêu cầu có cơ chế xoá trong app).

## Consequences
- (+) Không phụ thuộc nhà cung cấp auth; chi phí 0.
- (−) Trách nhiệm bảo mật thuộc về mình: bắt buộc pass 03-S01–S08 và threat model (P0-10) trước khi mở TestFlight external.
- (−) Không có social login khác (Google) ở MVP — thêm sau nếu PostHog cho thấy drop ở signup.

## Checklist liên quan
03-backend S01–S08, A15. 01-ios S01–S04. 02-web A04, S01–S05. 06-domain A03–A04 (App Review IAP, account deletion).
