# docs/ — Bản đồ tài liệu dự án Shadow

Trả lời 10 yêu cầu gốc bằng file nào:

| Yêu cầu | File |
|---|---|
| 1. iOS Swift best practice | `checklists/01-ios-swift.md`, `adr/0004` |
| 2. Front end React best practice | `checklists/02-web-react.md`, `adr/0003` (Pages), `adr/0007` |
| 3. Backend Go best practice | `checklists/03-backend-go.md`, `adr/0001`, `adr/0005` |
| 4. Database best practice | `checklists/04-database-postgres.md`, `adr/0003` |
| 5. Server best practice | `checklists/05-server-infra.md`, `adr/0003` |
| 6. Domain và phần còn lại | `checklists/06-domain-ops.md`, `adr/0006` |
| 7. Tools đo số liệu thật (≥ 6, mỗi phần 1 tool chính) | `03-MEASUREMENT-TOOLS.md` (I/W/B/D/S/O series), kết quả điền vào `evidence/SCORECARD.md` |
| 8. 6 file .md check source + target perfect, prompt AI, re-check tay | `checklists/01..06` — mỗi file có: mục 0 biên giới I/O, bảng Bare/Perfect, cột Evidence, mục "Prompt cho AI", mục "Re-check thủ công `[M]`", mục Evidence |
| 9. Input/Output must-have vs optional, design system chuẩn từ đầu | `01-CONTRACTS.md` (API/DB/tokens/events/fixtures), `02-DESIGN-SYSTEM.md` (tokens 3 lớp, Bare vs Perfect) |
| 10. Rõ ràng, file output kỳ vọng, break-down lớn → nhỏ, options → chọn | `00-MASTER-PLAN.md` (decision log 29 dòng, phase, cây thư mục), `04-CURRENT-STATE.md` (thực tế repo), `05-WBS-PHASE-0.md` (từng lệnh) |

## Thứ tự đọc cho người mới (hoặc AI mới)

1. `04-CURRENT-STATE.md` — repo đang ở đâu (5 phút).
2. `00-MASTER-PLAN.md` §1 decision log + §5 phase.
3. `05-WBS-PHASE-0.md` — việc kế tiếp, làm theo thứ tự.
4. Khi chạm biên giới giữa 2 hệ: `01-CONTRACTS.md`. Khi viết UI: `02-DESIGN-SYSTEM.md`.
5. Cuối mỗi task lớn: checklist tương ứng + `03-MEASUREMENT-TOOLS.md` → điền `evidence/SCORECARD.md`.

## Cấu trúc thư mục

```
docs/
├── README.md                 # file này
├── 00-MASTER-PLAN.md         # decision log, phase, exit criteria
├── 01-CONTRACTS.md           # 4 source of truth, endpoint catalogue, env contract
├── 02-DESIGN-SYSTEM.md       # tokens, palette, typography, components
├── 03-MEASUREMENT-TOOLS.md   # tool đo theo 6 phần + scorecard template
├── 04-CURRENT-STATE.md       # audit repo thực tế + gap + D29
├── 05-WBS-PHASE-0.md         # break-down Phase 0 tới mức lệnh
├── checklists/01..06-*.md    # 6 checklist AI-check + manual re-check
├── adr/                      # 0001–0007 (+ README index)
├── evidence/SCORECARD.md     # số liệu thật; thư mục con ios/ web/ backend/ db/ server/ domain/ design/
├── product/PRD.md, naming.md # template điền ở P0-01, P0-02
├── ops/accounts.md           # template điền ở P0-03 (không secret)
├── security/                 # threat-model.md (P0-10)
├── db/                       # ERD.md (P0-07)
├── ops/runbooks/, legal/, growth/, reviews/   # Phase 1+
```

## Quy ước viết docs

- Tiếng Việt, thuật ngữ kỹ thuật giữ tiếng Anh.
- Không ước lượng thời gian; chỉ có exit criteria và DoD kiểm được bằng lệnh.
- Placeholder `<domain>`, `<date>`, `<hash>` giữ nguyên trong docs; giá trị thật chỉ ở `ops/accounts.md` và tên file evidence.
- Mọi claim "đạt" phải trỏ tới file trong `evidence/`.
