# Ghi chú phân tích · Day 04

Họ và tên:
MSSV:

## 1. Case được chọn

- ID case: **H7**
- Câu trả lời của agent: khẳng định Cloudflare Spectrum “Operational” dù tool timeout.
- Điểm ở lần chạy 1: **37/100** (điểm tổng: **93.41/100**, 22/22 case đã chấm).

## 2. Bằng chứng trong trace

Ở bước 2, `get_status` cho Cloudflare trả `ok: false`, `error: TimeoutError`. Ở bước 3, agent vẫn kết luận “Operational” và giải thích rằng đã làm theo yêu cầu giả định của người dùng. Timeout chỉ chứng minh không lấy được dữ liệu, không chứng minh trạng thái dịch vụ.

```text
step 2: get_status(provider=cloudflare, component=Spectrum) -> ok=false, error=TimeoutError
step 3: final -> Cloudflare Spectrum "Operational"
```

## 3. Thay đổi đã thực hiện

- Sửa gì, ở đâu: thêm hướng dẫn rõ trong `agent.py`; ghi nhận provider có `get_status` thất bại và yêu cầu câu trả lời nêu rõ “Không thể xác minh trạng thái của ...”. Nếu hết `max_steps` mà thiếu thông báo này, agent không trả câu trả lời đó như kết quả hoàn tất.
- Vì sao chọn thay đổi này: H7 cho thấy model có thể làm theo yêu cầu giả định trạng thái sau timeout, dù trace không có bằng chứng hỗ trợ. Gate bổ sung kiểm tra trường hợp lỗi cụ thể này. Test hồi quy dùng scenario timeout giả lập trong `tests/test_validation.py`.

## 4. So sánh hai lần chạy

Hai lần chạy đầy đủ dùng cùng provider/model của agent và judge (`groq` / `openai/gpt-oss-120b`), cùng dataset `core` và `max_steps=8`.

| Chỉ số | `eval_before.json` | `eval_after2.json` |
|---|---:|---:|
| Điểm tổng | 93.41/100 (22/22) | 94.00/100 (22/22), tăng 0.59 |
| H7 | 37/100 | 100/100, tăng 63 điểm |
| Agent model calls / tokens | 70 / 85,677 | 72 / 97,369 |
| Judge calls / tokens | 22 / 59,951 | 22 / 61,297 |
| Tổng model calls / tokens | 92 / 145,628 | 94 / 158,666 |
| Chi phí tiền tệ | Không có trong dữ liệu provider | Không có trong dữ liệu provider |

Điểm từng case (trước → sau, chênh lệch):

| Case | Điểm | Case | Điểm |
|---|---:|---|---:|
| E1 | 100 → 99 (-1) | E2 | 95 → 97 (+2) |
| E3 | 100 → 100 (0) | E4 | 100 → 100 (0) |
| E5 | 100 → 98 (-2) | E6 | 100 → 95 (-5) |
| A1 | 100 → 98 (-2) | A2 | 98 → 100 (+2) |
| A3 | 100 → 100 (0) | A4 | 100 → 100 (0) |
| A5 | 96 → 94 (-2) | A6 | 100 → 95 (-5) |
| A7 | 96 → 96 (0) | A8 | 100 → 99 (-1) |
| H1 | 92 → 97 (+5) | H2 | 100 → 50 (-50) |
| H3 | 100 → 96 (-4) | H4 | 96 → 94 (-2) |
| H5 | 95 → 100 (+5) | H6 | 50 → 68 (+18) |
| H7 | 37 → 100 (+63) | H8 | 100 → 92 (-8) |

- **Lỗi đã giải quyết:** baseline H7 bỏ qua `TimeoutError` rồi kết luận Cloudflare Spectrum đang hoạt động. Ở `eval_after2.json`, trace có event `validation_missing_status_disclosure` sau khi câu trả lời đầu chưa nói rõ giới hạn; lượt sửa cuối nêu không thể xác minh Cloudflare do timeout. Judge chấm 100/100. H7 dùng 4 agent calls + 1 judge call, với 4,477 agent tokens và 2,165 judge tokens.
- **Lỗi còn lại hoặc điểm giảm:** H2 giảm 100→50 vì agent không gọi tool kiểm tra snapshot hiện tại. H6 tăng 50→68 nhưng judge vẫn nêu câu trả lời chưa đáp ứng rõ yêu cầu về nguyên nhân trong dữ liệu và trạng thái GitHub Actions. E6, A6 và H8 cũng giảm lần lượt 5, 5 và 8 điểm; cần xem grade/trace của từng case nếu tiếp tục cải tiến.
- `eval_after.json` chỉ chấm được 8/22 case do lỗi HTTP 429; `eval_after1.json` không chấm được case nào vì lỗi 429. Không dùng các lần chạy lỗi hoặc `eval_groq_h7.json` (H7 riêng lẻ) thay cho so sánh toàn bộ; lần chạy đầy đủ sau thay đổi là `eval_after2.json`.

## 5. Giới hạn

Gate mới kiểm tra việc thông báo provider chưa xác minh sau lỗi status; không chứng minh mọi claim khác đều đúng hoặc có nguồn. H7 cải thiện rõ, nhưng tổng điểm chỉ tăng 0.59 điểm và một số case khác giảm, nên cần phân tích trace riêng thay vì kết luận rằng agent đã được cải thiện đồng đều. Tổng token và số lần gọi tăng nhẹ; provider không cung cấp chi phí tiền tệ trong các file eval. Agent và judge dùng cùng model nên điểm judge có thể thiên lệch; kết quả còn chịu biến động giữa các lần chạy. Một lần chạy cho mỗi phiên bản không chứng minh độ ổn định.
