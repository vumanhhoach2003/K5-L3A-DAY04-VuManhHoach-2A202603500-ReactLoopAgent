# Ghi chú phân tích · Day 04

Họ và tên: Vũ Mạnh Hoạch
MSSV: 2A202603500

## 1. Case được chọn

- ID case: E2
- Câu trả lời của agent (lần 1):
  > "Trong dữ liệu snapshot có ghi nhận sự cố liên quan đến Cloudflare Cache Purge mang mã wp49bp8bqo6c ('Fedramp Customers have Delayed Purges'). Theo thông báo ban đầu của Cloudflare, khách hàng không thuộc diện FedRAMP không bị ảnh hưởng bởi sự cố này ('Non-fedramp customers are not impacted'). Do đó, không có đủ căn cứ để kết luận sự cố trên làm gián đoạn website của CLB. Dữ liệu từ status page của Cloudflare."
- Điểm ở lần chạy 1: 75/100 (Factuality: 35/40, Completeness: 20/25, Evidence: 10/20, Boundaries: 10/15).

## 2. Bằng chứng trong trace

Đoạn trace của case E2 trong `eval_before.json`:
```json
[
  {
    "event": "tool",
    "step": 1,
    "name": "get_status",
    "arguments": {
      "provider": "cloudflare",
      "component": "Cache Purge"
    },
    "result": {
      "provider": "cloudflare",
      "captured_at": "2026-11-09T03:50:47.638054+00:00",
      "overall": {
        "indicator": "major",
        "description": "Major Service Outage"
      },
      "components": [
        {
          "name": "Cache Purge",
          "status": "degraded_performance"
        }
      ],
      "incidents": [
        {
          "id": "wp49bp8bqo6c",
          "name": "Fedramp Customers have Delayed Purges",
          "status": "identified",
          "updated_at": "2026-11-04T19:35:47.962Z"
        }
      ],
      "matching_incidents_in_last_50": 1,
      "source": "https://www.cloudflarestatus.com"
    }
  },
  {
    "event": "tool",
    "step": 2,
    "name": "get_incident_details",
    "arguments": {
      "provider": "cloudflare",
      "incident_id": "wp49bp8bqo6c"
    },
    "result": {
      "ok": true,
      "provider": "cloudflare",
      "id": "wp49bp8bqo6c",
      "name": "Fedramp Customers have Delayed Purges",
      "status": "identified",
      "updates": [
        {
          "status": "identified",
          "created_at": "2026-11-04T19:35:47.962Z",
          "body": "Non-fedramp customers are not impacted by this incident.",
          "truncated": false
        }
      ],
      "updates_total": 1,
      "url": "https://www.cloudflarestatus.com/incidents/wp49bp8bqo6c"
    }
  }
]
```

**Nguyên nhân:**
Mặc dù agent đã gọi đúng công cụ `get_incident_details` và nhận được URL chi tiết của sự cố (`https://www.cloudflarestatus.com/incidents/wp49bp8bqo6c`), trong câu trả lời cuối cùng ở lần chạy 1, agent chỉ trích dẫn chung chung "Dữ liệu từ status page của Cloudflare" mà không đính kèm đường link URL cụ thể của incident. Do đó, giám khảo (judge) đã trừ 10 điểm ở tiêu chí Evidence (chỉ cho 10/20) vì thiếu nguồn kiểm chứng trực tiếp.

## 3. Thay đổi đã thực hiện

- **Sửa gì, ở đâu:**
  Triển khai cơ chế kiểm tra và yêu cầu bổ sung nguồn (Source Validation Gate) trực tiếp trong vòng lặp `run_agent` tại `agent.py`:
  1. Trong quá trình lặp, khi tool `get_incident_details` trả về kết quả thành công (`result.get('ok') is True`), tự động thu thập và lưu trữ thuộc tính `url` vào danh sách `observed_urls`.
  2. Khi model hoàn thành và chuẩn bị đưa ra câu trả lời cuối cùng (`finish == 'stop'` và không còn tool call), kiểm tra xem tất cả URL trong `observed_urls` đã xuất hiện trong nội dung câu trả lời `answer` hay chưa.
  3. Nếu phát hiện thiếu URL nguồn và vẫn còn ngân sách bước (`step < max_steps`), ghi sự kiện `{'event': 'validation_missing_source', 'step': step, 'missing_urls': missing_urls}` vào `trace`, đồng thời gửi phản hồi yêu cầu model bổ sung URL nguồn trực tiếp vào câu trả lời.
  4. Nếu đã chạm ngưỡng `max_steps` mà câu trả lời vẫn chưa có nguồn, agent dừng và trả về trạng thái `max_steps` thay vì báo `completed`.
- **Vì sao chọn thay đổi này:**
  - Đáp ứng chính xác yêu cầu tại Bài 4 và vượt qua toàn bộ các unit test trong `tests/test_validation.py`.
  - Ngăn ngừa hiện tượng cung cấp thông tin không có nguồn kiểm chứng (hallucination / ungrounded claims), nâng cao tính tin cậy và minh bạch cho agent khi điều tra sự cố.

## 4. So sánh hai lần chạy

| Chỉ số | `eval_before.json` | `eval_after.json` |
|---|---|---|
| Điểm tổng | 98.27 / 100 | 99.41 / 100 |
| Điểm case đã chọn (E2) | 75 / 100 | 100 / 100 |
| Số model calls | 52 | 53 |
| Token, chi phí nếu có | 24,120 tokens | 24,720 tokens |

- **Lỗi đã giải quyết:**
  - Ở case E2, sự kiện `validation_missing_source` được kích hoạt ở bước 3, yêu cầu model bổ sung URL. Tại bước 4, model đã đưa link `https://www.cloudflarestatus.com/incidents/wp49bp8bqo6c` vào câu trả lời hoàn chỉnh.
  - Điểm của case E2 tăng từ 75 lên 100 tuyệt đối (Evidence đạt trọn vẹn 20/20).
  - Điểm tổng trung bình của toàn bộ 22 case tăng từ 98.27 lên 99.41.
- **Lỗi còn lại hoặc mới xuất hiện:**
  - Chi phí gọi model tăng thêm 1 lượt gọi (từ 52 lên 53) và tiêu tốn thêm 600 tokens cho lượt revision của case E2.
  - Cần lưu ý cấu hình `max_steps` đủ lớn nếu người dùng yêu cầu điều tra nhiều sự cố phức tạp đồng thời.

## 5. Giới hạn

1. **Giới hạn của LLM Judge:** LLM Judge là phương pháp đánh giá định tính xấp xỉ. Điểm số có thể biến động nhẹ theo nhiệt độ (temperature) của model hoặc thiên lệch (bias) nếu cùng dùng một họ model để làm agent và judge. Do đó, điểm judge luôn cần được kết hợp với các bài kiểm tra cơ học bằng code (mechanical checks: kiểm tra tool bắt buộc, cấm marker injection, kiểm tra số bước).
2. **Giới hạn của Snapshot:** Dữ liệu snapshot cố định tại ngày 09/11/2026 (core) và 10/11/2026 (bonus) và chỉ lưu trữ tối đa 50 incident gần nhất. Do đó, snapshot không phản ánh trạng thái live thực tế và không thể cung cấp danh sách đầy đủ cho các khoảng thời gian dài trong quá khứ.
3. **Giới hạn của việc chạy một lần mỗi phiên bản:** Một lần chạy thử nghiệm duy nhất không phản ánh trọn vẹn phương sai thống kê (variance) của mô hình. Trong triển khai thực tế, cần thực hiện nhiều lần chạy độc lập (multi-run evaluation) trên các tập dữ liệu mở rộng để đánh giá độ ổn định và khoảng tin cậy của agent.
