# Phân tích yêu cầu — vai Consumer (Analytics)

- Cặp đàm phán: pair-06 (IoT Analytics)
- Product: IoT Analytics (Consumer)
- Consumer service: Analytics (aggregate, dashboard)
- Provider service: IoT Ingestion (telemetry producer)
- Người viết: Nhóm 06
- Ngày: 2026-05-19

---

## 1. Resource Consumer cần nhận / tạo

| Resource | Consumer dùng để làm gì? | Field bắt buộc với Consumer | Field có thể tùy chọn |
|---|---|---|---|
| `telemetry.event` | Nhận event telemetry để aggregate theo giờ/ngày, lưu raw cho replay | `deviceId`, `timestamp`, `metrics[]` (name,value), `eventId` | `zoneId`, `location`, `metadata` |
| `device.status` | Cập nhật trạng thái thiết bị (online/offline) để điều chỉnh aggregate/hiển thị dashboard) | `deviceId`, `status`, `timestamp` | `reason`, `source` |
| `aggregated.metric` | Bản ghi aggregate (hourly/daily) phục vụ dashboard và API query | `key` (deviceId|zoneId|interval), `intervalStart`, `interval`, `metrics` | `count`, `min`, `max`, `median` |

---

## 2. Cách nhận / API Consumer cần hỗ trợ

1. Nhận sự kiện (async):
	 - Consumer subscribe queue/topic `telemetry.ingested` và `device.status.changed` 
	 - Kỳ vọng message là JSON theo schema mẫu ở dưới.

2. API query (sync) cho dashboard: Consumer expose endpoint để frontend lấy aggregate:

| Method | Path | Khi nào gọi | Kỳ vọng response |
|---|---:|---|---|
| GET | `/analytics/aggregates` | Frontend yêu cầu hiển thị dashboard | 200 + list aggregated.metric theo filter |
| GET | `/analytics/aggregates?deviceId={id}&from={ts}&to={ts}&interval=hour` | Lấy dữ liệu biểu đồ | 200 + time-series |

Ví dụ response (tóm tắt):
{
	"key": "device-123|2026-05-19T10:00:00Z|hour",
	"intervalStart": "2026-05-19T10:00:00Z",
	"interval": "hour",
	"metrics": { "temperature.avg": 23.4, "temperature.count": 120 }
}

---

## 3. Error case Consumer cần xử lý (ít nhất 6 case)

| Case | Consumer hiểu là gì? | Consumer sẽ xử lý thế nào? |
|---|---|---|
| Invalid schema / missing `deviceId` | Event không hợp lệ | Bỏ event, log đầy đủ, gửi metric lỗi, nếu có `eventId` gửi cảnh báo cho Provider |
| Invalid timestamp / out-of-range | Dữ liệu thời gian không dùng được cho aggregate | Bỏ hoặc lưu vào bucket lỗi để điều tra; không ảnh hưởng aggregate hiện tại |
| Duplicate event (retry) | Có thể gây double-count | Consumer phải idempotent theo `eventId` (dedupe store/requests) |
| Out-of-order event | Event cũ cập nhật aggregate đã finalize | Nếu event trong window chưa finalize thì apply, nếu đã finalize thì ghi vào correction stream hoặc flag để re-aggregate |
| Downstream DB write failure | Không lưu được aggregate | Retry theo backoff, nếu liên tục fail đẩy vào DLQ và cảnh báo vận hành |
| Schema version mismatch | Provider gửi version khác | Từ chối hoặc map theo migration rules; đặt câu hỏi với Provider và versioning strategy |

---

## 4. Giả định bổ sung

- Provider gửi `eventId` duy nhất cho mỗi event.
- Timestamps là UTC ISO-8601 và có độ chính xác tối thiểu giây.
- Consumer có quyền đọc topic/queue và có hệ thống DLQ để lưu event lỗi.
- Aggregate theo `deviceId` mặc định; có thể mở rộng theo `zoneId`.

---

## 5. Câu hỏi cho Provider

1. Provider sẽ gắn `eventId` duy nhất và `schemaVersion` trên mỗi message chứ?
2. Có guarantee ordering per-partition (ví dụ Kafka partition key = deviceId)?
3. Với giá trị metric, unit sẽ được chuẩn hóa hay kèm theo trường `unit` trong event?
4. Chính sách retry và visibility timeout trên queue là bao lâu (ảnh hưởng idempotency)?
5. Khi thiết bị offline, Provider có gửi `device.status.changed` liên tục hay chỉ gửi on/off một lần?

---

## 6. Rủi ro tích hợp & đề xuất

| Rủi ro | Tác động | Đề xuất xử lý |
|---|---|---|
| Provider đổi kiểu dữ liệu cho metric (string → number) | Consumer parse lỗi, aggregate sai | Thống nhất schema + contract, validate schema ở ingress, reject/alert nếu mismatch |
| Không có `eventId` hoặc không global-unique | Khó dedupe, dẫn tới double-count | Yêu cầu `eventId` bắt buộc; nếu không có, dùng hash(deviceId+timestamp+payload) tạm thời |
| Retry nhiều lần dẫn đến duplicate | Over-count hoặc DB bùng nổ | Consumer idempotent theo `eventId`, dùng dedupe cache TTL phù hợp |
| Event out-of-order ảnh hưởng aggregate final | Aggregate bị thiếu/cộng sai | Thiết kế cửa sổ thời gian chậm (late-arrival window) và correction job |

---

## 7. Ví dụ schema mẫu

`telemetry.ingested` sample:

{
	"eventId": "evt-0001",
	"schemaVersion": "1.0",
	"deviceId": "device-123",
	"timestamp": "2026-05-19T10:12:03Z",
	"zoneId": "zone-7",
	"metrics": [ { "name": "temperature", "value": 23.4, "unit": "C" }, { "name": "humidity", "value": 56 } ],
	"metadata": { "firmware": "v1.2" }
}

`device.status.changed` sample:

{
	"eventId": "evt-0002",
	"schemaVersion": "1.0",
	"deviceId": "device-123",
	"status": "offline",
	"timestamp": "2026-05-19T10:13:00Z",
	"reason": "power-loss"
}

`aggregated.metric` sample:

{
	"key": "device-123|2026-05-19T10:00:00Z|hour",
	"intervalStart": "2026-05-19T10:00:00Z",
	"interval": "hour",
	"metrics": { "temperature.avg": 23.4, "temperature.count": 120 }
}
