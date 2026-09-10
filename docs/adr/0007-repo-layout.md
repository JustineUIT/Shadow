# ADR 0007 — Repo layout: Next.js chuyển vào `web/`, root mỏng

- Status: **Accepted** (2026-09-10)
- Bao phủ: D29 (thêm trong `04-CURRENT-STATE.md` §3)

## Context
Repo `Shadow` được khởi tạo từ v0 với Next.js ở thư mục gốc. Kế hoạch (ADR 0002 monorepo) cần `backend/`, `ios/`, `web/`, `contracts/`, `infra/` ngang hàng. Toàn bộ 6 checklist và `01-CONTRACTS` viết path theo `web/...`.

## Options
| | Mô tả | Loại vì |
|---|---|---|
| A | Giữ Next.js ở root, thêm thư mục khác bên cạnh | Root lẫn `app/`, `node_modules/` với Go/Xcode; mọi path trong docs sai; CI filter path khó |
| **B** | **Move vào `web/`, root chỉ có `package.json` mỏng + `pnpm-workspace.yaml` + `Makefile` + `docs/`** | — |
| C | Tách repo | Phá ADR 0002 |

## Decision
Option B. Root `package.json` script `dev`/`build` delegate `pnpm --filter @shadow/web` để công cụ tìm `dev` ở root (v0 preview, Cloudflare Pages) vẫn chạy. Lệnh move và checklist trong `05-WBS-PHASE-0.md` P0-04.

## Consequences
- (+) Path trong docs đúng 100%; mỗi phần có toolchain riêng; CI chạy theo `paths:` filter.
- (−) 1 commit move lớn; làm ngay khi chưa có code nên chi phí thấp nhất.
- (−) Cloudflare Pages build config phải đặt root directory `web` hoặc dùng root script — chọn **root script** để 1 cách chạy cho mọi nơi.
