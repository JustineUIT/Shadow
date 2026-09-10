# 08 — WBS PHASE 3: Frontend React (Next.js 16 Static Export: Landing + Legal + Admin SPA)

> Phase 3 = "Web Presence & Admin Portal": Tái cấu trúc Next.js thành static export triển khai trên Cloudflare Pages, tích hợp Design Tokens đồng bộ với iOS, trang Landing Page siêu nhẹ ($\le 90$ KB JS), hệ thống trang Pháp lý (Legal) đầy đủ, và cổng quản trị Admin Portal (SPA bảo mật).
> Format chuẩn: **Input → Hành động (lệnh/file) → Output → DoD (Kiểm tra tự động & thủ công)**.
> Nguyên tắc: Static HTML thuần không cần Node.js runtime khi chạy production, Lighthouse Mobile $\ge 95/100$, 0 lỗi Accessibility (axe-core).

---

## Bảng phụ thuộc Phase 3

```
P0-04 Monorepo Layout ───────► P3-01 Next.js 16 Config ──► P3-04 Landing Page
P0-06 tokens.json ───────────► P3-02 Token CSS Map ──────► P3-05 shadcn UI Adaptor
P0-05 openapi.yaml ──────────► P3-03 openapi-typescript ─► P3-07 Admin API Client
P0-01 PRD & Legal Content ───► P3-06 Legal Pages ────────► P3-09 Sitemap & SEO
P3-07 Admin Client ──────────► P3-08 Admin Features ─────► P3-10 Playwright E2E
P3-11 Cloudflare Pages ──────► P3-12 Security Headers ───► P3-13 Lighthouse CI
```

---

## Chi tiết từng gói công việc (Work Packages)

### P3-01 & P3-02 — Next.js 16 Static Export & Design Token Integration

| ID | Input | Hành động (Lệnh / File) | Output | DoD (Definition of Done) |
|---|---|---|---|---|
| P3-01.1 | `web/next.config.ts` | Cấu hình Next.js 16: `output: 'export'`, `trailingSlash: false`, `reactStrictMode: true`, `images: { unoptimized: true }`. Xóa bỏ hoàn toàn cờ `ignoreBuildErrors: true` để ép TypeScript strict mode. | `next.config.ts` | Chạy `pnpm -F @shadow/web build` tạo thư mục `web/out/` chứa toàn bộ static HTML/CSS/JS. |
| P3-01.2 | `web/package.json` | Cấu hình scripts chuẩn: `lint: next lint`, `typecheck: tsc --noEmit`, `test: vitest run`, `test:e2e: playwright test`, `size: size-limit`. Gỡ bỏ dependency `@vercel/analytics` (thay bằng PostHog tự host/client-side). | `package.json` | Lệnh `pnpm -F @shadow/web typecheck` chạy hoàn tất với 0 lỗi kiểu. |
| P3-02.1 | `contracts/tokens/tokens.json` | Cấu hình Style Dictionary sinh file `web/src/design-system/tokens.css` chứa các biến CSS `--ds-color-*`, `--ds-space-*`, `--ds-radius-*`, `--ds-font-*`. | `tokens.css` | Import `tokens.css` vào `web/app/globals.css`; lint cấm sử dụng hardcode arbitrary color (ví dụ `bg-[#0F766E]`) trong mã React. |
| P3-02.2 | `web/app/globals.css` | Map các class Tailwind v4 / shadcn sang biến semantic `--ds-*`: `bg-background` -> `--ds-color-bg-canvas`, `text-foreground` -> `--ds-color-text-primary`, `border-border` -> `--ds-color-border-subtle`. | `globals.css` | Giao diện tự động đồng bộ màu sắc với iOS Client mà không cần sửa code giao diện. |

---

### P3-03 — Type-Safe API Integration (openapi-typescript + openapi-fetch)

| ID | Input | Hành động (Lệnh / File) | Output | DoD (Definition of Done) |
|---|---|---|---|---|
| P3-03.1 | `contracts/openapi.yaml` | Cấu hình lệnh `openapi-typescript contracts/openapi.yaml -o web/src/api/schema.d.ts` sinh definitions kiểu TypeScript cho toàn bộ endpoints, requests, responses. | `schema.d.ts` | Lệnh `make gen-ts` sinh file tự động; commit file vào git; CI kiểm tra không có drift. |
| P3-03.2 | `openapi-fetch` | Xây dựng API Client trong `web/src/api/client.ts` với `createClient<paths>()`: middleware tự động đính kèm `X-Client-Platform: web` (hoặc `admin`), `X-Client-Version`, quản lý Access Token trong bộ nhớ (in-memory state) và xử lý tự động refresh token qua cookie `httpOnly` tại `api.<domain>/v1/auth/refresh`. | `client.ts`, `auth.ts` | Mọi lệnh gọi API có autocomplete 100% tên route, query params, request body và response types. |
| P3-03.3 | Problem Details Mapper | Viết module `web/src/api/problem.ts` phân giải RFC 7807 `ProblemDetails` thành thông báo toast hiển thị cho người dùng. | `problem.ts` | Gặp lỗi 400/409/429 -> hiển thị toast thông báo tiếng Việt chính xác theo mã lỗi `code`. |

---

### P3-04 & P3-05 — High-Performance Landing Page & shadcn Adaptor

| ID | Input | Hành động (Lệnh / File) | Output | DoD (Definition of Done) |
|---|---|---|---|---|
| P3-04.1 | `docs/product/PRD.md` | Xây dựng trang Landing Page `web/app/(marketing)/page.tsx` gồm các khối component nhẹ: `HeroSection` (giới thiệu phương pháp học Shadowing + FSRS, nút CTA đăng ký Waitlist / Tải app), `FeaturesSection` (3 Core Loops), `HowItWorksSection`, `SocialProofSection`, `FAQSection`, `FooterSection`. | Landing Page Components | Toàn bộ bundle JavaScript của Landing Page $\le 90$ KB (gzipped), 0 framework nặng (không nạp TanStack Table/Form ở trang marketing). |
| P3-04.2 | Waitlist Form | Triển khai form thu thập email đăng ký dùng thử sớm: validate định dạng email bằng Zod, gọi API `POST /v1/waitlist`, hiển thị trạng thái thành công/thất bại kèm hiệu ứng mượt mà. | `WaitlistForm.tsx` | Tốc độ submit form $\le 300$ ms, có cơ chế debounce chống spam bấm liên tục. |
| P3-05.1 | shadcn/ui Base | Tinh chỉnh các component `Button`, `Input`, `Dialog`, `DropdownMenu`, `Card`, `Table`, `Badge` trong `web/components/ui/` để tuân thủ 100% token của Design System. | UI Components | Độ tương phản màu sắc (Contrast Ratio) văn bản đạt chuẩn WCAG AA ($\ge 4.5:1$ cho text thường, $\ge 3:1$ cho text lớn). |

---

### P3-06 — Legal Pages & Compliance (Bắt buộc cho App Store & Luật An toàn dữ liệu)

| ID | Input | Hành động (Lệnh / File) | Output | DoD (Definition of Done) |
|---|---|---|---|---|
| P3-06.1 | `docs/legal/*.md` | Xây dựng các trang tĩnh hiển thị tài liệu pháp lý tại `web/app/legal/`: `/legal/privacy` (Chính sách quyền riêng tư), `/legal/terms` (Điều khoản sử dụng), `/legal/licenses` (Bản quyền phần mềm mã nguồn mở), `/legal/support` (Kênh hỗ trợ người dùng), `/legal/account-deletion` (Hướng dẫn và cổng yêu cầu xóa tài khoản theo yêu cầu của Apple). | 5 Legal Pages | Render trực tiếp từ Markdown tĩnh khi build, tốc độ tải tức thì ($< 200$ ms), có mục lục điều hướng nhanh. |
| P3-06.2 | `.well-known` Config | Tạo các file tĩnh trong `web/public/`: `.well-known/security.txt` (Khai báo kênh báo cáo lỗ hổng bảo mật), `robots.txt` (Cho phép index trang marketing/legal, chặn index `/admin/*`), `sitemap.xml` tự động sinh qua `web/app/sitemap.ts`. | Security & SEO static files | Truy cập `https://<domain>/.well-known/security.txt` trả về HTTP 200 chuẩn MIME type `text/plain`. |

---

### P3-07 & P3-08 — Admin Portal SPA (Quản trị Hệ thống Bảo mật)

| ID | Input | Hành động (Lệnh / File) | Output | DoD (Definition of Done) |
|---|---|---|---|---|
| P3-07.1 | Admin Auth Guard | Xây dựng trang đăng nhập Admin tại `web/app/admin/(auth)/login/page.tsx` và Layout bảo vệ tại `web/app/admin/(app)/layout.tsx`: kiểm tra phiên đăng nhập và role `admin`, tự động chuyển hướng về trang đăng nhập nếu chưa xác thực hoặc phiên hết hạn. | Admin Layout & Guard | Người dùng thường (role `user`) không thể truy cập bất kỳ route nào thuộc `/admin/*`. |
| P3-08.1 | Users Management | Màn hình `web/app/admin/(app)/users/page.tsx`: danh sách người dùng phân trang bằng Cursor, bộ lọc theo ngày tạo/trình độ, xem chi tiết tiến độ học, thao tác khóa/mở khóa tài khoản (có ghi nhận vào `admin_audit_logs`). | Users Management View | Tích hợp TanStack Query và TanStack Table v8, thao tác mượt mà với 10,000 người dùng. |
| P3-08.2 | Dictionary Manager | Màn hình `web/app/admin/(app)/dictionary/page.tsx`: tìm kiếm từ vựng, tra cứu IPA/Audio URL/Ví dụ, chỉnh sửa nghĩa từ điển và duyệt bổ sung từ vựng mới. | Dictionary Editor View | Lưu log kiểm toán chi tiết (diff JSON) của mọi chỉnh sửa dữ liệu từ điển. |
| P3-08.3 | System Metrics & Jobs | Màn hình `web/app/admin/(app)/jobs/page.tsx`: theo dõi trạng thái River background jobs (pending, running, failed), nút kích hoạt chạy lại job lỗi (Retry job). | Jobs Dashboard | Cập nhật realtime trạng thái jobs mỗi 5 giây. |

---

### P3-09, P3-10 & P3-11 — Testing, E2E Automation & Cloudflare Pages Deployment

| ID | Input | Hành động (Lệnh / File) | Output | DoD (Definition of Done) |
|---|---|---|---|---|
| P3-09.1 | Vitest & Testing Library | Viết unit tests cho các component React quan trọng, form validation và custom hooks trong `web/tests/unit/`. | Unit test suite | Coverage đạt $\ge 80\%$ cho các tiện ích logic và form components. |
| P3-10.1 | Playwright E2E | Viết kịch bản kiểm thử End-to-End tự động trong `web/tests/e2e/`: flow đăng ký waitlist, flow duyệt trang legal, flow đăng nhập admin và thao tác trên bảng dữ liệu. Tích hợp `@axe-core/playwright` kiểm tra tự động 100% lỗi accessibility. | E2E & A11y test suites | Toàn bộ kịch bản E2E chạy thành công trên Chromium, Firefox, WebKit; 0 lỗi critical a11y. |
| P3-11.1 | Cloudflare Pages Config | Tạo file `web/public/_headers` thiết lập các security headers nghiêm ngặt: `Content-Security-Policy`, `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`, `Referrer-Policy: strict-origin-when-cross-origin`, `Strict-Transport-Security: max-age=31536000; includeSubDomains`. Tạo file `web/public/_redirects` cấu hình SPA fallback cho `/admin/*`. | `_headers`, `_redirects` | Kiểm tra qua SecurityHeaders.io đạt điểm **A / A+**. |
| P3-12.1 | CI/CD Pages Deploy | Tạo GitHub Actions workflow `.github/workflows/web.yml`: kiểm tra lint, typecheck, test, build static export, chạy Lighthouse CI kiểm tra performance budgets, tự động deploy lên Cloudflare Pages qua Direct Upload. | Workflow file | Deploy hoàn tất trong $\le 90$ giây sau khi merge code. |

---

## Tiêu chí hoàn thành Phase 3 (Exit Criteria)

1. [ ] Build Next.js 16 ở chế độ `output: 'export'` thành công 100%, 0 TypeScript errors/warnings.
2. [ ] Bundle JavaScript First Load của Landing Page $\le 90$ KB (gzipped) kiểm chứng qua `size-limit`.
3. [ ] Điểm số Lighthouse Mobile trên URL triển khai thực tế đạt: **Performance $\ge 95$, Accessibility $\ge 95$, Best Practices = 100, SEO = 100**.
4. [ ] Core Web Vitals đo trên môi trường thử nghiệm đạt: **LCP $\le 2.0$s, CLS $\le 0.05$, TBT $\le 150$ms**.
5. [ ] Quét tự động bằng `axe-core` đạt **0 lỗi** thuộc nhóm Critical và Serious trên tất cả các trang Marketing và Legal.
6. [ ] Đầy đủ 5 trang Pháp lý tĩnh hoạt động chuẩn xác theo yêu cầu xét duyệt App Store của Apple.
7. [ ] Cổng Admin Portal SPA hoạt động trơn tru, bảo vệ quyền truy cập chặt chẽ, ghi log audit đầy đủ.
