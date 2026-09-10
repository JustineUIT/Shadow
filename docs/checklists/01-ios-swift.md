# Checklist 01 — iOS Swift

> Dùng để AI check source `ios/`, sau đó bạn re-check thủ công các mục `[M]`. Mỗi mục có **Target Bare** (gate ship) và **Target Perfect** (làm sau). Bằng chứng = `file:dòng` hoặc output lệnh. Không có bằng chứng = FAIL.
> Tham chiếu: quyết định D11–D16, D20, D27 trong `00-MASTER-PLAN.md`; contract trong `01-CONTRACTS.md`; tokens trong `02-DESIGN-SYSTEM.md`; số đo trong `03-MEASUREMENT-TOOLS.md` §1.

## 0. Biên giới: iOS nhận gì, phải trả gì

| Hướng | MUST | OPT | Nguồn sự thật |
|---|---|---|---|
| **Input** từ contracts | `contracts/openapi.yaml` → sinh `APIClient` (types + client); `tokens.json` → `Tokens.swift`; `events.yaml` → `Events.swift`; `fixtures/fsrs_cases.json` → test parity | `tokens.dark.json` | `01-CONTRACTS.md` §6 |
| **Input** từ backend | Mọi response theo §2 Contracts: Problem Details, cursor, UUIDv7, RFC3339, enum snake_case có `unknown`; `GET /v1/version` để force update | | `01-CONTRACTS.md` §2, §4 |
| **Input** cấu hình | `Debug/Staging/Release.xcconfig` với `API_BASE_URL`, `CDN_BASE_URL`, `REVENUECAT_PUBLIC_KEY`, `SENTRY_DSN`, `POSTHOG_KEY`, `POSTHOG_HOST`, `BUNDLE_ID_SUFFIX` | | `01-CONTRACTS.md` §8 |
| **Output** tới backend | Mọi request có `Authorization`, `X-Client-Platform: ios`, `X-Client-Version: <semver>+<build>`, `X-Timezone` cho `/study/*`; `Idempotency-Key` cho 3 POST; `client_review_id` UUID sinh tại client cho mỗi review | `X-Request-Id` tự sinh để trace | `01-CONTRACTS.md` §2.2 |
| **Output** analytics | Event đúng tên/props trong `events.yaml`, không PII | | §7 Contracts |
| **Output** evidence | `docs/evidence/ios/*` theo `03-MEASUREMENT-TOOLS.md` §1 | | |
| **Output** store | Build TestFlight, privacy labels, review notes, screenshots | | Checklist 06 mục A |

Hợp đồng nội bộ giữa các package (dependency chỉ đi **xuống**, vi phạm = build fail vì SPM không cho vòng):

```
App → Features/* → Core, DesignSystem, APIClient
Features/* ✗ Features/*        (feature không import feature khác; giao tiếp qua Core hoặc App router)
Core → APIClient (chỉ protocol), GRDB, FSRS
DesignSystem → (không import gì của app)
APIClient → OpenAPIRuntime, OpenAPIURLSession
```

## 1. Cấu trúc file kỳ vọng

```
ios/
├── App/
│   ├── App.xcodeproj  (chỉ target App + AppTests UI; mọi logic trong Packages)
│   ├── App/{AppMain.swift, AppRouter.swift, AppDependencies.swift, Info.plist, PrivacyInfo.xcprivacy, App.entitlements}
│   ├── Config/{Base,Debug,Staging,Release}.xcconfig
│   └── Assets.xcassets (AppIcon; màu KHÔNG ở đây, ở Tokens.swift)
├── Packages/
│   ├── APIClient/   Sources/APIClient/{Generated (không commit), Middleware/{AuthMiddleware,RetryMiddleware,HeadersMiddleware,MetricsMiddleware}.swift, APIClientFactory.swift}  openapi-generator-config.yaml  Tests/
│   ├── DesignSystem/ Sources/DesignSystem/{Generated/Tokens.swift, Components/*, Catalog/*}  Tests/
│   ├── Core/        Sources/Core/{Models, Database/{AppDatabase.swift, Migrations.swift, Records/*}, FSRS/{FSRS.swift, Parameters.swift}, Sync/{SyncEngine.swift, PendingReviewQueue.swift, Reachability.swift}, Auth/{SessionStore.swift, Keychain.swift}, Observability/{Sentry.swift, MetricKitSubscriber.swift, Analytics.swift}, Generated/Events.swift}  Tests/
│   └── Features/    Sources/{Onboarding,Auth,Decks,Study,Stats,Paywall,Settings}/{*View.swift, *Model.swift}  Tests/
├── .swiftlint.yml  .swift-format  Package.resolved (commit)
├── ci_scripts/ci_post_clone.sh   (Xcode Cloud: cài swiftlint, chạy gen tokens check)
└── App.xctestplan  Performance.xctestplan
```

## 2. A — Kiến trúc & dự án (A01–A12)

| ID | Mục | Target Bare | Target Perfect | Cách check (bằng chứng) | M |
|---|---|---|---|---|---|
| A01 | Xcode project + SPM local packages theo cấu trúc §1; App target không chứa logic | Mọi `.swift` ngoài `App/App/` nằm trong `Packages/`; App target ≤ 5 file | Package `Features` tách thành nhiều package riêng | `find ios/App/App -name '*.swift' \| wc -l` ≤ 5; `Package.swift` mỗi package có `swiftLanguageVersions: [.v6]` | |
| A02 | Swift 6 strict concurrency + SwiftLint + swift-format + 3 xcconfig | `SWIFT_STRICT_CONCURRENCY = complete`, `SWIFT_VERSION = 6.0`, 0 warning; `.swiftlint.yml` bật `no_hardcoded_color` (custom rule), `force_unwrapping`, `implicitly_unwrapped_optional`, `todo` warn; `Debug/Staging/Release.xcconfig` khác `API_BASE_URL` và `BUNDLE_ID_SUFFIX` | `-warnings-as-errors` bật ở CI | `grep -r "SWIFT_STRICT_CONCURRENCY" ios/App/Config/Base.xcconfig`; `xcodebuild build 2>&1 \| grep -c "warning:"` = 0; `swiftlint lint --strict` exit 0 | |
| A03 | Dependency direction đúng §0 | Không package Feature import Feature khác; DesignSystem không import Core | | `grep -rn "^import" ios/Packages/Features \| grep -E "import (Onboarding\|Auth\|Decks\|Study\|Stats\|Paywall\|Settings)$"` = 0; `grep -rn "^import Core" ios/Packages/DesignSystem` = 0 | |
| A04 | State: `@Observable` model per screen, `@MainActor`; không `ObservableObject` mới; không singleton mutable ngoài `AppDependencies` | Mọi `*Model.swift` là `@MainActor @Observable final class`; dependency inject qua init | Dùng `@Environment` cho dependency container | `grep -rn "ObservableObject" ios/Packages` = 0; `grep -rn "static let shared" ios/Packages` ≤ 1 (Sentry) | |
| A05 | Networking chỉ qua `APIClient` generated + middleware | Không `URLSession.shared.data` ngoài APIClient; middleware chuỗi: Headers → Auth → Retry → Metrics; 401 → refresh 1 lần → retry; refresh có `actor` lock chống 2 refresh song song | Cert pinning (SPKI) cho `api.` | `grep -rn "URLSession" ios/Packages --include=*.swift -l` chỉ ra file trong `APIClient/`; test `AuthMiddlewareTests.test401RefreshesOnceAndRetries` pass; test `testConcurrentRefreshOnlyOneNetworkCall` pass | |
| A06 | Retry policy đúng Contracts §5: 3× exp backoff + jitter **chỉ** GET và POST có `Idempotency-Key`; 429 đọc `RateLimit-Reset`, không retry ngay; 5xx retry, 4xx không | | | `RetryMiddlewareTests`: 3 case (GET 500 → retry 3; POST không key 500 → 0 retry; 429 → chờ reset) | |
| A07 | Decode chịu lỗi: enum có `unknown`; field lạ bỏ qua; date decode RFC3339 ms | Mọi enum từ API có case fallback; test decode dùng `examples` trong openapi | | `grep -rn "case unknown" ios/Packages/Core/Sources/Core/Models \| wc -l` ≥ số enum trong openapi; `DecodeExamplesTests` pass | |
| A08 | Local DB GRDB: schema mirror server cho `decks, cards, card_states, pending_reviews, review_logs_local, dictionary_cache, sync_state`; migrations versioned `v1…`; `DatabaseMigrator.eraseDatabaseOnSchemaChange = false` ở Release | Migration test từ v1 → head với DB seed thật | FTS5 cho search offline | `Migrations.swift` có `migrator.registerMigration("v1")…`; test `MigrationTests.testMigrateFromV1Fixture` pass; `grep eraseDatabaseOnSchemaChange` chỉ trong `#if DEBUG` | |
| A09 | FSRS Swift cùng version server, parity 100% fixtures; preview interval offline dùng FSRS local | `FSRSParityTests` chạy toàn bộ `fsrs_cases.json`, so `stability/difficulty/interval` với tolerance 1e-6; `fsrs_version` từ `/v1/version` so với local, khác major → tắt preview + báo | Optimizer parameters per-user tải từ server | Test pass 100%; `grep -rn "fsrs_version_mismatch"` có xử lý | |
| A10 | Offline-first Study: học hoàn toàn không mạng; review ghi `pending_reviews` (có `client_review_id`, `reviewed_at`, `duration_ms`, `rating`, `state_before`) trong **cùng transaction** với update `card_states` local | Kill app giữa session → mở lại không mất review | | `PendingReviewQueueTests.testWriteIsAtomic`; Airplane mode học 50 thẻ → mở mạng → sync → server `count` = 50 | [M] |
| A11 | SyncEngine: push batch ≤ 100 review với `Idempotency-Key` = hash(batch); xử lý `results[].status`: `applied` → xoá pending + ghi state; `duplicate` → xoá pending; `rejected` → giữ + gắn lỗi, không retry vô hạn (max 5); pull queue sau push; conflict **server-wins**; chạy khi: app foreground, có mạng lại (`NWPathMonitor`), sau session; không chạy song song (`actor`) | Background sync `BGAppRefreshTask` | Delta sync bằng `updated_since` | `SyncEngineTests` 5 case (applied/duplicate/rejected/kill-mid-sync/no-parallel); log `sync_completed` event có `pushed/pulled/result` | |
| A12 | Force update + version gate: launch gọi `GET /v1/version`, `build < min_ios_build` → màn chặn; 426 từ bất kỳ request → cùng màn | Cache version 1 giờ | | `VersionGateTests`; grep `upgrade_required` | |

## 3. S — Bảo mật & dữ liệu (S01–S08)

| ID | Mục | Target Bare | Target Perfect | Cách check | M |
|---|---|---|---|---|---|
| S01 | Token trong Keychain (`kSecAttrAccessibleAfterFirstUnlockThisDeviceOnly`), không UserDefaults, không log | | Biometric gate cho admin | `grep -rn "UserDefaults" ios/Packages \| grep -i token` = 0; `grep -rn "print(" ios/Packages` = 0 (dùng `Logger`); Keychain wrapper có test | |
| S02 | `Logger` (os.log) với privacy: `\(x, privacy: .private)` cho email/token/user id | | | `grep -rn "privacy: .public" ios/Packages \| grep -iE "email\|token"` = 0 | |
| S03 | Sign in with Apple: nonce SHA-256, gửi `raw_nonce` + `identity_token`; xử lý `credentialRevoked` → logout | | | Code `ASAuthorizationAppleIDProvider` có nonce; test | |
| S04 | Delete account: gọi `DELETE /v1/me` với `confirm`, sau 202 → xoá Keychain + xoá file GRDB + reset PostHog id | | | `DeleteAccountTests`; sau delete `ls Application Support` không còn `app.sqlite` | [M] |
| S05 | ATS mặc định (không `NSAllowsArbitraryLoads`); chỉ HTTPS; `API_BASE_URL` Release là `https://api.` | Cert pinning | | `plutil -p Info.plist \| grep -i arbitrary` = 0 | |
| S06 | `PrivacyInfo.xcprivacy` khai báo: tracking = false, collected data types (email, user id, usage data, purchase), required reason API (`UserDefaults CA92.1`, `FileTimestamp C617.1`, `SystemBootTime 35F9.1` nếu dùng) | | | File tồn tại; upload TestFlight 0 email ITMS-91053 | |
| S07 | Không secret trong repo: xcconfig chỉ public key; `.gitignore` có `*.xcuserdata`, `Config/Local.xcconfig` | | | `gitleaks detect --source ios` 0 finding | |
| S08 | Idempotency: `client_review_id` UUID v4/v7 sinh lúc tạo review, không đổi khi retry; `Idempotency-Key` cho tạo deck/card lưu cùng draft đến khi 2xx | | | Test: retry 3 lần cùng key; server nhận 1 | |

## 4. P — Performance (P01–P08) — số từ `03-MEASUREMENT-TOOLS.md` §1

| ID | Mục | Target Bare | Target Perfect | Cách check | M |
|---|---|---|---|---|---|
| P01 | Sentry + MetricKit subscriber bật ở Release; dSYM upload tự động (Xcode Cloud post-build hoặc `sentry-cli`); `tracesSampleRate` ≤ 0.2 | | Sentry Profiling | `MetricKitSubscriber.swift` có `MXMetricManager.shared.add`; Sentry release có dSYM (screenshot); `ci_scripts/ci_post_xcodebuild.sh` upload | |
| P02 | Cold launch ≤ 400 ms (Release, máy thật, median 5) | | ≤ 250 ms | I1/I2 evidence `ios/<date>-perf-metrics.json` | |
| P03 | Không làm việc nặng trong `init`/`body`: DB mở lazy, không network trước first frame, không `Task` trong `App.init` | | | Instruments App Launch: post-main không có span network/DB; grep `init()` trong `AppMain` | |
| P04 | List dùng `LazyVStack`/`List` với `id` ổn định; ảnh/âm thanh cache disk (`URLCache` 50 MB + audio dir); hitch < 5 ms/s | | | I3 evidence | |
| P05 | Memory: peak Study ≤ 150 MB; 0 leak (I4); ảnh downsample bằng `ImageIO` khi > màn hình | | ≤ 100 MB | I1 `XCTMemoryMetric` + I4 | |
| P06 | Kích thước app: download ≤ 30 MB; audio/từ điển không bundle; asset catalog nén; không dynamic framework ngoài Apple | | ≤ 15 MB | I7 evidence | |
| P07 | Performance test plan + `.xcbaseline` commit; CI fail khi regress > 10% | | | `Performance.xctestplan` tồn tại; `.xcbaseline` trong repo; Xcode Cloud log | |
| P08 | Build time: clean ≤ 120 s; 0 expression type-check > 100 ms | | | I10 evidence | |

## 5. X — Accessibility (X01–X07)

| ID | Mục | Target Bare | Target Perfect | Cách check | M |
|---|---|---|---|---|---|
| X01 | Mọi control có `accessibilityLabel`; icon-only button có label tiếng Việt/Anh theo locale | | `accessibilityHint` cho action không rõ | Accessibility Inspector Audit 0 issue "missing label" (I8) | |
| X02 | Dynamic Type: chỉ dùng `Font` từ `Tokens.swift` (map TextStyle), không `.system(size:)`; 5 màn chính không cắt chữ tới AX3 | AX5 | | `grep -rn "\.system(size:" ios/Packages` = 0; ảnh chụp AX3 5 màn | [M] |
| X03 | VoiceOver học được 3 thẻ mắt nhắm: đọc front, nút "Lật", 4 nút rating có label + `accessibilityValue` = interval | | Custom rotor | Quay video 1 phút | [M] |
| X04 | Contrast từ tokens đã đo (DS-05); không hardcode màu | | | DS-03 evidence | |
| X05 | Reduce Motion: flip → crossfade; respect `accessibilityReduceMotion` | | | DS-11 | [M] |
| X06 | Touch target ≥ 44×44 | | | Inspector hit region | |
| X07 | Không dùng màu duy nhất để truyền nghĩa (rating có text + icon) | | | Review 4 nút rating có label text | |

## 6. T — Tests (T01–T06)

| ID | Mục | Target Bare | Target Perfect | Cách check | M |
|---|---|---|---|---|---|
| T01 | Swift Testing cho Core: FSRS parity (A09), PendingReviewQueue, SyncEngine, Migrations, Keychain, cursor/date decode | Coverage `Core` ≥ 70% | ≥ 85% | `xcodebuild test -enableCodeCoverage YES`; `xcrun xccov view --report --json` → `Core` lineCoverage | |
| T02 | APIClient middleware tests với mock transport (401→refresh, retry, headers đủ) | | | Test list trong A05/A06 | |
| T03 | Decode test dùng `examples` từ openapi (script copy examples → `Tests/Fixtures/*.json`) | Mọi schema có example được decode | | `DecodeExamplesTests` số case = số schema | |
| T04 | XCUITest smoke: login OTP (reviewer account) → mở deck → học 3 thẻ → xem stats | Chạy trên Xcode Cloud mỗi PR | Snapshot tests 3 size × 2 theme | Xcode Cloud workflow log xanh | |
| T05 | Performance test plan (P07) | | | | |
| T06 | Test không phụ thuộc mạng thật: mọi test dùng mock transport / GRDB in-memory; `Staging` chỉ dùng trong UI test có tag | | | `grep -rn "api-staging" ios/Packages/*/Tests` = 0 | |

## 7. B — Build, CI, release (B01–B06)

| ID | Mục | Target Bare | Target Perfect | Cách check | M |
|---|---|---|---|---|---|
| B01 | Xcode Cloud: workflow `PR` (build + test + lint qua `ci_post_clone.sh`), workflow `main` (archive → TestFlight internal) | | `release/*` → external | Screenshot 2 workflow; 3 lần chạy gần nhất xanh | |
| B02 | Version: `MARKETING_VERSION` semver, `CURRENT_PROJECT_VERSION` = Xcode Cloud build number; `X-Client-Version` lấy từ `Bundle.main` | | | Grep `CFBundleShortVersionString` được đọc trong `HeadersMiddleware` | |
| B03 | Tokens drift: `ci_post_clone.sh` chạy `style-dictionary build` và `git diff --exit-code Tokens.swift` | | | Script tồn tại; log | |
| B04 | `Package.resolved` commit; dependency ≤ 8 (GRDB, swift-openapi-*, Sentry, RevenueCat, PostHog, FSRS nếu dùng lib) | | | `jq '.pins \| length' Package.resolved` ≤ 8 | |
| B05 | 3 scheme Debug/Staging/Release; Staging bundle id suffix `.staging` cài song song | | | `BUNDLE_ID_SUFFIX` trong Staging.xcconfig | |
| B06 | Crash symbolicated: Sentry issue mẫu có stack frame tên hàm | | | Screenshot | |

## 8. Bare Minimum vs Perfect (tóm tắt)

| Bare (ship TestFlight external) | Perfect |
|---|---|
| A01–A12, S01–S08, P01–P07, X01–X07, T01–T04, T06, B01–B06 | Cert pinning (A05), BGAppRefresh (A11), FTS5 offline dictionary (A08), Snapshot tests (T04), Widget/Live Activity streak, Siri Shortcut, iPad size classes, dark mode, biometric admin |

## 9. G — Prompt cho AI (copy nguyên văn)

```
Bạn là iOS reviewer. Đầu vào: docs/checklists/01-ios-swift.md, docs/01-CONTRACTS.md, contracts/openapi.yaml,
contracts/fixtures/fsrs_cases.json, thư mục ios/ (toàn bộ), và evidence trong docs/evidence/ios/ nếu có.
Kiểm tra từng mục A01→A12, S01→S08, P01→P08, X01→X07, T01→T06, B01→B06.
Với mỗi mục trả về đúng 1 dòng bảng:
ID | PASS/FAIL/N-A | bằng chứng (đường dẫn:file:dòng hoặc lệnh + output rút gọn) | việc cần sửa (nếu FAIL)
Quy tắc:
- Không PASS nếu không có bằng chứng cụ thể. Mục có số đo phải trích số từ file evidence, không tự ước lượng.
- Mục [M] chỉ ghi "CẦN MANUAL", không tự PASS.
- Chạy (hoặc mô tả lệnh bạn đã chạy) các grep trong cột "Cách check" và dán output.
- Cuối cùng liệt kê: mọi URLSession ngoài APIClient, mọi UserDefaults chứa token, mọi .system(size:), mọi hex màu, mọi print(.
- Kết thúc bằng bảng tổng: số PASS / FAIL / MANUAL theo nhóm A,S,P,X,T,B.
```

## 10. Re-check thủ công (bạn làm, ~45 phút, máy thật)

1. **Offline (A10)**: bật Airplane mode, mở app, học 50 thẻ trong 1 deck, kill app giữa chừng 1 lần, học tiếp, tắt Airplane. Chờ sync. Mở admin → user → số review hôm nay phải = 50, không 49, không 51.
2. **Token (S01)**: kết nối Xcode → Devices → Download Container → mở `Library/Preferences/*.plist`, grep `eyJ` (JWT) = không có.
3. **Delete account (S04)**: xoá tài khoản → container không còn `app.sqlite`; đăng nhập lại cùng email → là user mới, deck cũ không còn.
4. **Dynamic Type AX3 (X02)** + **VoiceOver (X03)** + **Reduce Motion (X05)** theo `02-DESIGN-SYSTEM.md` §9 re-check.
5. **Launch (P02)**: tắt app hoàn toàn, mở 5 lần, đếm bằng Instruments App Launch. Không dùng cảm giác.
6. **Force update (A12)**: đổi `min_ios_build` trên staging lớn hơn build hiện tại → mở app phải bị chặn.
7. **IAP (liên quan 06-A03)**: sandbox mua monthly → huỷ → restore trên máy khác cùng Apple ID → entitlement đúng.

## 11. Evidence phải có khi đóng Phase 2

`docs/evidence/ios/`: `<date>-perf-metrics.json`, `-launch.md`, `-hitches.md`, `-leaks.md`, `-size.txt`, `-a11y-audit.md`, `-lint.json`, `-periphery.json`, `-build-timing.txt`, `-sentry.png`, `-metrickit-summary.md` (sau TestFlight 7 ngày), `testflight-round-1.md`, `checklist-01-run-<date>.md` (bảng AI + ghi chú manual).
