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

| Chỉ số | `eval_before.json` | `eval_after.json` |
|---|---|---|
| Điểm tổng | 93.41/100 (22/22) | Chưa có eval đầy đủ |
| Điểm case đã chọn | H7: 37/100 | H7: 97/100 (`eval_groq_h7.json`, 1/1 case) |
| Số model calls | H7: 3 lần gọi agent | H7: 4 lần agent + 1 lần judge |
| Token, chi phí nếu có | Xem usage trong file eval; file không ghi chi phí tiền tệ | H7: 4,477 agent tokens + 2,397 judge tokens; không có số tiền chi phí |

- Lỗi đã giải quyết: trong lần chạy riêng H7, trace ghi `validation_missing_status_disclosure` sau khi model bỏ sót timeout; câu trả lời cuối nêu rõ không thể xác minh Cloudflare, và judge chấm 97/100.
- Lỗi còn lại hoặc mới xuất hiện: chưa biết điểm tổng sau thay đổi vì `eval_after.json` bị 429 và `eval_groq_h7.json` chỉ chứa H7. Không dùng điểm H7 riêng làm điểm toàn bộ. Khi rà soát thủ công, nguồn GitHub trong câu trả lời H7 là đường dẫn giả dạng tool (`/functions.get_status?...`), không phải URL thật `https://www.githubstatus.com` từ tool result; điểm judge 97 không chứng minh mọi chi tiết đều hoàn hảo.

## 5. Giới hạn

Gate kiểm tra câu trả lời có thông báo provider chưa xác minh, không chứng minh mọi claim khác đều đúng hoặc được dẫn nguồn. Điểm judge là ước lượng trên snapshot cố định. H7 sau thay đổi được chạy riêng thành công; cần chạy đủ 22 case sau thay đổi để có điểm tổng có thể so sánh với baseline. Lần chạy trước đã chấm đủ 22 case nhưng mỗi phiên bản chỉ chạy một lần.
