# Biên bản đàm phán hợp đồng API — Pair-06 (IoT Analytics)

- Cặp đàm phán: pair-06
- Product: IoT Analytics (Consumer) / IoT Ingestion (Provider)
- Provider: iot-ingest
- Consumer: analytics
- Phiên: v1.0
- Ngày: 2026-05-19

---

## Issue #1 — Event naming

- Raised by: Consumer
- Endpoint: N/A (topic naming)
- Concern: Tên event/topic cần rõ ràng và thống nhất giữa Producer và Consumer.
- Proposal: Dùng pattern `<domain>.<resource>.<action>` và cụ thể: `sensors.telemetry.ingested`, `devices.status.changed`.
- Resolution: Accepted
- Rationale: Tên rõ ràng, hỗ trợ discovery và phép lọc theo domain/resource.
- Impact: Consumer sẽ subscribe các topic này; Provider cần publish đúng tên.

---

## Issue #2 — Retry policy & idempotency

- Raised by: Consumer
- Endpoint: Broker-level behavior / ingest
- Concern: Retry từ broker có thể gây duplicate processing.
- Proposal: Broker cung cấp at-least-once semantics; producer simple; consumer phải idempotent. `eventId` bắt buộc trên mỗi message.
- Resolution: Accepted
- Rationale: Giữ cho producer đơn giản, chuyển trách nhiệm đảm bảo không double-count cho consumer.
- Impact: Cần implement dedupe store (ví dụ Redis) theo `eventId` với TTL >= retention window.

---

## Issue #3 — Ordering guarantee

- Raised by: Provider / Consumer
- Endpoint: publish to broker
- Concern: Nhiều nghiệp vụ yêu cầu ordering per-device để duy trì trạng thái chính xác.
- Proposal: Producer đặt partition key = `deviceId` khi publish để đảm bảo ordering per-partition; consumer vẫn hỗ trợ late-arrival.
- Resolution: Accepted
- Rationale: Partitioning theo `deviceId` là cách thông dụng để giữ ordering mà không tắt tính scale-out.
- Impact: Producer cần set partition key; consumer cần xử lý out-of-order và có cửa sổ chờ (late-arrival window).

---

## Issue #4 — Payload schema & required fields

- Raised by: Both
- Endpoint: messages (telemetry.ingested, device.status.changed)
- Concern: Thiếu sự rõ ràng về trường bắt buộc (ví dụ `eventId`, `schemaVersion`, `deviceId`, `timestamp`, `metrics`).
- Proposal: Chuẩn schema như trong `openapi.yaml` (TelemtryIngest / DeviceStatusIngest) với các trường bắt buộc: `eventId`, `schemaVersion`, `occurredAt`, `data.deviceId`, `data.timestamp`, `data.metrics[]` (cho telemetry).
- Resolution: Accepted
- Rationale: Ràng buộc rõ ràng giúp cả hai bên validate và giảm drift schema.
- Impact: Provider phải tạo `eventId` và gán `schemaVersion`; consumer validate và đẩy invalid vào DLQ.

---

## Issue #5 — Versioning strategy

- Raised by: Both
- Endpoint: messages
- Concern: Khi schema thay đổi (breaking vs non-breaking) cần có cách xử lý.
- Proposal: Sử dụng `schemaVersion` field (MAJOR.MINOR). MAJOR bump = incompatible → producer/consumer thỏa thuận; consumer có thể từ chối MAJOR unknown và gửi vào DLQ.
- Resolution: Accepted
- Rationale: Rõ ràng và nhẹ, phù hợp Lab02 scope; AsyncAPI/Lab03 sẽ mở rộng quy tắc.
- Impact: Thêm monitoring/alert khi gặp `schemaVersion` lạ; cập nhật tài liệu khi bump MAJOR.

---

## Issue #6 — Retention & DLQ policy

- Raised by: Consumer
- Endpoint: broker/config
- Concern: Cần retention đủ để replay và điều tra, và DLQ cho event lỗi.
- Proposal: Raw events retention = 14 days (configurable); DLQ retention = 30 days; invalid events and persistent processing errors -> DLQ with error metadata and alerting.
- Resolution: Accepted
- Rationale: Cân bằng chi phí lưu trữ và khả năng reprocess.
- Impact: Ops need to configure broker retention and alerting; consumer uses TTL for dedupe store accordingly.

---

## Issue #7 — Analytics query API

- Raised by: Consumer
- Endpoint: `/analytics/aggregates` (sync)
- Concern: Frontend cần API để lấy aggregated metrics (hour/day).
- Proposal: Expose `GET /analytics/aggregates` với params `deviceId|zoneId|from|to|interval` và trả `AggregatedMetricPage`.
- Resolution: Accepted
- Rationale: Cần sync API để hiển thị dashboard và kiểm thử khi demo Lab02.
- Impact: Implement endpoint in consumer service; update `openapi.yaml` accordingly.

---

## Issue #8 — Ingest HTTP endpoints (implementation note)

- Raised by: Provider
- Endpoint: `/ingest/v1/telemetry`, `/ingest/v1/device-status`
- Concern: Producer cần 1 HTTP ingress để nhận payload từ gateway before publishing to broker.
- Proposal: Provide internal ingest endpoints (POST) that validate, enrich and publish to topic.
- Resolution: Accepted
- Rationale: Simplifies gateway integration and centralizes validation.
- Impact: Provider implements endpoints and documents behavior in `openapi.yaml`.

---

# Chốt hợp đồng v1.0

Provider sign-off:  
Consumer sign-off:  
Witness (GV/TA):    
Date: 2026-05-19

---

## Ghi chú warning nếu Spectral còn cảnh báo

| Warning | Lý do chấp nhận tạm thời | Kế hoạch sửa |
|---|---|---|
|  |  |  |
