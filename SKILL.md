---
name: vpay-partner-api-integration
description: >-
  Hỗ trợ agent thiết kế, xây dựng và tích hợp các chức năng nạp thẻ cào (charging), mua mã thẻ cào (buy card), nạp tiền điện thoại (Topup) và kiểm tra seri thẻ cào qua hệ thống VPay API.
---

# VPay Partner API Integration Skill

## Overview

Skill này hướng dẫn các AI Agent (như Antigravity, Cursor, Windsurf, Copilot, v.v.) nhanh chóng viết mã nguồn, thiết kế cấu trúc database và tích hợp hệ thống **VPay Partner API** vào dự án của người dùng một cách chính xác và hiệu quả nhất.

Nhiệm vụ của Agent bao gồm:
1. Thiết kế cơ sở dữ liệu để lưu trữ lịch sử thẻ cào, trạng thái giao dịch.
2. Viết các module/service để gọi API gửi thẻ, mua thẻ, nạp topup.
3. Thiết kế endpoint nhận Callback từ VPay và đối soát bảo mật.
4. Xử lý các trường hợp lỗi nghiệp vụ (như nạp sai mệnh giá, lỗi kết nối, sai chữ ký).

---

## Quick Start (Tích hợp nhanh)

Dưới đây là sơ đồ tích hợp nhanh cho các Agent:

### 1. Tạo Chữ Ký MD5 (Signature Generation)
Bắt buộc phải ghép các chuỗi tham số theo đúng thứ tự (không có ký tự phân tách hay khoảng trắng) trước khi chuyển đổi sang chuỗi MD5 chữ thường (lowercase).

```python
import hashlib

def generate_vpay_signature(partner_key: str, *params) -> str:
    # Ghép tất cả các tham số lại với nhau
    raw_str = "".join(str(p) for p in params)
    # Mã hóa MD5 và trả về chuỗi viết thường
    return hashlib.md5(raw_str.encode('utf-8')).hexdigest()
```

### 2. Thiết kế Database Tối thiểu
Bảng lưu vết giao dịch đổi thẻ cào (`card_transactions`):
```sql
CREATE TABLE IF NOT EXISTS card_transactions (
    request_id VARCHAR(50) PRIMARY KEY, -- ID duy nhất đối tác tự sinh
    telco VARCHAR(20) NOT NULL,          -- Nhà mạng (VIETTEL, MOBIFONE...)
    code VARCHAR(30) NOT NULL,           -- Mã PIN thẻ cào
    serial VARCHAR(30) NOT NULL,         -- Số Seri thẻ cào
    declared_value INT NOT NULL,         -- Mệnh giá khai báo gửi lên
    actual_value INT DEFAULT 0,          -- Mệnh giá thực tế do VPay trả về
    amount INT DEFAULT 0,                -- Số tiền thực nhận sau chiết khấu
    status VARCHAR(20) DEFAULT 'pending',-- pending, success, wrong_value, failed
    trans_id INT,                        -- ID giao dịch phía VPay
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

---

## Workflow (Quy trình dành cho Agent)

### Bước 1: Gửi thẻ cào lên VPay (`charging`)
Khi người dùng thực hiện nạp thẻ cào trên hệ thống của bạn, Agent cần thực hiện:

1. Tạo một mã `request_id` duy nhất.
2. Lưu thông tin thẻ vào database với trạng thái `pending`.
3. Tạo chữ ký `sign` bằng công thức:
   `sign = md5(partner_key + code + serial)`
4. Gửi yêu cầu HTTP POST (dưới dạng formdata) tới:
   `http://{{domain_post}}/chargingws/v2`
   * Tham số: `telco`, `code`, `serial`, `amount`, `request_id`, `partner_id`, `sign`, `command="charging"`
5. Nhận phản hồi từ API:
   * Nếu nhận được `status = 99` (chờ xử lý) hoặc `status = 1` (thành công ngay lập tức): Giữ trạng thái giao dịch trong DB là `pending` hoặc chuyển thành `success` nếu xử lý đồng bộ.
   * Nếu nhận được mã lỗi khác: Cập nhật giao dịch thành `failed` kèm thông báo lỗi.

### Bước 2: Xử lý Callback kết quả từ VPay
VPay sẽ gọi về callback URL của bạn (GET hoặc POST JSON). Khi nhận được request:

1. **Xác thực chữ ký:**
   * Tính toán chữ ký hợp lệ: `callback_sign_calc = md5(partner_key + code + serial)`
   * Đối chiếu với `callback_sign` do VPay gửi sang. Nếu không trùng khớp, trả về mã lỗi HTTP `400 Bad Request` và ghi log cảnh báo bảo mật. Do NOT xử lý tiếp.
2. **Cập nhật Database:**
   * Tìm giao dịch tương ứng với `request_id`.
   * So sánh mệnh giá khai báo (`declared_value`) và mệnh giá thực tế nhận được (`value` hoặc `card_value`):
     * **Trường hợp đúng mệnh giá:** Cập nhật `status = 'success'`, lưu `actual_value` và số tiền thực nhận `amount`. Thực hiện cộng số dư/xu cho người dùng.
     * **Trường hợp sai mệnh giá (status = 2):** Cập nhật `status = 'wrong_value'`, ghi nhận thực tế và xử lý phạt chiết khấu hoặc hủy thẻ tùy theo chính sách hệ thống.
     * **Trường hợp lỗi (status = 3):** Cập nhật `status = 'failed'`.
3. **Phản hồi hệ thống VPay:**
   * Trả về JSON `{"status": true}` hoặc text `OK` để VPay ngừng gửi lại callback.

### Bước 3: Mua mã thẻ cào (`buycard`)
Dành cho chức năng mua thẻ cào từ hệ thống:

1. Tạo chữ ký: `sign = md5(partner_key + partner_id + command + request_id)` (với `command = "buycard"`).
2. Gửi request POST JSON đến endpoint `https://tenmien.com/api/cardws`.
3. Kiểm tra mã lỗi trả về:
   * `1`: Thành công -> Trích xuất danh sách thẻ (seri, mã thẻ, hạn sử dụng) trả về cho người dùng.
   * `2`: Thanh toán thành công nhưng hệ thống đang lấy thẻ -> Chuyển giao dịch thành trạng thái chờ tải lại (Re-download).
   * Lỗi số dư (`102`): Thông báo không đủ tiền trong ví API.
   * Lỗi hết hàng (`118`): Thông báo loại thẻ/mệnh giá này đang tạm hết.

---

## Common Mistakes (Các lỗi Agent thường gặp)

1. **Sai thứ tự ghép chữ ký:**
   * ❌ Sai: `md5(partner_id + partner_key + command + request_id)`
   *  Đúng: `md5(partner_key + partner_id + command + request_id)` (đối với mua thẻ/topup).
   * ❌ Sai: `md5(code + serial + partner_key)`
   *  Đúng: `md5(partner_key + code + serial)` (đối với nạp thẻ/callback).

2. **Quên xử lý trường hợp sai mệnh giá (lỗi lệch mệnh giá):**
   * Nếu người dùng khai báo mệnh giá 50k nhưng thực tế nạp thẻ 100k (hoặc ngược lại), VPay trả về `status = 2`. Nếu Agent chỉ kiểm tra `status == 1` thì sẽ bỏ sót giao dịch thành công sai mệnh giá này, dẫn đến mất mát tiền của hoặc không cộng tiền cho khách hàng.

3. **Thiếu kiểm tra trùng lặp `request_id`:**
   * VPay có thể gửi lại callback nhiều lần nếu kết nối mạng chập chờn. Agent cần kiểm tra trạng thái giao dịch trước khi cộng tiền cho người dùng để tránh việc cộng tiền nhiều lần (Idempotency).
