# 04 — CURRENT STATE: Repo `Shadow` đang ở đâu so với kế hoạch

> Audit tại commit `99b67e6` ("braining storm"), branch `main`, ngày 2026-09-10.
> Mục đích: đối chiếu **thực tế** với `00-MASTER-PLAN.md` để biết chính xác việc kế tiếp là gì, không đoán.
> Tên dự án chính thức: **Shadow**. Mọi chỗ trong docs ghi `englishapp/` phải hiểu là `shadow/` (repo root).

## 0. Kết luận 30 giây

| Câu hỏi | Trả lời |
|---|---|
| Đang ở phase nào? | **Trước Phase 0**. Chỉ có tài liệu kế hoạch, chưa có artifact Phase 0 nào (`contracts/`, ADR, ERD, PRD). |
| Code sản phẩm đã có gì? | **Không có.** Thư mục gốc là scaffold Next.js 16 mặc định của v0 (trang placeholder). |
| Cấu trúc repo có khớp kế hoạch không? | **Không.** Kế hoạch: `web/` là thư mục con của monorepo. Thực tế: Next.js nằm ở root. Phải quyết định (xem §3). |
| Việc làm ngay tiếp theo? | Tái cấu trúc repo thành monorepo (P0-04) → tạo `contracts/` skeleton (P0-05, P0-06) → ADR (P0-09). Chi tiết từng bước trong `05-WBS-PHASE-0.md`. |

## 1. Inventory: file hiện có và số phận của nó

Ký hiệu: **KEEP** giữ nguyên, **MOVE** chuyển vào `web/`, **EDIT** giữ nhưng phải sửa, **DELETE** bỏ, **DOC** tài liệu.

| File hiện tại | Loại | Quyết định | Lý do / việc phải làm |
|---|---|---|---|
| `docs/00-MASTER-PLAN.md` | DOC | EDIT | Đổi `englishapp/` → `shadow/`; thêm dòng cho file 04, 05, `adr/` vào bảng §0 |
| `docs/01-CONTRACTS.md` | DOC | KEEP | Đầy đủ (443 dòng, 9 mục). Là input cho P0-05 |
| `docs/02-DESIGN-SYSTEM.md` | DOC | KEEP | Đầy đủ (359 dòng). Là input cho P0-06 |
| `docs/03-MEASUREMENT-TOOLS.md` | DOC | KEEP | Đầy đủ. Template SCORECARD ở §8 chưa được tạo thành file thật → tạo `docs/evidence/SCORECARD.md` |
| `docs/checklists/01..06-*.md` | DOC | KEEP | 6 checklist đầy đủ, mỗi file có mục 0 (biên giới I/O), Bare/Perfect, prompt AI, re-check `[M]`, evidence |
| `app/page.tsx` | Web | DELETE (thay) | Placeholder của v0. Thay bằng landing "coming soon" ở P0-08 |
| `app/layout.tsx` | Web | MOVE + EDIT | `metadata.title = 'v0 App'` → Shadow; bỏ `@vercel/analytics` (D17: deploy Cloudflare Pages, analytics là PostHog — D22); `lang="en"` → `"vi"` (02-web A10); thêm `className="bg-background"` vào `<html>` |
| `app/globals.css` | Web | MOVE + EDIT | Đang là token mặc định shadcn (oklch xám). Phải được **sinh** từ `contracts/tokens/tokens.json` (D21). Giữ cấu trúc `@theme inline`, thay giá trị bằng import `tokens.css` generated |
| `components/ui/button.tsx` | Web | MOVE | Dùng lại, nhưng variant phải map sang token semantic của 02-DS §3 |
| `components.json` | Web | MOVE | shadcn config, cập nhật path alias sau khi move |
| `lib/utils.ts` | Web | MOVE | `cn()` giữ |
| `next.config.mjs` | Web | MOVE + EDIT | **Xoá `typescript.ignoreBuildErrors: true`** (vi phạm WEB-01 "0 warning", 02-web A01 TS strict). Thêm `output: 'export'` (D17). `images.unoptimized: true` giữ (bắt buộc với static export). Đổi sang `next.config.ts` |
| `package.json` | Web | MOVE + EDIT | `name: my-project` → `@shadow/web`; bỏ `@vercel/analytics`; thêm scripts `lint`, `typecheck`, `test` (02-web B01) |
| `pnpm-lock.yaml`, `pnpm-workspace.yaml` | Repo | EDIT | `pnpm-workspace.yaml` ở root khai báo `packages: ['web', 'contracts/tokens']` |
| `postcss.config.mjs`, `tsconfig.json` | Web | MOVE | tsconfig bật `strict`, `noUncheckedIndexedAccess` |
| `public/*.png, *.svg, *.jpg` | Web | MOVE + thay | Icon v0 mặc định → thay bằng icon Shadow khi có logo (P0-02). `placeholder-*` xoá khi không dùng |
| `.gitignore` | Repo | EDIT | Thêm: `backend/bin/`, `*.xcuserstate`, `ios/**/DerivedData`, `infra/compose/.env`, `docs/evidence/**/*.raw` (file thô > 5 MB đẩy lên R2, chỉ giữ summary) |

**Thiếu hoàn toàn** (kế hoạch có, repo chưa có): `contracts/`, `backend/`, `ios/`, `infra/`, `.github/workflows/`, `docs/adr/`, `docs/evidence/`, `docs/product/`, `docs/security/`, `docs/ops/`, `docs/db/`, `docs/legal/`, `docs/growth/`, `docs/reviews/`, `CODEOWNERS`, `.editorconfig`, `Makefile` root.

## 2. Gap theo 6 phần (đối chiếu Bare Minimum của từng checklist)

| # | Phần | Bare Minimum đã có | Chưa có | Task đóng gap |
|---|---|---|---|---|
| 1 | iOS Swift | Checklist 01 + quyết định D11–D14, D16, D27 | Toàn bộ `ios/`. Chưa có Apple Developer account xác nhận | P0-03 (account), Phase 2 |
| 2 | Web React | Scaffold Next 16 + Tailwind v4 + shadcn (đúng stack D17/D18) | Static export, tokens generated, landing, legal, admin, CI, `_headers` | P0-04 (move), P0-06, P0-08, Phase 3 |
| 3 | Backend Go | Checklist 03 + D01–D06, D15 | Toàn bộ `backend/`, `openapi.yaml` | P0-05, Phase 1 |
| 4 | Database | Checklist 04 + D07 | Neon project, ERD, migration 0001, dữ liệu từ điển | P0-03, P0-07, Phase 1 DATA-* |
| 5 | Server | Checklist 05 + D08–D10 | VPS, compose, Caddy, scripts | P0-03, Phase 1 OPS-* |
| 6 | Domain/Ops | Checklist 06 + D22–D23 | Tên chính thức chưa check trademark/App Store; chưa mua domain; chưa có account nào ghi trong `docs/ops/accounts.md` | P0-02, P0-03 |

Chưa có **bất kỳ số liệu nào** trong `docs/evidence/` → mọi ô SCORECARD hiện là "chưa đo", không phải FAIL.

## 3. Quyết định tái cấu trúc repo (D29)

Vấn đề: Next.js ở root xung đột với monorepo `backend/ ios/ web/ contracts/ infra/`.

| Option | Mô tả | Ưu | Nhược |
|---|---|---|---|
| A | Giữ Next.js ở root, thêm `backend/`, `ios/` làm thư mục con | Không phải move, v0 preview chạy ngay | Root bị lẫn `app/`, `components/` với `backend/`; `pnpm-lock` root gắn với web; Xcode/Go tooling khó chịu vì `node_modules` ở root; không khớp mọi path trong 6 checklist |
| B | **Move Next.js vào `web/`, root chỉ giữ `pnpm-workspace.yaml` + `package.json` mỏng + `Makefile` + `docs/`** | Khớp 100% path trong docs; mỗi phần độc lập tooling; CI filter theo path | Phải move 1 lần; v0 preview cần root `package.json` có script `dev` |
| C | Tách 4 repo | Độc lập tuyệt đối | Phá D24 (monorepo), contract khó đồng bộ |

**CHỌN B.** Để v0 preview (và bất kỳ tool nào tìm `dev` ở root) vẫn chạy, root `package.json` chỉ có:

```json
{
  "name": "shadow",
  "private": true,
  "scripts": {
    "dev": "pnpm --filter @shadow/web dev",
    "build": "pnpm --filter @shadow/web build",
    "gen": "make gen"
  }
}
```

Cấu trúc đích sau P0-04 (chỉ liệt kê phần thay đổi so với `00-MASTER-PLAN.md` §3):

```
shadow/                              # = repo root (tên repo giữ "Shadow")
├── package.json                     # mỏng, chỉ delegate (ở trên)
├── pnpm-workspace.yaml              # packages: ['web', 'contracts/tokens']
├── Makefile                         # gen | lint | test | evidence  (gọi xuống từng phần)
├── .editorconfig  CODEOWNERS  .gitignore
├── contracts/  backend/  ios/  web/  infra/  .github/  docs/   # như 00-MASTER-PLAN §3
```

Lệnh move (chạy 1 lần, commit riêng "chore: move web to web/ (D29)"):

```bash
mkdir web
git mv app components lib public components.json next.config.mjs postcss.config.mjs tsconfig.json web/
git mv package.json web/package.json
git mv pnpm-lock.yaml web/pnpm-lock.yaml   # rồi chạy pnpm install ở root để sinh lock mới của workspace
# tạo root package.json, pnpm-workspace.yaml theo mẫu trên
pnpm install && pnpm dev                   # DoD: preview vẫn mở được trang
```

ADR tương ứng: `docs/adr/0007-repo-layout.md`.

## 4. Những điểm trong scaffold v0 **trái** với checklist (phải sửa, không được mang theo)

| Phát hiện | Vi phạm | Sửa |
|---|---|---|
| `typescript.ignoreBuildErrors: true` | 02-web A01 (TS strict, build 0 lỗi) | Xoá. Build phải đỏ khi có lỗi type |
| `@vercel/analytics` | D17 (Cloudflare Pages), D22 (PostHog), 06 O-series (cookieless analytics) | Gỡ package |
| Không có `output: 'export'` | D17 | Thêm |
| `<html lang="en">` | 02-web A10 (vi mặc định, hreflang) | `lang="vi"`, cấu trúc `app/[locale]` ở WEB-02 |
| Token màu là default shadcn (xám oklch) | D21, 02-DS §3 (palette đã đo contrast) | Sinh `tokens.css` từ `tokens.json`, không sửa tay `globals.css` |
| `metadata.title = 'v0 App'` | 02-web S-series/SEO | Đặt theo `docs/product/naming.md` |
| Không có `lint`/`typecheck`/`test` scripts | 02-web B01 | Thêm eslint (next + a11y), `tsc --noEmit`, vitest |

## 5. Những gì docs còn **thiếu** để "làm ngay được" (và file nào đóng nó)

| Thiếu | File tạo trong lần này |
|---|---|
| Break-down Phase 0 xuống mức từng lệnh/ từng file | `05-WBS-PHASE-0.md` |
| ADR cho D01–D29 (P0-09) | `docs/adr/0001..0007-*.md` |
| File SCORECARD thật (03 §8 chỉ có template) | `docs/evidence/SCORECARD.md` |
| Template PRD, naming, accounts (P0-01..03) để điền | `docs/product/PRD.md`, `naming.md`, `docs/ops/accounts.md` (template có sẵn cột) |
| Hướng dẫn cách đọc `docs/` cho người mới / AI | `docs/README.md` |

## 6. Điều **không** thay đổi

Toàn bộ 28 quyết định D01–D28, bảng Bare/Perfect, KPI, chi phí trong `00-MASTER-PLAN.md` giữ nguyên. Audit này chỉ thêm D29 (repo layout) và chỉ ra scaffold hiện tại cần sửa gì.
