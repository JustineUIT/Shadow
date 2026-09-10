# ADR 0002 — Contract-first: một file sinh ba phía

- Status: **Accepted** (2026-09-10)
- Bao phủ: D02, D14, D19, D21, D24

## Context
Nỗi đau được nêu tường minh trong yêu cầu: "làm cái này không khớp cái kia, sửa tới sửa lui". Có 3 client/server viết bằng 3 ngôn ngữ (Go, Swift, TS) và 2 UI platform (iOS, web) cần cùng token màu/chữ.

## Options
| Biên giới | Options | Loại vì |
|---|---|---|
| API | Mỗi bên tự viết theo doc; GraphQL; gRPC; **OpenAPI 3.1 contract-first** | Tự viết: lệch chắc chắn. GraphQL: over-fetch control không cần, cache CDN khó. gRPC: web/iOS tooling nặng, không cache HTTP |
| Client iOS | Alamofire + model tay; **swift-openapi-generator** | Model tay = lệch |
| Client web | axios + type tay; **openapi-typescript + openapi-fetch** | như trên |
| Design tokens | Viết tay 2 nơi; Figma Tokens plugin; **tokens.json (DTCG) + Style Dictionary** | Viết tay lệch; Figma plugin phụ thuộc Figma |
| Repo | Multi-repo; **Monorepo** | Contract + 3 consumer trong 1 PR, CI drift 1 chỗ |

## Decision
- `contracts/openapi.yaml` là **source of truth duy nhất** của API. Sinh: Go (oapi-codegen strict server + types), Swift (swift-openapi-generator build plugin), TS (openapi-typescript).
- `contracts/tokens/tokens.json` sinh `Tokens.swift` + `tokens.css` qua Style Dictionary.
- `contracts/events.yaml` sinh `Events.swift` + `events.ts`.
- `contracts/fixtures/fsrs_cases.json` là fixture parity Go ↔ Swift.
- Code sinh **được commit**. CI job `contract-drift`: `make gen && git diff --exit-code`.
- Quy trình đổi API: sửa yaml → gen → mới sửa code. Breaking change theo checklist `01-CONTRACTS.md` §9.

## Consequences
- (+) Không thể có field lệch giữa 3 phía mà CI xanh.
- (+) `examples` trong yaml dùng làm test decode Swift + schemathesis fuzz.
- (−) Học 3 generator; lỗi generator (đặc biệt Swift với `oneOf`) phải tránh bằng schema đơn giản: không `oneOf/anyOf` ở MVP, dùng `enum` + field optional.
- (−) Codegen commit → diff PR to; chấp nhận, review chỉ phần tay.

## Checklist liên quan
01-CONTRACTS §1, §6, §9. 03-backend B04. 02-web B03. 01-ios A04.
