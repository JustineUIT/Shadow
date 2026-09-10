# Checklist 02 — Web React (Next.js 16 static export: Landing + Legal + Admin)

> Dùng để AI check source `web/`, sau đó bạn re-check `[M]`. Tham chiếu D17–D19, D28; `01-CONTRACTS.md` §2, §4.8, §8; `02-DESIGN-SYSTEM.md` §6.2; số đo `03-MEASUREMENT-TOOLS.md` §2.

## 0. Biên giới: Web nhận gì, phải trả gì

| Hướng | MUST | OPT | Nguồn |
|---|---|---|---|
| **Input** từ contracts | `openapi.yaml` → `src/api/schema.d.ts` (openapi-typescript, commit); `tokens.json` → `src/design-system/tokens.css`; `events.yaml` → `src/lib/events.ts` | | Contracts §6 |
| **Input** từ backend | Problem Details `code` → bảng toast; cookie `rt` do server set trên `api.<domain>` path `/v1/auth/refresh`; CORS allow `https://<domain>`, `https://admin.<domain>`, preview origin | | Contracts §2.3, §2.6 |
| **Input** env | Chỉ `NEXT_PUBLIC_*` (§8 Contracts), validate bằng zod trong `src/lib/env.ts` lúc build; thiếu → build fail | | |
| **Input** content | Copy landing từ `docs/product/PRD.md`; legal từ `docs/legal/*` (markdown → page) | | |
| **Output** tới backend | Header `X-Client-Platform: web` (landing waitlist) hoặc `admin`; `X-Client-Version: <YYYY.MM.DD>+<git sha 7>`; `Authorization: Bearer` từ memory; refresh gọi với `credentials: 'include'` | | Contracts §2.2 |
| **Output** tới người dùng | Landing, `/legal/{privacy,terms,licenses,support,account-deletion}`, `/admin/*`; `sitemap.xml`, `robots.txt`, `.well-known/apple-app-site-association` (universal links, OPT), `_headers`, `_redirects` | `/design` catalog (non-prod) | Checklist 06 L01–L05 |
| **Output** evidence | `docs/evidence/web/*` (§2 Measurement) | | |

Quy tắc chia mã: `app/(marketing)` và `app/legal` **không import** gì từ `src/features/admin` và không import TanStack (giữ landing ≤ 90 KB JS). ESLint `no-restricted-imports` ép điều này.

## 1. Cấu trúc file kỳ vọng

```
web/
├── app/
│   ├── layout.tsx  (html lang theo locale, fonts Inter latin+vietnamese, theme.css)
│   ├── (marketing)/{page.tsx, pricing/, faq/}  components/{Hero,Features,Pricing,FAQ,DownloadCTA}.tsx
│   ├── legal/{privacy,terms,licenses,support,account-deletion}/page.tsx  (từ content/legal/*.md)
│   ├── admin/
│   │   ├── (auth)/login/page.tsx
│   │   └── (app)/{layout.tsx (guard), users/, decks/, dictionary/, jobs/, metrics/, audit-logs/}
│   ├── design/page.tsx  (chỉ khi NEXT_PUBLIC_ENV !== 'prod'; noindex)
│   ├── sitemap.ts  robots.ts  not-found.tsx
├── src/
│   ├── api/{schema.d.ts (generated), client.ts (openapi-fetch + middleware), problem.ts, auth.ts (memory token + refresh)}
│   ├── design-system/{tokens.css (generated), theme.css (shadcn map), components/*}
│   ├── features/admin/{users,decks,dictionary,jobs,metrics,audit}/{*.tsx, queries.ts}
│   ├── lib/{env.ts, events.ts, analytics.ts (PostHog cookieless), vitals.ts, i18n/}
│   └── content/legal/*.md
├── public/{_headers, _redirects, .well-known/, og.png, favicon...}
├── tests/{unit/*.test.tsx, e2e/*.spec.ts, a11y.spec.ts}
├── next.config.ts (output:'export', images unoptimized, trailingSlash)
├── lighthouserc.json  budget.json  .size-limit.json  playwright.config.ts  vitest.config.ts
├── eslint.config.mjs  stylelint.config.mjs  tsconfig.json (strict)
└── .env.example
```

## 2. A — Kiến trúc & code (A01–A12)

| ID | Mục | Target Bare | Target Perfect | Cách check | M |
|---|---|---|---|---|---|
| A01 | Next.js 16 `output: 'export'`, không API route, không server action, không `next/image` optimization (dùng `unoptimized` + ảnh pre-sized) | `next build` 0 warning; thư mục `out/` deploy lên Pages | | `grep "output: 'export'" next.config.ts`; `find app -name 'route.ts'` = 0; build log | |
| A02 | TS `strict: true`, `noUncheckedIndexedAccess`, `exactOptionalPropertyTypes`; `tsc --noEmit` 0 error; không `any` (eslint `no-explicit-any: error`) | | | `tsconfig.json` các flag; `grep -rn ": any" src app` = 0 | |
| A03 | API client duy nhất `src/api/client.ts` (openapi-fetch typed từ `schema.d.ts`), middleware: headers (`X-Client-Platform`, `X-Client-Version`, `X-Request-Id`), auth bearer, 401 → refresh 1 lần → retry, Problem Details parse | | | `grep -rn "fetch(" src app \| grep -v "src/api/"` = 0; test `client.test.ts` 401 flow | |
| A04 | Token: access trong **memory** (module scope/Zustand), refresh **chỉ** cookie httpOnly do server set; không `localStorage`/`sessionStorage` cho token; reload trang → gọi `/v1/auth/refresh` với `credentials:'include'` để lấy access mới | | | `grep -rn "localStorage\|sessionStorage" src app` = 0 (trừ theme pref nếu có); Playwright: sau login `localStorage` length = 0; `document.cookie` không chứa `rt` | |
| A05 | Problem Details mapping: `src/api/problem.ts` switch trên `code` theo bảng Contracts §2.3; mọi lỗi hiện toast + bắn event `error_shown {code, endpoint}`; `entitlement_required`/`upgrade_required` xử lý riêng | | | Test bảng: mọi `code` trong contract có case; `exhaustive` không bắt buộc (client chịu giá trị lạ → default) | |
| A06 | Data fetching admin: TanStack Query, key convention `['admin','users', params]`, cursor pagination `useInfiniteQuery` với `next_cursor`; không `useEffect` fetch | | Prefetch on hover | `grep -rn "useEffect" src/features \| grep -i fetch` = 0; `queries.ts` mỗi feature | |
| A07 | Admin route guard: `app/admin/(app)/layout.tsx` client guard: không có access → thử refresh → fail → redirect `/admin/login?next=`; role ≠ admin → 403 page | | | Playwright: vào `/admin/users` khi chưa login → redirect | |
| A08 | Bảng lớn virtualized (TanStack Table + `@tanstack/react-virtual`), 10k dòng cuộn mượt | | Column resize/persist | Playwright đo `performance.now()` scroll 10k dòng, không long task > 200 ms; DOM rows ≤ 60 | |
| A09 | Landing bundle tách khỏi admin: `no-restricted-imports` chặn `(marketing)`/`legal` import `src/features/admin` và `@tanstack/*` | | | `eslint.config.mjs` rule tồn tại; `size-limit` route `/` ≤ 90 KB | |
| A10 | i18n vi/en cho landing + legal (`app/[locale]` hoặc build 2 lần static), `html lang`, `hreflang` | | Admin i18n | `curl -s <domain>/ \| grep '<html lang="vi"'`; `hreflang` trong head | |
| A11 | Env validate zod (`src/lib/env.ts`), build fail khi thiếu; không đọc `process.env` ngoài file này | | | `grep -rn "process.env" src app \| grep -v "src/lib/env.ts"` = 0 | |
| A12 | Component chia nhỏ: không file `.tsx` > 300 dòng; page chỉ compose | | | `find app src -name '*.tsx' -exec wc -l {} + \| awk '$1>300'` = 0 | |

## 3. S — Bảo mật (S01–S08)

| ID | Mục | Target Bare | Target Perfect | Cách check | M |
|---|---|---|---|---|---|
| S01 | `public/_headers`: `X-Content-Type-Options: nosniff`, `Referrer-Policy: strict-origin-when-cross-origin`, `Strict-Transport-Security: max-age=63072000; includeSubDomains`, `Permissions-Policy: camera=(), microphone=(), geolocation=()`, `X-Frame-Options: DENY` cho `/admin/*` | CSP report-only: `default-src 'self'; script-src 'self' <posthog host>; connect-src 'self' https://api.<domain> <posthog>; img-src 'self' data: https://cdn.<domain>; style-src 'self' 'unsafe-inline'; font-src 'self'; frame-ancestors 'none'` | CSP enforcing với hash cho inline script Next | `curl -I https://<domain>` có đủ header; securityheaders.com A (O1) | |
| S02 | Không secret trong bundle: chỉ `NEXT_PUBLIC_*`; `gitleaks` 0 | | | `grep -rn "sk_\|secret" out/_next` = 0; gitleaks | |
| S03 | CORS phía server chỉ allow origin web/admin (check ở checklist 03), client gửi `credentials:'include'` chỉ cho `/v1/auth/refresh` và `/logout` | | | `grep -rn "credentials" src/api` chỉ 2 chỗ | |
| S04 | Admin: logout xoá memory token + gọi `POST /v1/auth/logout` (server clear cookie); idle 30 phút → logout | | Re-auth cho action nguy hiểm (cấp entitlement) | Playwright test logout | |
| S05 | XSS: không `dangerouslySetInnerHTML` ngoài render markdown legal đã sanitize (`rehype-sanitize`) | | | `grep -rn dangerouslySetInnerHTML src app` ≤ 1 và có sanitize | |
| S06 | Dependency: `pnpm audit --prod` 0 high; `knip` 0 unused; renovate/dependabot bật | | | Output lệnh | |
| S07 | `robots.txt` disallow `/admin`, `/design`; admin pages `noindex`; sitemap không chứa admin | | | `curl <domain>/robots.txt`; `curl <domain>/sitemap.xml \| grep admin` = 0 | |
| S08 | Không PII trong PostHog: cookieless (`persistence: 'memory'`), `distinct_id` = user uuid sau login, không autocapture input | | | `analytics.ts` config; PostHog live events không có email | |

## 4. P — Performance & SEO (P01–P10) — số từ `03-MEASUREMENT-TOOLS.md` §2

| ID | Mục | Target Bare | Target Perfect | Cách check | M |
|---|---|---|---|---|---|
| P01 | Lighthouse mobile (median 5 run, prod URL): Perf ≥ 95, A11y ≥ 95, BP 100, SEO 100 | | Perf ≥ 98 | W1 evidence | |
| P02 | LCP ≤ 2.0 s lab, CLS ≤ 0.05, TBT ≤ 150 ms; hero image `priority`, kích thước khai báo, `fetchpriority=high`; không font swap gây CLS (`font-display: optional` hoặc `size-adjust`) | LCP ≤ 1.2 s | | W1; `grep fetchpriority app/(marketing)` | |
| P03 | Budget: JS landing ≤ 90 KB gzip, CSS ≤ 20 KB, font ≤ 100 KB (Inter subset latin+vietnamese, 2 weight), ≤ 30 request, third-party ≤ 1 | | JS ≤ 50 KB | W2/W6 `size-limit` CI | |
| P04 | Fonts: `next/font/google` Inter `subsets: ['latin','vietnamese']`, `display: 'swap'` + `adjustFontFallback`; không load font từ CDN bên thứ ba | | | `grep -rn "fonts.googleapis" app` = 0 (next/font self-host) | |
| P05 | Ảnh: WebP/AVIF, pre-sized, `<img width height>`, lazy ngoài viewport; OG image 1200×630 | | | Lighthouse "Properly size images" pass | |
| P06 | Cache: `_headers` cho `/_next/static/*` `Cache-Control: public, max-age=31536000, immutable`; HTML `no-cache`; Cloudflare cache hit ≥ 95% | | | `curl -I <domain>/_next/static/...` ; W10 | |
| P07 | SEO: title/description theo trang, canonical, OG/Twitter, JSON-LD `Organization` + `SoftwareApplication` + `FAQPage`, `sitemap.xml`, `robots.txt`, `hreflang` | | | Rich Results Test 0 error (O7); GSC coverage (O6) | |
| P08 | Field vitals: `web-vitals` → PostHog `web_vital` event từ ngày đầu | | CrUX Good | W5 dashboard | |
| P09 | Admin route: First Load ≤ 250 KB, TTI ≤ 3 s trên desktop; code split theo feature (`dynamic()`) | | | W6 | |
| P10 | Không long task > 200 ms khi tương tác admin (search, mở drawer) | | | Playwright + `PerformanceObserver('longtask')` | |

## 5. X — Accessibility (X01–X07)

| ID | Mục | Target Bare | Target Perfect | Cách check | M |
|---|---|---|---|---|---|
| X01 | axe (`@axe-core/playwright`) 0 serious/critical mọi route public + admin chính | 0 moderate | | W7 evidence | |
| X02 | Keyboard: Tab order hợp lý, focus ring luôn thấy (token `focus`), Esc đóng dialog, không focus trap ngoài modal | | | Playwright keyboard spec; video 1 phút | [M] |
| X03 | Semantic HTML: `main/header/nav/footer`, heading tuần tự h1→h2→h3, 1 `h1`/trang | | | axe rule `heading-order`, `landmark-one-main` | |
| X04 | Form: label liên kết, lỗi `aria-describedby`, `aria-invalid`; OTP input `inputmode=numeric autocomplete=one-time-code` | | | grep + axe | |
| X05 | Contrast từ tokens (DS-05); không hardcode màu (DS-04) | | | DS evidence | |
| X06 | `prefers-reduced-motion` tắt animation (`tw-animate-css` + media query) | | | `grep -rn "prefers-reduced-motion" src/design-system/theme.css` ≥ 1 | |
| X07 | Touch target ≥ 44 px mobile; text ≥ 14 px; zoom không bị chặn (`user-scalable` không = no) | | | Playwright `boundingBox`; `grep user-scalable=no` = 0 | |

## 6. T — Tests (T01–T03)

| ID | Mục | Target Bare | Target Perfect | Cách check | M |
|---|---|---|---|---|---|
| T01 | Vitest + Testing Library: `problem.ts` mapping (mọi code), `client.ts` 401 flow (msw), `env.ts` fail-fast, components design-system có test render 3 trạng thái | Coverage `src/api` + `src/lib` ≥ 80% | ≥ 90% | `vitest run --coverage --reporter=json` | |
| T02 | Playwright E2E (chạy trên preview URL của PR, backend staging): landing render + CTA event; legal 5 trang 200; admin login OTP (reviewer account) → users list → user detail → cấp entitlement → audit log có dòng; logout; guard redirect | | Visual regression `toHaveScreenshot` 3 viewport | `tests/e2e/*.spec.ts` ≥ 6 spec; CI log | |
| T03 | a11y spec: axe mỗi route (X01) + keyboard spec (X02) | | | `tests/a11y.spec.ts` | |

## 7. B — Build & CI (B01–B05)

| ID | Mục | Target Bare | Target Perfect | Cách check | M |
|---|---|---|---|---|---|
| B01 | `pnpm` lockfile commit; Node LTS pin `.nvmrc`/`engines`; `pnpm install --frozen-lockfile` trong CI | | | Files tồn tại; CI log | |
| B02 | Contract drift: `openapi-typescript` chạy trong `make gen`; CI `contract.yml` fail nếu `schema.d.ts` khác | | | Workflow step `git diff --exit-code web/src/api/schema.d.ts` | |
| B03 | `web.yml`: lint (eslint + stylelint) → typecheck → vitest → build → deploy Pages preview (PR) / prod (main) → Playwright trên preview URL → LHCI assert budgets → size-limit. Bất kỳ bước fail → PR đỏ | | Visual regression step | Workflow file; 1 PR mẫu đỏ vì budget (screenshot) | |
| B04 | Deploy Cloudflare Pages qua `wrangler pages deploy out --project-name` với API token scope Pages only; preview URL comment vào PR | | | Workflow + secret tên `CF_PAGES_TOKEN` | |
| B05 | `X-Client-Version` = `NEXT_PUBLIC_BUILD_VERSION` sinh trong CI (`YYYY.MM.DD+sha7`), hiện ở footer admin | | | `curl` admin footer | |

## 8. Bare Minimum vs Perfect

| Bare | Perfect |
|---|---|
| A01–A12, S01 (không CSP enforcing), S02–S08, P01–P09, X01–X07, T01–T03, B01–B05 | CSP enforcing hash, visual regression, Storybook/`/design` đầy đủ, admin i18n, prefetch, re-auth cho action nguy hiểm, web app học (ngoài phạm vi landing/admin) |

## 9. G — Prompt cho AI (copy nguyên văn)

```
Bạn là Web reviewer (Next.js 16 static export). Đầu vào: docs/checklists/02-web-react.md, docs/01-CONTRACTS.md,
contracts/openapi.yaml, thư mục web/ (toàn bộ, gồm public/_headers, next.config.ts, eslint.config.mjs, lighthouserc.json,
.size-limit.json, .github/workflows/web.yml) và evidence docs/evidence/web/ nếu có.
Kiểm tra từng mục A01→A12, S01→S08, P01→P10, X01→X07, T01→T03, B01→B05.
Với mỗi mục trả về đúng 1 dòng bảng:
ID | PASS/FAIL/N-A | bằng chứng (file:dòng hoặc lệnh + output rút gọn) | việc cần sửa (nếu FAIL)
Quy tắc:
- Không PASS nếu không có bằng chứng. Số đo phải trích từ file evidence (lhci json, size-limit json, axe json), không ước lượng.
- Mục [M] chỉ ghi "CẦN MANUAL".
- Dán output các grep trong cột "Cách check".
- Cuối cùng liệt kê: mọi fetch() ngoài src/api, mọi localStorage/sessionStorage, mọi process.env ngoài env.ts,
  mọi hex màu/px hardcode trong tsx, mọi dangerouslySetInnerHTML, mọi file tsx > 300 dòng.
- Kết thúc bằng bảng tổng PASS/FAIL/MANUAL theo nhóm.
```

## 10. Re-check thủ công (bạn làm, ~30 phút)

1. **Token (A04)**: login admin trên prod, mở DevTools → Application → Local/Session Storage rỗng; Cookies của `admin.<domain>` không có token; Cookies của `api.<domain>` có `rt` HttpOnly ✓ Secure ✓ SameSite=Strict ✓ Path=/v1/auth/refresh ✓. Reload trang → vẫn đăng nhập (refresh chạy).
2. **Keyboard (X02)**: đi hết landing và 1 màn admin chỉ bằng bàn phím; luôn thấy focus ring; Esc đóng dialog.
3. **Mobile thật**: mở landing trên iPhone qua 4G (tắt Wi-Fi), cảm nhận < 2 s; Safari Reader mode hiển thị đúng (semantic ok).
4. **Header (S01)**: `curl -I https://<domain>` và `https://admin.<domain>/admin/login` — đọc từng header theo danh sách.
5. **Ngôn ngữ (A10)**: đổi ngôn ngữ trình duyệt → landing đúng ngôn ngữ; `hreflang` trong source.
6. **Legal (06 L01–L05)**: 5 URL mở được, có ngày cập nhật, có email liên hệ, đúng tên pháp lý.

## 11. Evidence phải có khi đóng Phase 3

`docs/evidence/web/`: `<date>-lhci/`, `-size-limit.json`, `-axe.json`, `-playwright.json`, `-wpt.md`, `-unlighthouse/`, `-lint.json`, `-knip.json`, `-cf-cache.png`, `checklist-02-run-<date>.md`.
