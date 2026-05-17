# Biên bản đàm phán hợp đồng API

- Cặp đàm phán: Pair 03 — Core Business ↔ Access Gate
- Product: A / B
- Provider: Access Gate
- Consumer: Core Business
- Phiên: v1.0
- Ngày: 2026-05-17

---

## Issue #1

- **Raised by**: Consumer
- **Endpoint**: `GET /access/logs/recent`
- **Concern**: Consumer đề xuất dùng offset pagination (`?page=1&size=20`) để dễ implement phía Core. Provider lo ngại skip lớn làm chậm DB khi log nhiều.
- **Proposal**: Dùng cursor-based pagination với `cursor` + `limit`. Consumer truyền `cursor` từ response trước vào request tiếp theo.
- **Resolution**: Accepted
- **Rationale**: Cursor pagination ổn định hơn khi dữ liệu thay đổi liên tục (log vào/ra realtime). Tránh vấn đề "page drift" khi có record mới chèn vào giữa 2 lần query.
- **Impact**: Consumer phải lưu lại `nextCursor` từ response để query trang kế tiếp. Không thể nhảy trực tiếp đến trang N.

---

## Issue #2

- **Raised by**: Provider
- **Endpoint**: `GET /access/logs/recent`, `GET /access/logs/{logId}`
- **Concern**: Provider dùng tên field `accessDirection` (đầy đủ hơn), Consumer muốn dùng `direction` (ngắn gọn, khớp với naming trong Core Business schema nội bộ).
- **Proposal**: Giữ tên `direction` để nhất quán với codebase Consumer.
- **Resolution**: Accepted
- **Rationale**: `direction` là tên đủ rõ nghĩa trong context access log. Dùng tên ngắn giúp Consumer không cần map field. Có thể thêm `description: "Hướng vào (IN) hoặc ra (OUT)"` vào schema để bù đắp.
- **Impact**: Access Gate phải serialize tên field là `direction` thay vì `accessDirection` trong response. Core không cần mapping.

---

## Issue #3

- **Raised by**: Consumer
- **Endpoint**: `GET /access/logs/recent`, `GET /access/logs/{logId}`
- **Concern**: `operatorNote` — Consumer muốn field này là `required` để luôn có giá trị. Provider nói rằng không phải log nào cũng có ghi chú của nhân viên vận hành.
- **Proposal**: Để `operatorNote` là optional với `type: [string, "null"]`. Khi không có ghi chú, Access Gate trả `null`.
- **Resolution**: Modified — `operatorNote` không nằm trong `required`, nhưng khi có mặt thì phải là string hoặc null. Provider cam kết luôn trả field này (kể cả null) trong response.
- **Rationale**: OpenAPI 3.1 cho phép union type với `null` thay vì `nullable: true`. Việc Provider luôn trả field (kể cả null) giúp Consumer không phải kiểm tra `field not present` vs `field is null`.
- **Impact**: Consumer phải xử lý null cho `operatorNote`. Schema dùng `type: [string, "null"]` đúng chuẩn OpenAPI 3.1.

---

## Issue #4

- **Raised by**: Consumer
- **Endpoint**: `GET /cards/{cardId}`
- **Concern**: Consumer muốn `cardStatus` trả về là một string enum đơn giản (`ACTIVE`/`SUSPENDED`/`BLOCKED`). Provider muốn trả thêm metadata theo từng loại (ngày khóa, lý do khóa...).
- **Proposal**: Dùng `oneOf` + `discriminator` với `statusCode` là discriminator field. Mỗi loại status là một schema riêng với field đặc trưng.
- **Resolution**: Accepted
- **Rationale**: `oneOf` + `discriminator` là pattern chuẩn OpenAPI 3.1 cho polymorphic response. Consumer có thể đọc `statusCode` trước để quyết định parse schema nào. Thể hiện đúng tiêu chí lab A3.
- **Impact**: Consumer phải implement logic đọc `statusCode` trước khi xử lý `cardStatus`. Không thể dùng flat object mapping. Prism mock hỗ trợ discriminator.

---

## Issue #5

- **Raised by**: Consumer
- **Endpoint**: `GET /access/logs/recent`
- **Concern**: Bao lâu thì Access Gate xóa log cũ? Core cần query log lịch sử để audit và báo cáo tháng.
- **Proposal**: Provider cam kết lưu log tối thiểu 90 ngày. Query ngoài khoảng trả `200` với `items: []`.
- **Resolution**: Accepted
- **Rationale**: 90 ngày đủ cho nhu cầu audit theo quy định nội bộ. Trả `200 + items: []` (không phải `404`) để Consumer không cần xử lý error riêng cho "hết hạn".
- **Impact**: Access Gate phải implement retention policy 90 ngày. Consumer phải hiểu `items: []` có nghĩa là "không có log" (không phải lỗi).

---

## Issue #6

- **Raised by**: Provider
- **Endpoint**: Tất cả endpoint
- **Concern**: Core Business có thể gọi `/access/logs/recent` liên tục (polling ngắn) gây quá tải Access Gate. Không có cơ chế rate limit trong contract.
- **Proposal**: Ghi vào contract: polling interval tối thiểu 60 giây cho `/access/logs/recent`. Cho phép gọi ngay lập tức khi có event trigger (không phải scheduled polling).
- **Resolution**: Accepted
- **Rationale**: Giảm tải Access Gate, tránh tình huống Core polling quá mức khi hệ thống bình thường. Event-driven trigger vẫn được phép để xử lý realtime khi cần.
- **Impact**: Consumer phải implement throttle/debounce 60 giây cho scheduled polling. Access Gate có thể trả `429 Too Many Requests` nếu vi phạm (sẽ bổ sung vào contract nếu cần Lab 03).

---

# Chốt hợp đồng v1.0

Provider sign-off:  Access Gate Team — Đỗ Tuyến  
Consumer sign-off:  Core Business Team — Đỗ Tuyến  
Witness (GV/TA):    FIT4110 Teaching Team  
Date:               2026-05-17

---

## Ghi chú warning nếu Spectral còn cảnh báo

| Warning | Lý do chấp nhận tạm thời | Kế hoạch sửa |
|---|---|---|
| Không có tag `gates` / `cards` trên webhook | Lab 02 không yêu cầu webhook cho cặp này | Bỏ webhook, chỉ dùng REST paths |
