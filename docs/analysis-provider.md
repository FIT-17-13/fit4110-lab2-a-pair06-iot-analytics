# Phân tích yêu cầu — vai Provider (IoT Ingestion)

- Cặp đàm phán: pair-06
- Product: IoT Ingestion (Producer)
- Provider service: `iot-ingest`
- Consumer service: `analytics`
- Người viết: Nhóm 06
- Ngày: 2026-05-19

---

## 1. Resource chính (góc nhìn Provider)

| Resource | Mô tả | Thuộc tính bắt buộc | Thuộc tính tùy chọn |
|---|---|---|---|
| `telemetry.event` | Event chứa các measurement từ device, gửi tới topic `sensors.telemetry.ingested` | `eventId`, `schemaVersion`, `occurredAt`, `data.deviceId`, `data.timestamp`, `data.metrics[]` | `data.zoneId`, `data.metadata` |
| `device.status` | Event trạng thái thiết bị, gửi tới `devices.status.changed` | `eventId`, `schemaVersion`, `occurredAt`, `data.deviceId`, `data.status`, `data.timestamp` | `data.reason`, `data.source` |

---

## 2. Hành vi Producer

- Producer nhận telemetry từ device hoặc gateway.
- Batch vs stream: Producer có thể gom nhiều metrics vào một event `telemetry.ingested` cho mỗi cửa sổ đọc (tùy triển khai), nhưng cần ghi rõ trong `schemaVersion`.
- Gợi ý API HTTP ingest (nội bộ): `POST /ingest/v1/telemetry` — nhận JSON từ gateway, xác thực, enrich metadata, rồi publish lên topic.
- Với trạng thái thiết bị: `POST /ingest/v1/device-status` hoặc event nội bộ do tầng kết nối tạo ra.

## 3. Các trường hợp lỗi (Provider)

| Trường hợp | Tình huống | Hành xử của Provider |
|---|---|---|
| Payload thiết bị không hợp lệ | Thiếu `deviceId` hoặc `metrics` bị malformed | Từ chối ingest trả 400 với Problem details; KHÔNG publish event; tăng metric lỗi cho monitoring |
| Xác thực/ủy quyền thất bại | Thiếu hoặc token không hợp lệ từ gateway | Trả 401/403; không publish |
| Publish xuống broker thất bại | Broker không sẵn sàng/timeout | Thử lại publish theo backoff; nếu vẫn fail thì persist vào fallback cục bộ hoặc DLQ để replay sau |
| Payload quá lớn | Kích thước payload vượt giới hạn | Từ chối với 413 hoặc tách payload thành nhiều events sau khi thương lượng |
| Gateway gửi trùng | Gateway gửi lại cùng reading | Có thể detect duplicates ở ingress (tùy chọn) bằng khóa trùng; vẫn phải đảm bảo `eventId` publish ra là duy nhất |

## 4. Giả định bổ sung


- Gateways / thiết bị phải kèm `deviceId` trong mỗi message.
- Timestamp từ thiết bị có thể bị lệch; `occurredAt` được producer gán tại thời điểm ingest, `data.timestamp` là thời điểm đọc trên thiết bị.
- Producer sẽ tạo `eventId` (UUID) cho mỗi event publish.
- Producer đặt partition key = `deviceId` khi publish lên broker để hỗ trợ ordering theo thiết bị.

---

## 5. Câu hỏi cho Consumer (Analytics)

1. Có yêu cầu ordering nghiêm ngặt theo `deviceId` không, hay chỉ cần chấp nhận out-of-order trong cửa sổ aggregate?
2. Kỳ vọng retention raw events trong broker là bao lâu để hỗ trợ replay?
3. Cần chuẩn hóa unit (ví dụ temperature về C) hay provider gửi unit, consumer tự map?
4. Với trạng thái flapping (online/offline nhiều lần), có cần throttle hay gửi đầy đủ stream?
5. Khi `schemaVersion` không khớp, consumer mong muốn provider từ chối hay publish kèm header `schemaVersion` và let consumer handle?

---

## 6. Rủi ro tích hợp & đề xuất xử lý

| Rủi ro | Tác động | Đề xuất xử lý |
|---|---|---|
| Thiếu `eventId` | Consumer khó dedupe → double-count | Provider phải luôn tạo `eventId`; ghi rõ trong hợp đồng |
| Timestamp không phải UTC | Consumer aggregate sai | Producer chuẩn hóa timestamp về ISO-8601 UTC hoặc kèm timezone info |
| Metrics lớn / batch | Consumer gặp vấn đề timing/size | Thống nhất kích thước tối đa cho `metrics`; cân nhắc tách ra nhiều events |
| Schema drift | Consumer parse/mapping lỗi | Dùng `schemaVersion`, tăng MAJOR khi incompatible và phối hợp qua AsyncAPI ở Lab03 |
| Producer không đặt partition | Ordering bị mất | Producer phải đặt partition key = `deviceId` khi publish |

