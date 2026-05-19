# User Story — IoT Ingestion → Analytics

## 1. Cơ chế

**Queue async** (topic `telemetry.ingested`, `device.status.changed`)

## 2. Bối cảnh

IoT Ingestion gửi telemetry realtime từ thiết bị. Analytics tiêu thụ các event này để:
- Aggregate theo giờ/ngày
- Cung cấp dữ liệu time-series cho dashboard
- Phát hiện bất thường (anomaly) đơn giản

## 3. Nhu cầu của Consumer (Analytics)

- Nhận từng event telemetry, lưu raw và cập nhật aggregate theo `deviceId` (mặc định) hoặc `zoneId`.
- Đảm bảo idempotency: cùng `eventId` chỉ được xử lý 1 lần.
- Hỗ trợ late-arrival window 5 phút (cấu hình) để xử lý event trễ.
- Có DLQ cho event không parse được hoặc lỗi persist.

## 4. Event / Schema trọng tâm

- `telemetry.ingested` (sử dụng để aggregate)
- `device.status.changed` (cập nhật trạng thái thiết bị)

Ví dụ `telemetry.ingested`:

{
	"eventId": "evt-0001",
	"schemaVersion": "1.0",
	"deviceId": "device-123",
	"timestamp": "2026-05-19T10:12:03Z",
	"zoneId": "zone-7",
	"metrics": [ { "name": "temperature", "value": 23.4, "unit": "C" } ]
}

Ví dụ `device.status.changed`:

{
	"eventId": "evt-0002",
	"schemaVersion": "1.0",
	"deviceId": "device-123",
	"status": "offline",
	"timestamp": "2026-05-19T10:13:00Z"
}

## 5. Acceptance Criteria (AC)

1. Consumer subscribe được topic và xử lý `telemetry.ingested` thành aggregate hourly và daily.
2. Với cùng `eventId`, Consumer chỉ apply một lần (idempotency proven).
3. Event có schema không hợp lệ được đẩy vào DLQ cùng log chi tiết.
4. Aggregate API `GET /analytics/aggregates` trả dữ liệu hợp lệ cho khoảng thời gian yêu cầu.
5. Có ghi chép trong `negotiation-log.md` về `eventId`, `schemaVersion`, và retry/ordering guarantees.

## 6. Edge cases & lỗi cần xử lý

- Thiếu `deviceId` hoặc `timestamp` → reject event → DLQ.
- Duplicate event / Retry → dedupe theo `eventId`.
- Out-of-order events → nếu trong window xử lý, áp dụng; nếu ngoài window, gửi correction hoặc flag.
- Downstream DB failure → retry/backoff rồi DLQ nếu không thành công.

## 7. Câu hỏi gợi ý cho phiên đàm phán

1. Provider có đảm bảo `eventId` unique không? Có format bắt buộc không?
2. Ordering guarantee (per device partition) là gì? Có cần partition key là `deviceId`?
3. Unit cho mỗi metric có được chuẩn hóa không (ví dụ temperature luôn là Celsius)?
4. Provider có gửi schemaVersion kèm theo message không?
5. Policy cho event trễ (late arrival) và retry/backoff là gì?

## 8. Ghi chú phạm vi Lab 02

Trong Lab 02 cần chuẩn bị:
- Mô tả event schema tối thiểu trong `docs/event-contract-template.md` hoặc `negotiation-log.md`.
- Triển khai consumer đơn giản (simulated) đủ để demo aggregate hourly và truy vấn API.
- Không yêu cầu bắt buộc viết AsyncAPI đầy đủ, nhưng nên ghi rõ các trường bắt buộc, `eventId`, `schemaVersion` và DLQ policy.
