# Event Contract sơ bộ — IoT Telemetry (pair-06)

> Mục tiêu: mô tả event contract sơ bộ để chuẩn bị cho Lab 03 (AsyncAPI). Tài liệu này tập trung vào `telemetry.ingested` và `device.status.changed` — hai event chính Producer (IoT Ingestion) gửi đến Consumer (Analytics).

## 1. Thông tin dependency

- Dependency số: pair-06
- Producer: IoT Ingestion (telemetry-ingest)
- Consumer: IoT Analytics (analytics)
- Cơ chế: Queue / Topic async (ví dụ: Kafka topic hoặc cloud pub/sub)
- Event/topic dự kiến: `telemetry.ingested`, `device.status.changed`
- Người ghi: Nhóm 06
- Ngày: 2026-05-19

## 2. Mục đích nghiệp vụ

- `telemetry.ingested`: Gửi các mẫu telemetry từ thiết bị (sensor readings) để `Analytics` tổng hợp, lưu raw cho replay và hiển thị dashboard.
- `device.status.changed`: Báo trạng thái thiết bị (online/offline) để điều chỉnh aggregate và trạng thái hiển thị.

## 3. Tên event / topic

| Mục | Giá trị |
|---|---|
| Tên event (chính) | `telemetry.ingested` |
| Topic / queue (gợi ý) | `sensors.telemetry.ingested` |
| Producer | `iot-ingest` |
| Consumer | `analytics` |

| Mục | Giá trị |
|---|---|
| Tên event (trạng thái) | `device.status.changed` |
| Topic / queue (gợi ý) | `devices.status.changed` |
| Producer | `iot-ingest` |
| Consumer | `analytics` |

## 4. Thiết kế payload schema sơ bộ

— Metadata chung (bắt buộc cho mọi event):

```json
{
  "eventId": "uuid",            // bắt buộc, chuỗi, UUID (RFC4122) hoặc định danh duy nhất do producer tạo
  "occurredAt": "2026-05-19T10:12:03Z", // bắt buộc, ISO-8601 UTC
  "schemaVersion": "1.0",      // bắt buộc, chuỗi version (MAJOR.MINOR)
  "source": "iot-ingest",      // bắt buộc, tên service producer
  "correlationId": "uuid|null",// tùy chọn nhưng khuyến nghị, dùng để trace luồng nghiệp vụ
  "requestId": "uuid|null"     // tùy chọn, ánh xạ tới request gốc nếu có
}
```

1) `telemetry.ingested` - payload `data` schema

```json
{
  "eventId": "evt-0001",
  "schemaVersion": "1.0",
  "occurredAt": "2026-05-19T10:12:03Z",
  "source": "iot-ingest",
  "correlationId": "corr-123",
  "data": {
    "deviceId": "device-123",            // bắt buộc, chuỗi
    "timestamp": "2026-05-19T10:12:00Z", // bắt buộc, ISO-8601 UTC (có thể khác nhẹ so với occurredAt)
    "zoneId": "zone-7",                  // tùy chọn, chuỗi
    "metrics": [                           // bắt buộc, mảng không rỗng
      { "name": "temperature", "value": 23.4, "unit": "C" },
      { "name": "humidity", "value": 56 }
    ],
    "metadata": { "firmware": "v1.2" } // tùy chọn: đối tượng chứa thông tin không quan trọng cho processing
  }
}
```

Kiểu trường & ràng buộc (telemetry.ingested):
- `eventId`: chuỗi, duy nhất cho mỗi event; khuyến nghị UUID do producer tạo.
- `schemaVersion`: chuỗi dạng `MAJOR.MINOR` (ví dụ `1.0`).
- `occurredAt`, `data.timestamp`: ISO-8601 UTC; `data.timestamp` là thời điểm đọc trên thiết bị.
- `deviceId`: chuỗi không rỗng, định danh thiết bị chuẩn.
- `metrics`: mảng các đối tượng { `name`: string, `value`: number, `unit`?: string } — `value` phải là số; `name` hạn chế ký tự chữ-số, dấu chấm hoặc gạch dưới.
- `zoneId`: khóa phân vùng tùy chọn; dùng cho grouping khi aggregate.

Quy tắc xác thực:
- Từ chối (reject) message nếu thiếu `eventId`, `schemaVersion`, `occurredAt` hoặc `data.deviceId`.
- `metrics` phải không rỗng và mỗi phần tử phải có `name` và `value` là số.

2) `device.status.changed` - payload

```json
{
  "eventId": "evt-0002",
  "schemaVersion": "1.0",
  "occurredAt": "2026-05-19T10:13:00Z",
  "source": "iot-ingest",
  "data": {
    "deviceId": "device-123",
    "status": "offline",    // enum: ["online","offline","unknown"]
    "timestamp": "2026-05-19T10:13:00Z",
    "reason": "power-loss"  // tùy chọn
  }
}
```

## 5. Error, edge-cases và delivery concerns

Các case dưới đây nêu vấn đề, ảnh hưởng và hướng xử lý sơ bộ.


- Duplicate event
  - Vấn đề: Producer gửi trùng event (do retry) → consumer có thể tính double-count.
  - Ảnh hưởng: Tồn tại over-count, aggregates không nhất quán.
  - Xử lý: Consumer thực hiện dedupe theo `eventId` (idempotent store). Producer giữ semantics at-least-once; consumer cần thiết kế store dedupe với TTL cấu hình được.

- Out-of-order event
  - Vấn đề: Event đến không theo thứ tự thời gian (timestamp thiết bị khác với thời gian ingest).
  - Ảnh hưởng: Aggregate window có thể tính sai nếu event tới muộn.
  - Xử lý: Thiết kế cửa sổ chờ (late-arrival window) ví dụ 5–15 phút; hỗ trợ re-processing hoặc correction stream cho events cũ.

- Missing/invalid schema
  - Vấn đề: Thiếu trường bắt buộc hoặc kiểu dữ liệu không đúng.
  - Ảnh hưởng: Consumer không parse/aggregate được.
  - Xử lý: Validate tại ingress; gửi các event invalid vào DLQ kèm lý do, alert cho vận hành và metric; thông báo Provider nếu có `eventId`.

- Retry policy
  - Vấn đề: Khi consumer xử lý lỗi, broker/proxy có thể retry gửi lại message.
  - Ảnh hưởng: Gây duplicate processing nếu consumer không idempotent.
  - Xử lý: Định nghĩa retry/backoff ở tầng broker; consumer phải idempotent và chịu được at-least-once delivery. Ghi rõ số lần retry và backoff trong Lab03.

- Dead-letter queue (DLQ)
  - Vấn đề: Các event liên tục lỗi validation hoặc processing.
  - Ảnh hưởng: Có thể chặn pipeline nếu không tách ra.
  - Xử lý: Chuyển vào DLQ với metadata lỗi, giữ retention cho điều tra, bật cảnh báo tự động khi DLQ tăng trưởng.

- Ordering concern
  - Vấn đề: Nếu cần đảm bảo ordering theo thiết bị để đúng nghiệp vụ.
  - Ảnh hưởng: Aggregate và trạng thái có thể sai nếu ordering bị phá.
  - Xử lý: Sử dụng partition key = `deviceId` (với Kafka) để đảm bảo ordering per-partition; đồng thời vẫn xử lý retry và idempotency.

- Retention concern
  - Vấn đề: Thời gian lưu raw events để replay là bao lâu.
  - Ảnh hưởng: Chi phí lưu trữ và khả năng reprocess lịch sử.
  - Xử lý: Quy định retention (ví dụ raw events 7–30 ngày; DLQ lâu hơn); persist aggregates vào kho lưu trữ dài hạn.

- Idempotency handling
  - Vấn đề: Event trùng không được làm sai lệch aggregate cuối cùng.
  - Ảnh hưởng: Ảnh hưởng đến độ chính xác dữ liệu.
  - Xử lý: Consumer lưu index dedupe (ví dụ Redis/DB) keyed theo `eventId` với TTL lớn hơn cửa sổ retry tối đa; lưu offsets đã xử lý khi cần.

Each of the above items will be recorded in the negotiation log (phần sau) to carry forward to Lab03.

## 6. Negotiation / Design decision log

| Issue | Decision | Rationale | Impact |
|---|---|---|---|
| Event naming | Use `sensors.telemetry.ingested`, `devices.status.changed` | Clear domain.subject.action pattern; aligns with AsyncAPI/topic conventions | Easy discovery, consistent grouping |
| Retry policy | At-least-once delivery; broker retry with exponential backoff; consumer idempotent | Simpler producer; consumer handles duplicates | Consumer must implement dedupe and retry-safe writes |
| Ordering | Partition by `deviceId` for per-device ordering where possible | Many analytics require per-device ordering for state transitions | Requires producer to set partition key; consumer still handles late-arrival |
| Payload structure | Flatten `data` with `deviceId`, `timestamp`, `metrics[]`; `metadata` optional | Keeps top-level metadata separate and makes `data` focused | Easier validation and versioning |
| Versioning | `schemaVersion` field; MAJOR bump for incompatible changes | Allows minor/patch extension while keeping compatibility rules explicit | Consumers must check `schemaVersion` and handle or reject unknown MAJOR |
| Retention | Raw events: 14 days (configurable); DLQ: 30 days | Balance cost vs ability to reprocess | Define process for snapshots and long-term aggregates storage |
| Idempotency | `eventId` required; consumer dedupe cache TTL = retention window + safety margin | Needed to avoid double-count | Operational cost for dedupe store; must size accordingly |

## 7. Chuẩn bị cho Lab 03 (AsyncAPI-ready)

- Topic naming convention: `<domain>.<resource>.<action>` (e.g., `sensors.telemetry.ingested`). Keep ASCII, lowercase, dots as separators.
- Schema organization: separate `components/schemas` per event type; keep `metadata` shared component.
- Versioning strategy: include `schemaVersion` in message; AsyncAPI `message.version` mirrors this. MAJOR changes → new message name or new major version.
- Backward compatibility: support additive fields (optional) in minor versions; require consumers to ignore unknown fields.
- Extensibility: `metadata` object for non-critical info; allow `metrics` to contain named metrics to avoid schema churn.

## 8. Issues chuyển sang Lab 03

1. Xác định broker-level retry/backoff và visibility timeout cụ thể.
2. Cụ thể hóa DLQ format và retention bằng config.
3. Viết AsyncAPI skeleton mapping các event ở trên và các component schema.
