# Phân tích yêu cầu — vai Provider (Access Gate)

- Cặp đàm phán: Pair 03 — Core Business ↔ Access Gate
- Product: A / B
- Provider service: Access Gate
- Consumer service: Core Business
- Người viết: Đỗ Tuyến
- Ngày: 2026-05-17

---

## 1. Resource chính

| Resource | Mô tả | Thuộc tính bắt buộc | Thuộc tính tùy chọn |
|---|---|---|---|
| `AccessLog` | Bản ghi một lần quẹt thẻ ra/vào cổng | `logId`, `cardId`, `gateId`, `direction`, `timestamp`, `status` | `operatorNote` (nullable) |
| `GateStatus` | Trạng thái hiện tại của một cổng vật lý | `gateId`, `state`, `lastUpdated` | `faultReason` (nullable, chỉ có khi FAULT) |
| `Card` | Thông tin thẻ RFID và chủ thẻ | `cardId`, `holderName`, `holderType`, `issuedAt`, `cardStatus` | `expiresAt` (nullable) |
| `CardStatus` | Trạng thái thẻ (oneOf: Active/Suspended/Blocked) | `statusCode` + các field theo loại | — |

---

## 2. Action/API dự kiến

| Method | Path | Mục đích | Consumer gọi khi nào? |
|---|---|---|---|
| GET | `/health` | Kiểm tra service còn sống | Startup check, monitoring |
| GET | `/access/logs/recent` | Lấy log quẹt thẻ gần đây (có pagination, filter) | Core cần audit log định kỳ hoặc xử lý cảnh báo |
| GET | `/access/logs/{logId}` | Lấy chi tiết một log cụ thể | Core nhận được logId từ sự kiện khác và cần tra cứu |
| GET | `/gates/{gateId}/status` | Lấy trạng thái gate hiện tại | Core cần biết cổng nào đang FAULT để xử lý cảnh báo |
| GET | `/cards/{cardId}` | Lấy thông tin thẻ RFID | Core cần xác minh thẻ hoặc audit người dùng |

---

## 3. Error case

Tối thiểu 5 case.

| Status | Tình huống | Response body dự kiến |
|---:|---|---|
| 400 | Tham số query sai (ví dụ `direction=BOTH`) | `Problem` (ENUM_MISMATCH) |
| 401 | Thiếu Bearer token | `Problem` (unauthorized) |
| 403 | Token hợp lệ nhưng service không có quyền đọc dữ liệu Access Gate | `Problem` (forbidden) |
| 404 | `logId`, `gateId` hoặc `cardId` không tồn tại | `Problem` (not-found) |
| 422 | Khoảng thời gian `fromTime` > `toTime` | `Problem` (INVALID_TIME_RANGE) |
| 500 | Lỗi kết nối database nội bộ Access Gate | `Problem` (internal) |

---

## 4. Giả định bổ sung

- Giả định 1: Access Gate lưu log tối đa 90 ngày; query quá khoảng thời gian này vẫn trả 200 với `items: []`.
- Giả định 2: Mỗi thẻ có duy nhất một `cardStatus` tại một thời điểm; Core không được giả định thẻ luôn ACTIVE.
- Giả định 3: Access Gate không tự thay đổi trạng thái thẻ — chỉ đọc từ Card Management Service; API này chỉ là read-only.
- Giả định 4: Bearer token phải thuộc service `core-business` (scope `access-gate:read`).
- Giả định 5: `cursor` trong pagination là opaque string — Core không được parse hoặc tự sinh cursor.

---

## 5. Câu hỏi cho Consumer

1. Core Business có cần filter log theo khoảng thời gian `fromTime`/`toTime` không, hay chỉ cần `limit` gần nhất?
2. Core cần tần suất gọi `/gates/{gateId}/status` là bao nhiêu? (realtime mỗi giây hay polling mỗi phút?)
3. Khi thẻ bị BLOCKED, Core xử lý khác gì so với SUSPENDED — có cần biết `blockedReason` không?

---

## 6. Rủi ro tích hợp

| Rủi ro | Tác động | Đề xuất xử lý |
|---|---|---|
| Core parse `cardStatus` như object phẳng, không xét `statusCode` | Parse lỗi khi SUSPENDED/BLOCKED có field khác ACTIVE | Thống nhất dùng `oneOf` + `discriminator` trong `openapi.yaml` |
| Core gọi `/access/logs/recent` không có filter → trả quá nhiều record | Timeout, memory spike | Thống nhất `limit` mặc định = 20, tối đa = 100 |
| `operatorNote` null khiến Core crash nếu không kiểm tra null | NullPointerException phía Core | Ghi rõ `type: [string, "null"]` trong schema, Core phải xử lý null |
| Cổng bị FAULT mà Core vẫn ra quyết định dựa trên cache cũ | Quyết định sai | Core phải invalidate cache khi nhận state = FAULT |
