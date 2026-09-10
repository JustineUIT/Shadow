# ADR 0004 — iOS: SwiftUI iOS 17+, Swift 6 strict, MV + SPM packages, GRDB, Xcode Cloud

- Status: **Accepted** (2026-09-10)
- Bao phủ: D11, D12, D13, D27

## Context
Mục tiêu năng lực là Swift thật (không RN). App phải học **offline hoàn toàn** (máy bay, mạng yếu) và sync idempotent. Solo dev → build time và test được là ưu tiên.

## Options
| Hạng mục | Options | Loại vì |
|---|---|---|
| UI | UIKit; **SwiftUI (iOS 17+)** | iOS 17 mở khoá `@Observable`, `NavigationStack` ổn; bỏ iOS 16 mất < 5% thiết bị mục tiêu |
| Kiến trúc | MVVM cổ điển; TCA; VIPER; **MV + ViewModel nhẹ `@Observable` + SPM package theo feature** | TCA: learning curve + boilerplate lớn cho solo; VIPER: quá nhiều file. Package theo feature ép dependency direction và build incremental |
| Local DB | Core Data; SwiftData; **GRDB (SQLite)** | SwiftData chưa ổn với migration + sync phức tạp, khó test; GRDB: schema/migration tường minh, `ValueObservation` cho SwiftUI, FTS nếu cần |
| Networking | Alamofire; tự viết; **swift-openapi-generator + URLSession** | ADR 0002 |
| CI | GitHub Actions macOS; **Xcode Cloud** | GitHub macOS tính phút ×10; Xcode Cloud 25 giờ/tháng free, TestFlight tích hợp |

## Decision
- Target iOS 17.0, Swift 6 language mode, strict concurrency `complete`, 0 warning được phép.
- Packages: `APIClient`, `DesignSystem`, `Core` (Models, Database/GRDB, FSRS, Sync, Keychain, Observability), `Features/*`. App target chỉ compose.
- Dependency direction: `Features → Core, DesignSystem, APIClient`; `Core → APIClient`; `DesignSystem` không import gì nội bộ.
- Server là source of truth; GRDB là cache + hàng đợi `pending_reviews`; conflict server-wins.
- FSRS Swift port cùng version với `go-fsrs`, verify bằng `fsrs_cases.json`.

## Consequences
- (+) Test từng package bằng `swift test` không cần simulator (trừ UI).
- (+) Preview catalog DesignSystem compile độc lập.
- (−) Swift 6 strict làm chậm lúc đầu (Sendable, actor isolation); đây là chi phí học có chủ đích.
- (−) Không có Android; chỉ xem xét sau khi có doanh thu (00 §5 Phase 5).

## Checklist liên quan
01-ios-swift: A01–A12, S01–S08, T01–T06, B01–B06.
