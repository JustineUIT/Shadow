# 07 — WBS PHASE 2: Mobile iOS Swift Client (Native SwiftUI + Offline-First)

> Phase 2 = "Native Mobile Experience": Xây dựng ứng dụng iOS Native hoàn chỉnh bằng Swift 6, SwiftUI, GRDB SQLite (Offline-First), FSRS v4.5 engine local, kiến trúc Modular Local SPM Packages, tích hợp Token Design System tự động.
> Format chuẩn: **Input → Hành động (lệnh/file) → Output → DoD (Kiểm tra tự động & thủ công)**.
> Nguyên tắc: UI mượt mà 60/120 FPS, 0 memory leaks, hoạt động offline 100% cho core study loop.

---

## Bảng phụ thuộc Phase 2

```
P0-05 openapi.yaml ──────────► P2-01 APIClient Package ──► P2-06 Sync Engine
P0-06 tokens.json ───────────► P2-02 DesignSystem Pkg ──► P2-07 Feature UI Screens
P0-11 fsrs_cases.json ───────► P2-03 Core/FSRS Package ──► P2-05 Local Study Loop
P1-01 DB Schema ─────────────► P2-04 GRDB Local Store ──► P2-06 Sync Engine
P2-08 Audio Cache Engine ────► P2-09 Native Player ─────► P2-10 Pronunciation Flow
P2-11 Observability Setup ───► P2-12 MetricKit & Sentry ─► P2-13 TestFlight Build
```

---

## Chi tiết từng gói công việc (Work Packages)

### P2-01 — APIClient Package (swift-openapi-generator)

| ID | Input | Hành động (Lệnh / File) | Output | DoD (Definition of Done) |
|---|---|---|---|---|
| P2-01.1 | `contracts/openapi.yaml` | Thiết lập local SPM package `ios/Packages/APIClient`: cấu hình `Package.swift` với plugin `swift-openapi-generator`, transport `OpenAPIURLSession`. | `Package.swift`, `openapi-generator-config.yaml` | Lệnh `swift build` tự động sinh mã API Client type-safe trong build artifact. |
| P2-01.2 | `01-CONTRACTS.md` | Viết các Client Middlewares trong `ios/Packages/APIClient/Sources/APIClient/Middleware/`: `AuthMiddleware` (tự động gắn `Authorization: Bearer <token>`), `HeadersMiddleware` (`X-Client-Platform: ios`, `X-Client-Version`, `X-Timezone`), `RetryMiddleware` (exponential backoff khi gặp lỗi mạng tạm thời hoặc HTTP 503). | 3 Middleware files | Unit tests giả lập request/response kiểm tra header được chèn chính xác 100%. |
| P2-01.3 | Error Handling | Triển khai mapping RFC 7807 Problem Details response thành enum `APIError` có kiểu rõ ràng trong Swift, xử lý an toàn case lỗi chưa biết (`unknown(ProblemDetails)`). | `ProblemDetailsMapper.swift` | Client giải mã chính xác mọi mã lỗi HTTP 4xx/5xx thành UI message thân thiện. |

---

### P2-02 — DesignSystem Package (SwiftUI Tokens & Reusable Components)

| ID | Input | Hành động (Lệnh / File) | Output | DoD (Definition of Done) |
|---|---|---|---|---|
| P2-02.1 | `contracts/tokens/tokens.json` | Cấu hình Style Dictionary v4 sinh file `ios/Packages/DesignSystem/Sources/DesignSystem/Generated/Tokens.swift` chứa toàn bộ color palette (24 semantic colors), spacing scale (4pt base), corner radii, font scales. | `Tokens.swift` | Lệnh `make gen-tokens` sinh file không lỗi; lint kiểm tra không có hardcode hex `#` hoặc số pt trong Feature views. |
| P2-02.2 | `02-DESIGN-SYSTEM.md` | Xây dựng 11 component SwiftUI cơ bản trong `ios/Packages/DesignSystem/Sources/DesignSystem/Components/`: `AppButton` (Primary, Secondary, Ghost, Destructive), `CardContainer`, `RatingButton` (4 mức: Again, Hard, Good, Easy), `Typography` view modifiers, `SearchBar`, `StreakBadge`, `ProgressBar`, `EmptyStateView`, `SkeletonLoader`, `ToastView`, `TagChip`. | Component source files | Mọi component hỗ trợ chuẩn Accessibility (VoiceOver accessibilityLabel/hint), Dynamic Type scale từ xSmall tới AX3. |
| P2-02.3 | Component Catalog | Viết `CatalogView` hiển thị toàn bộ components và trạng thái (enabled, disabled, loading, focused) phục vụ kiểm tra trực quan trên SwiftUI Previews. | `CatalogView.swift` | Previews load nhanh dưới 2s, không crash khi chuyển đổi Dynamic Type size. |

---

### P2-03 & P2-04 — Core Package: FSRS Engine & GRDB Local Database

| ID | Input | Hành động (Lệnh / File) | Output | DoD (Definition of Done) |
|---|---|---|---|---|
| P2-03.1 | FSRS v4.5 Spec | Triển khai thuật toán FSRS trong `ios/Packages/Core/Sources/Core/FSRS/FSRS.swift`: tính toán Stability, Difficulty, Next Review Date hoàn toàn trên client bằng Swift thuần (zero external dependency). | `FSRS.swift`, `FSRSParameters.swift` | Chạy test suite `FSRSParityTests` đọc `contracts/fixtures/fsrs_cases.json` đạt 100% pass với sai số $\le 10^{-6}$ so với Go backend. |
| P2-04.1 | GRDB.swift | Xây dựng `AppDatabase.swift` quản lý SQLite database cục bộ với `DatabasePool` (bật WAL mode, memory limit 50MB): định nghĩa schema migration cục bộ cho các bảng `local_decks`, `local_cards`, `local_card_states`, `pending_reviews`, `cached_dictionary`. | `AppDatabase.swift`, `DatabaseMigrations.swift` | Tốc độ đọc 10,000 bản ghi từ SQLite cục bộ $\le 15$ ms trên iPhone thật. |
| P2-04.2 | Invariant Protection | Thiết lập unique constraint trên `pending_reviews.client_review_id`, cascading delete hợp lý cho cards khi deck bị xóa. | GRDB Record Models | Kiểm tra transaction an toàn: rollback hoàn toàn nếu ghi review bị ngắt giữa chừng. |

---

### P2-05 & P2-06 — Core Study Loop & Offline-First Sync Engine

| ID | Input | Hành động (Lệnh / File) | Output | DoD (Definition of Done) |
|---|---|---|---|---|
| P2-05.1 | FSRS + GRDB | Xây dựng `StudyEngine` trong `Packages/Core`: tính toán hàng đợi học tập cục bộ (Local Study Queue) ngay cả khi thiết bị ở chế độ Máy bay (Airplane Mode). Thuật toán chọn thẻ Due $\le$ hiện tại + Thẻ mới theo giới hạn cài đặt. | `StudyEngine.swift` | Người dùng có thể học liên tục 100 thẻ offline không cần mạng, không giật lag. |
| P2-05.2 | Review Persistence | Khi người dùng nhấn Rating (Again/Hard/Good/Easy): cập nhật ngay lập tức `local_card_states` trong SQLite, đồng thời chèn bản ghi vào hàng đợi `pending_reviews` (chứa `client_review_id: UUID`, `rating`, `duration_ms`, `reviewed_at`). | `ReviewStore.swift` | UI cập nhật trạng thái thẻ tiếp theo trong thời gian $\le 16$ ms (1 frame 60fps). |
| P2-06.1 | SyncEngine | Xây dựng `SyncEngine` theo cơ chế Event-driven & Background Sync: lắng nghe trạng thái kết nối mạng qua `NWPathMonitor`. Khi có mạng -> tự động đóng gói tối đa 100 pending reviews nộp lên `POST /v1/study/reviews`. | `SyncEngine.swift`, `PendingReviewQueue.swift` | Server phản hồi thành công -> xóa các pending reviews đã nộp. Xử lý xung đột (conflict resolution) theo nguyên tắc Last-Write-Wins dựa trên timestamp UTC. |

---

### P2-07 — SwiftUI Feature Modules (Clean MV Pattern)

| ID | Input | Hành động (Lệnh / File) | Output | DoD (Definition of Done) |
|---|---|---|---|---|
| P2-07.1 | `OnboardingFeature` | Màn hình giới thiệu giá trị, khảo sát trình độ CEFR mục tiêu (A1..C2), chọn giờ học hàng ngày, yêu cầu quyền nhận thông báo cục bộ (Local Notifications). | `OnboardingView.swift`, `OnboardingModel.swift` | Flow mượt mà, lưu trạng thái hoàn thành onboarding vào Keychain/UserDefaults. |
| P2-07.2 | `AuthFeature` | Màn hình đăng nhập hỗ trợ Sign in with Apple native (`ASAuthorizationAppleIDButton`) và OTP qua Email. Lưu JWT Token và Refresh Token an toàn vào iOS Keychain với cờ `kSecAttrAccessibleAfterFirstUnlock`. | `AuthView.swift`, `AuthModel.swift`, `KeychainStore.swift` | Đăng nhập Apple 1 chạm hoàn tất trong $\le 1.5$ giây. |
| P2-07.3 | `StudyFeature` | Màn hình Flashcard lật 3D (3D flip rotation effect), hiển thị từ vựng, IPA, audio player, nghĩa tiếng Việt, câu ví dụ thực tế; hàng nút 4 mức đánh giá (Again, Hard, Good, Easy) với khoảng cách bấm rộng $\ge 44$ pt. | `StudyView.swift`, `CardView.swift`, `StudyModel.swift` | Tỷ lệ Animation Hitch $< 5$ ms/s khi lật thẻ liên tục. |
| P2-07.4 | `DecksFeature` | Danh sách bộ thẻ (System Decks & User Decks), tìm kiếm từ vựng tích hợp từ điển, tạo deck mới, thêm từ vựng từ tra cứu vào deck cá nhân. | `DecksView.swift`, `DictionarySearchView.swift` | Cuộn danh sách 1,000 từ vựng với tốc độ 60/120 FPS không giật lag. |
| P2-07.5 | `StatsFeature` | Biểu đồ trực quan hóa tiến độ học tập: Streak hiện tại, số từ đã thuộc (Mastered), biểu đồ 30 ngày qua Swift Charts native. | `StatsView.swift`, `StatsModel.swift` | Render biểu đồ tức thì $\le 50$ ms từ dữ liệu SQLite cục bộ. |
| P2-07.6 | `SettingsFeature` | Cài đặt tài khoản, tùy chỉnh mục tiêu từ/ngày, giờ thông báo ôn tập, dark mode toggle, thông tin bản quyền (Open Source Licenses), chính sách bảo mật (Privacy Policy), nút yêu cầu Xóa tài khoản (Account Deletion theo quy định Apple App Store). | `SettingsView.swift`, `SettingsModel.swift` | Tuân thủ 100% App Store Review Guidelines mục 5.1.1. |

---

### P2-08, P2-09 & P2-10 — Native Audio Engine & Pronunciation Caching

| ID | Input | Hành động (Lệnh / File) | Output | DoD (Definition of Done) |
|---|---|---|---|---|
| P2-08.1 | AVFoundation & Caching | Xây dựng `AudioPlayerService`: tải file phát âm audio từ CDN (`cdn.<domain>/audio/*.mp3`), lưu cache cục bộ vào `CachesDirectory` với dung lượng tối đa 200 MB (tự động xóa LRU khi đầy). | `AudioPlayerService.swift`, `AudioDiskCache.swift` | Âm thanh đã tải cache phát ngay lập tức ($\le 50$ ms) khi lật mặt thẻ. |
| P2-09.1 | AVAudioSession | Cấu hình `AVAudioSession.sharedInstance().setCategory(.playback, mode: .spokenAudio, options: [.duckOthers])`: tôn trọng công tắc im lặng của iPhone (Silent Switch) hoặc tự động ducking audio khi cần. | Cấu hình Audio Session | Không chiếm quyền phát nhạc nền của các app khác (Spotify/Apple Music) khi không có audio học. |

---

### P2-11, P2-12 & P2-13 — Observability, MetricKit, Build & TestFlight

| ID | Input | Hành động (Lệnh / File) | Output | DoD (Definition of Done) |
|---|---|---|---|---|
| P2-11.1 | MetricKit Subscriber | Triển khai `MetricKitSubscriber` kế thừa `MXMetricManagerSubscriber`: thu thập báo cáo hiệu năng thực tế từ thiết bị người dùng (Cold Launch time, Memory peak, Hang rate, Battery drain, CPU time). | `MetricKitSubscriber.swift` | Định kỳ nhận và gửi payload MetricKit lên hệ thống giám sát mà không tốn pin người dùng. |
| P2-12.1 | Sentry SDK | Tích hợp Sentry Cocoa SDK: ghi nhận crash logs tự động kèm breadcrumbs (hành động trước crash), cấu hình `tracesSampleRate: 0.1` để theo dõi hiệu năng mà không quá tải mạng. | `SentryConfig.swift` | Tự động upload file dSYM lên Sentry khi CI build Release. |
| P2-13.1 | Xcode Cloud / CI Build | Thiết lập kịch bản build tự động `ci_scripts/ci_post_clone.sh`: kiểm tra lint bằng SwiftLint strict mode, chạy toàn bộ Unit Tests và Performance Tests, export file `.ipa` Release build và phân phối tự động lên TestFlight. | CI Scripts, Xcode Cloud Config | Build clean hoàn tất trong $\le 10$ phút; kích thước App Download trên App Store $\le 30$ MB. |

---

## Tiêu chí hoàn thành Phase 2 (Exit Criteria)

1. [ ] Ứng dụng chạy mượt mà trên iPhone thật (iOS 17+), thời gian Cold Launch Release build $\le 400$ ms.
2. [ ] Hoạt động Offline 100% cho Core Study Loop: học thẻ, lật thẻ, lưu kết quả, tự động đồng bộ khi có kết nối mạng trở lại mà không mất dữ liệu.
3. [ ] Memory leak = 0 sau 20 vòng lặp mở và hoàn thành phiên học Study; Peak RAM $\le 150$ MB.
4. [ ] Tỷ lệ giật khung hình (Hitch Time Ratio) khi cuộn danh sách 1,000 thẻ $< 5$ ms/s.
5. [ ] Bộ test FSRS Parity trên Swift đạt 100% pass khớp kết quả với Go Backend.
6. [ ] Kiểm tra Accessibility Inspector đạt 0 lỗi loại Error; hỗ trợ Dynamic Type mượt mà không vỡ layout.
7. [ ] Build TestFlight hoàn thành và phân phối nội bộ thành công cho nhóm kiểm thử.
