# ADR 0006 — Lõi học FSRS, dữ liệu mở, TTS self-host, observability free tier, email, admin UI

- Status: **Accepted** (2026-09-10)
- Bao phủ: D18, D20, D22, D23, D25, D26

## Context
Lõi sản phẩm là lặp lại ngắt quãng; dữ liệu từ điển phải hợp pháp và offline được; audio nhiều nhưng chi phí phải ~0; cần đo được mọi target trong checklist mà không trả tiền.

## Options → Decision
| Hạng mục | Options | Chọn | Lý do |
|---|---|---|---|
| Thuật toán | SM-2; Leitner; **FSRS** | FSRS, `go-fsrs` server-side là source of truth; Swift port cùng version cho preview offline; parity bằng `fsrs_cases.json` | Hiện đại, có optimizer per-user (Perfect tier) |
| Dữ liệu từ điển | Crawl Oxford/Cambridge (vi phạm ToS); LLM sinh (tốn token, sai); **Wiktionary kaikki.org + Tatoeba + CMUdict + NGSL** | Import 1 lần vào Postgres, trim theo tần suất | CC BY-SA / CC BY, attribution page trong app + web |
| TTS | ElevenLabs; OpenAI TTS; **Piper self-host pre-render → R2** | Bulk top 5000 từ trước, opus ≤ 20 KB; API TTS chỉ on-demand (Perfect) | Chi phí ~0, cache vĩnh viễn |
| Observability | Datadog; New Relic; **Sentry + Grafana Cloud free + PostHog** | Sentry (error 3 platform), Grafana (Prometheus/Loki/Tempo qua OTel), PostHog (product, cookieless) | Free tier đủ; mỗi target trong 03-MEASUREMENT có tool đo |
| Email | SES; Postmark; **Resend** | OTP + waitlist + transactional | Free tier, DKIM dễ, API sạch |
| Admin UI | Retool; Refine; React-Admin; **shadcn/ui + TanStack Table/Query** | Cùng tokens với landing; đủ CRUD | Không thêm hệ thống |

## Consequences
- (+) Toàn bộ content có thể phát hành offline (SQLite pack) sau này vì license cho phép.
- (−) Wiktionary CC BY-SA: dữ liệu derived giữ CC BY-SA; **code app không bị ảnh hưởng**; phải có trang attribution (06 L04).
- (−) Piper chất lượng thấp hơn ElevenLabs; chấp nhận cho MVP, đo bằng phản hồi TestFlight.
- (−) FSRS 2 implementation (Go + Swift): rủi ro lệch → fixtures 200 case + test cả 2 phía bắt buộc xanh trước merge.

## Checklist liên quan
03-backend A08 (FSRS), 04-database mục C (content), 06-domain L04 (licenses), O-series (PostHog), 01-ios P01 (Sentry/MetricKit).
