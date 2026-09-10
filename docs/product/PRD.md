# PRD — Shadow (v0.1, TODO: điền sau phỏng vấn P0-01)

> 1 trang. Mọi ô `TODO` phải được điền trước khi đóng P0-01. Không có mục "tính năng mong muốn" — chỉ có core loop.
> File này là **input** cho: P0-05 (chọn endpoint `M`), P0-08 (copy landing), P0-10 (asset cần bảo vệ), 06-domain (ASO).

## 1. Một câu

Shadow giúp **[persona]** **[kết quả]** bằng cách **[cơ chế: lặp lại ngắt quãng FSRS + audio + ví dụ thật]** trong **[X phút/ngày]**.

`TODO`

## 2. Persona chính (1 người, không phải "mọi người học tiếng Anh")

| Trường | Giá trị |
|---|---|
| Tên gọi persona | `TODO` (ví dụ: "Sinh viên năm 3 chuẩn bị IELTS 6.5") |
| Tuổi, nghề, thiết bị | `TODO` (phải là iPhone — D11) |
| Trình độ hiện tại (CEFR) | `TODO` |
| Mục tiêu 90 ngày | `TODO` |
| Đang dùng gì để học từ vựng | `TODO` (Anki? Quizlet? sổ tay?) |
| Đã trả tiền cho app học nào, bao nhiêu | `TODO` |
| Thời điểm học trong ngày | `TODO` (quyết định giờ nhắc mặc định IOS-13) |

## 3. Job-to-be-done

Khi `TODO tình huống`, tôi muốn `TODO động lực`, để `TODO kết quả`.

Nỗi đau xếp hạng (từ 10 phỏng vấn):

| # | Nỗi đau | Số người nhắc /10 | Shadow giải quyết? |
|---|---|---|---|
| 1 | `TODO` | | Y/N |
| 2 | `TODO` | | |
| 3 | `TODO` | | |

## 4. Ba core loop (đây là toàn bộ MVP)

| Loop | Mô tả 1 câu | Endpoint `M` liên quan (01-CONTRACTS §4) | Màn hình iOS |
|---|---|---|---|
| L1 — Học hằng ngày | Mở app → queue thẻ đến hạn → rating 4 mức → xong trong ≤ 10 phút, **offline được** | `GET /v1/study/queue`, `POST /v1/study/reviews` | Study |
| L2 — Thêm từ | Tra từ điển → xem IPA/audio/ví dụ → thêm vào deck | `GET /v1/dictionary/search`, `POST /v1/decks/{id}/cards` | Decks, Dictionary |
| L3 — Thấy tiến bộ | Streak, số từ nhớ, biểu đồ 30 ngày | `GET /v1/study/stats` | Stats |

`TODO`: xác nhận 3 loop này đúng với persona; nếu phỏng vấn cho thấy L2 không quan trọng (họ muốn deck sẵn), đổi L2 thành "Chọn deck hệ thống theo mục tiêu".

## 5. Non-goals (từ chối rõ ràng ở MVP)

- Android, web app học (chỉ landing + admin).
- Chat AI / hội thoại.
- Chấm phát âm (Perfect tier).
- Ngữ pháp, bài đọc, nghe dài.
- Social / leaderboard.
- `TODO` thêm nếu phỏng vấn bật ra.

## 6. Một success metric

| | Giá trị |
|---|---|
| Metric | **D7 retention của user đã hoàn thành ≥ 1 session học** |
| Bare | ≥ 15% |
| Tốt | ≥ 25% |
| Đo bằng | PostHog cohort `first_session_completed` → `session_completed` ngày 7 |
| Vì sao chỉ 1 | Nếu người ta quay lại sau 7 ngày thì loop L1 hoạt động; mọi thứ khác là hệ quả |

Ngưỡng waitlist để bắt đầu Phase 1: **`TODO` (gợi ý 30)** đăng ký thật từ kênh không phải bạn bè.

## 7. Pricing giả định (kiểm chứng ở TestFlight, chi tiết `pricing.md`)

| Gói | Giá | Bao gồm |
|---|---|---|
| Free | 0 | 1 deck hệ thống, 20 thẻ mới/ngày, stats 7 ngày |
| Pro tháng | `TODO` (tham chiếu 49–79k VND) | Không giới hạn, stats đầy đủ, audio offline |
| Pro năm | `TODO` (≈ 7 tháng) | như trên |

## 8. Kết quả phỏng vấn

| # | Alias | Sẽ thử? | Lý do 1 câu | File |
|---|---|---|---|---|
| 1 | | | | `interviews/01-*.md` |
| … | | | | |
| 10 | | | | |

Tổng "sẽ thử": `__/10` (cần ≥ 7).
