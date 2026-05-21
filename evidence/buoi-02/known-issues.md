# Known Issues — Lab 02

Ghi các lỗi còn tồn tại nếu chưa xử lý xong.

| Lỗi | Ảnh hưởng | Trạng thái | Cách xử lý / Ghi chú | Người phụ trách |
|---|---|---|---|---|
| Duplicate event (event trùng do retry) | Gây double-count trong aggregates | Đã nêu giải pháp (design) | Dedupe theo `eventId` / consumer idempotency — xem [docs/event-contract-template.md](../docs/event-contract-template.md) |  |
| Out-of-order event (event đến muộn/không theo thứ tự) | Aggregate window có thể sai | Đã nêu giải pháp (design) | Thiết kế cửa sổ chờ (late-arrival window), hỗ trợ re-processing — xem [docs/event-contract-template.md](../docs/event-contract-template.md) |  |
| Missing/invalid schema (thiếu trường / kiểu sai) | Consumer không parse / aggregate được | Đã nêu giải pháp (design) | Validate tại ingress; gửi invalid vào DLQ + alert; báo Provider nếu có `eventId` — xem [docs/event-contract-template.md](../docs/event-contract-template.md) |  |
| Retry policy gây duplicate processing | Consumer có thể xử lý trùng lặp nhiều lần | Đã nêu giải pháp (design) | At-least-once + broker backoff; consumer phải idempotent — xem [docs/event-contract-template.md](../docs/event-contract-template.md) |  |
| Dead-letter queue (DLQ) chưa cụ thể | Có thể làm chậm điều tra lỗi nếu không tách DLQ | Đã nêu giải pháp (design) | Định nghĩa DLQ format, metadata lỗi, retention, alerting — xem [docs/event-contract-template.md](../docs/event-contract-template.md) |  |
| Ordering concern (yêu cầu ordering theo device) | Trạng thái/nghiệp vụ sai nếu ordering bị phá | Đã nêu giải pháp (design) | Partition by `deviceId`; vẫn cần idempotency và xử lý late-arrival — xem [docs/event-contract-template.md](../docs/event-contract-template.md) |  |
| Retention của raw events | Ảnh hưởng chi phí lưu trữ và khả năng replay | Đã nêu giải pháp (design) | Quy định retention (ví dụ raw 14 ngày, DLQ 30 ngày) và lưu aggregates dài hạn — xem [docs/event-contract-template.md](../docs/event-contract-template.md) |  |
| Idempotency handling (dùng `eventId`) | Tránh double-count và inconsistent state | Đã nêu giải pháp (design) | `eventId` required; dedupe store (Redis/DB) với TTL phù hợp — xem [docs/event-contract-template.md](../docs/event-contract-template.md) |  |
| Các issue trong `negotiation-log.md` | Chưa được điền/đánh dấu | Chưa giải quyết | `negotiation-log.md` chứa template Issue #1..#6 — cần cặp đàm phán điền chi tiết và ký chốt — xem [negotiation-log.md](../../negotiation-log.md) |  |
| Một số trường trong `openapi.yaml` (ví dụ `resolvedAt`) | Hiện giá trị `null` trong mẫu | Chưa giải quyết / cần điền | Cập nhật `openapi.yaml` theo thỏa thuận cặp (path/schema/example) — xem [openapi.yaml](../../openapi.yaml) |  |

> Ghi chú: các mục "Đã nêu giải pháp (design)" nghĩa là giải pháp kĩ thuật đã được chỉ rõ trong `docs/event-contract-template.md` nhưng cần chuyển thành quyết định trong `negotiation-log.md` và/hoặc cập nhật chi tiết vào `openapi.yaml` hoặc AsyncAPI cho Lab 03 để hoàn thiện.
