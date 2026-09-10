# Naming — kiểm tra tên, trademark, domain, handle (P0-02)

> Tên làm việc hiện tại: **Shadow** (tên repo). Chưa xác nhận là tên phát hành.
> "Shadow" là từ phổ thông, rủi ro trùng App Store cao (shadowing là kỹ thuật luyện nghe-nói nổi tiếng → nhiều app tên "Shadowing"). Cần bảng dưới trước khi chốt.

## 1. Bảng ứng viên

Ký hiệu: ✅ trống / ⚠️ có app gần giống / ❌ trùng

| Tên | App Store VN | App Store US | `.com` | `.app` | Instagram | X/Twitter | Trademark N9/N41 VN | Trademark US | Điểm |
|---|---|---|---|---|---|---|---|---|---|
| Shadow | `TODO` | `TODO` | (chắc chắn ❌) | `TODO` | | | | | |
| Shadow English | | | | | | | | | |
| `TODO` ứng viên 3 | | | | | | | | | |
| `TODO` ứng viên 4 | | | | | | | | | |
| `TODO` ứng viên 5 | | | | | | | | | |

Điểm = số ✅. Chọn tên ≥ 7/8 ✅, ưu tiên `.com` hoặc `.app`.

Nguồn check:
- App Store: tìm trong app App Store trên iPhone, đổi region VN/US; và `https://apps.apple.com/us/search?term=`.
- Domain: Cloudflare Registrar.
- Handle: `namecheckr.com`, rồi vào từng mạng xác nhận.
- Trademark: WIPO Global Brand Database, USPTO (tess2.uspto.gov / trademarkcenter), IP Vietnam `wipopublish.ipvietnam.gov.vn` — nhóm **9** (phần mềm) và **41** (giáo dục).

## 2. Tên chốt

| | Giá trị |
|---|---|
| Tên hiển thị App Store (≤ 30 ký tự) | `TODO` |
| Subtitle (≤ 30) | `TODO` |
| Bundle ID | `com.<org>.shadow` → đổi nếu tên đổi |
| Domain chính | `TODO` (ghi giá trị thật **chỉ** ở `docs/ops/accounts.md`; docs khác giữ `<domain>`) |
| Ngày mua domain, auto-renew | `TODO` |
| Handle đã giữ | IG: / X: / TikTok: / YouTube: |
| App record App Store Connect đã tạo giữ tên | Y/N, ngày |

## 3. Quy tắc dùng tên trong code

- Package/module: `shadow` (lowercase) — `@shadow/web`, `@shadow/tokens`, Go module `github.com/JustineUIT/shadow/backend`, SPM `Shadow*`.
- Không hard-code tên hiển thị trong code; đọc từ `contracts/tokens/tokens.json` → `brand.name` (web) và `Info.plist CFBundleDisplayName` (iOS) để đổi tên 1 chỗ.
