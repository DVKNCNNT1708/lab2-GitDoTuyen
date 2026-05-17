# Phân tích yêu cầu — vai Consumer (Core Business)

- Cặp đàm phán: Pair 03 — Core Business ↔ Access Gate
- Product: A / B
- Consumer service: Core Business
- Provider service: Access Gate
- Người viết: Đỗ Tuyến
- Ngày: 2026-05-17

---

## 1. Resource Consumer cần nhận/gửi

| Resource | Consumer dùng để làm gì? | Field bắt buộc với Consumer | Field có thể tùy chọn |
|---|---|---|---|
| `AccessLog` | Audit trail, phát hiện truy cập bất thường, tổng hợp báo cáo | `logId`, `cardId`, `gateId`, `direction`, `timestamp`, `status` | `operatorNote` |
| `GateStatus` | Kiểm tra cổng có hoạt động trước khi ra quyết định policy | `gateId`, `state`, `lastUpdated` | `faultReason` |
| `Card` | Xác minh danh tính chủ thẻ và trạng thái thẻ | `cardId`, `holderName`, `holderType`, `cardStatus` | `expiresAt` |

---

## 2. API Consumer cần gọi

| Method | Path | Lúc nào gọi? | Kỳ vọng response |
|---|---|---|---|
| GET | `/health` | Startup, health-check định kỳ | `{ status: "ok" }` |
| GET | `/access/logs/recent` | Định kỳ 5 phút hoặc khi nhận trigger alert | Danh sách `AccessLog` paginated |
| GET | `/access/logs/{logId}` | Khi cần tra cứu chi tiết log cụ thể từ incident | Chi tiết `AccessLog` đầy đủ |
| GET | `/gates/{gateId}/status` | Khi cần kiểm tra trạng thái cổng trước khi audit | `GateStatus` với `state` và `lastUpdated` |
| GET | `/cards/{cardId}` | Khi phát hiện sự kiện bất thường cần xác minh chủ thẻ | `Card` với `cardStatus` đầy đủ |

---

## 3. Error case Consumer cần xử lý

Tối thiểu 5 case.

| Status | Consumer hiểu là gì? | Consumer sẽ xử lý thế nào? |
|---:|---|---|
| 400 | Tham số query không hợp lệ (direction sai, fromTime format sai) | Log lỗi, sửa payload trước khi retry |
| 401 | Token hết hạn hoặc không gửi token | Refresh Bearer token và retry một lần |
| 403 | Token hợp lệ nhưng thiếu scope `access-gate:read` | Báo lỗi cấu hình, không retry, cần admin cấp quyền |
| 404 | LogId/gateId/cardId không tồn tại trong Access Gate | Hiển thị trạng thái "không tìm thấy", không retry |
| 422 | Khoảng thời gian `fromTime` > `toTime` | Kiểm tra lại logic sinh tham số, không retry |
| 500 | Lỗi phía Access Gate | Retry theo exponential backoff tối đa 3 lần, sau đó alert |

---

## 4. Giả định bổ sung

- Giả định 1: Access Gate trả `200` với `items: []` khi không có log trong khoảng thời gian, không trả `404`.
- Giả định 2: `cardStatus` là `oneOf` — Core phải đọc `statusCode` để biết loại trước khi xử lý field riêng.
- Giả định 3: Core không cache `GateStatus` quá 60 giây; nếu `state = FAULT`, Core phải invalidate ngay.
- Giả định 4: `cursor` là opaque — Core không cần biết format, chỉ truyền lại khi cần trang kế tiếp.
- Giả định 5: `expiresAt` có thể null (thẻ không hạn) — Core không được assume thẻ hết hạn nếu field này null.

---

## 5. Câu hỏi cho Provider

1. Log Access Gate lưu tối đa bao nhiêu ngày? Có cần Core gọi theo batch để tải full history không?
2. Khi cổng đang FAULT, Access Gate vẫn ghi log DENIED hay không ghi gì? Core cần biết để phân biệt DENIED thật sự vs FAULT.
3. Thẻ SUSPENDED có thể tự động chuyển về ACTIVE không (ví dụ: hết thời gian phạt)? Hay Core phải polling lại?

---

## 6. Rủi ro tích hợp

| Rủi ro | Tác động | Đề xuất xử lý |
|---|---|---|
| Provider đổi kiểu `cardStatus` từ object sang string enum | Core parse lỗi runtime | Chốt schema `oneOf` + `discriminator` trong `openapi.yaml`, không inline |
| Provider thêm giá trị mới vào enum `state` của GateStatus | Core switch-case không bắt được case mới | Core phải có default case xử lý unknown state |
| `operatorNote` null mà Core không kiểm tra | NullPointerException hoặc hiển thị "null" ra UI | Quy định rõ `type: [string, "null"]` và Core luôn kiểm tra null |
| Tốc độ gọi `/access/logs/recent` quá cao | Access Gate bị quá tải | Thống nhất polling interval ≥ 1 phút cho batch, dùng event trigger thay polling nếu cần realtime |
