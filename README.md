# VPay Card & Topup Partner API Documentation for AI Agents

Tài liệu này được tối ưu hóa để các **AI Coding Agents (như Antigravity, Cursor, Windsurf, Copilot)** và lập trình viên có thể đọc hiểu, thiết kế cơ sở dữ liệu và viết mã kết nối nhanh chóng vào hệ thống tích hợp nạp thẻ cào, mua mã thẻ, và nạp tiền điện thoại (Topup).

---

## 📌 BẢN ĐỒ ĐỐI CHIẾU CHỮ KÝ (SIGNATURE MATCHING MAP)

Các Agent cần đặc biệt lưu ý thứ tự ghép chuỗi MD5 để tạo chữ ký (`sign`). Bất kỳ sai lệch nào về thứ tự hoặc khoảng trắng đều dẫn đến lỗi `116: Sai chu ký` hoặc từ chối giao dịch.

| Chức năng | Phương thức | Endpoint | Công thức ghép `sign` |
| :--- | :---: | :--- | :--- |
| **Đổi thẻ cào** | `POST`/`GET` | `/chargingws/v2` | `md5(partner_key + code + serial)` |
| **Kiểm tra trạng thái thẻ** | `POST` | `/chargingws/v2` | `md5(partner_key + code + serial)` |
| **Nhận Callback đổi thẻ** | `POST`/`GET` | Callback URL | `md5(partner_key + code + serial)` |
| **Kiểm tra seri thẻ cào** | `POST` | `/api/checkcard` | `md5(partner_key + serial)` |
| **Mua thẻ cào** | `POST` | `/api/cardws` | `md5(partner_key + partner_id + command + request_id)` |
| **Kiểm tra tồn kho mua thẻ** | `POST` | `/api/cardws` | `md5(partner_key + partner_id + command + "")` |
| **Tải lại thẻ (Re-download)** | `POST` | `/api/cardws` | `md5(partner_key + partner_id + command + request_id)` |
| **Lấy số dư ví mua thẻ** | `POST` | `/api/cardws` | `md5(partner_key + partner_id + command + "")` |
| **Tạo lệnh nạp Topup** | `POST` (JSON) | `/api/rechargews` | `md5(partner_key + partner_id + command + request_id)` |
| **Kiểm tra đơn nạp Topup** | `POST` (JSON) | `/api/rechargews` | `md5(partner_key + partner_id + command + request_id)` |
| **Lấy số dư ví Topup** | `POST` (JSON) | `/api/rechargews` | `md5(partner_key + partner_id + command)` |

> [!WARNING]  
> Đối với các hàm không truyền `request_id` (như kiểm tra tồn kho, lấy số dư ví mua thẻ), giá trị `request_id` trong chuỗi ghép chữ ký sẽ là chuỗi rỗng `""`.

---

## 1. PHÂN HỆ ĐỔI THẺ (CARD CHARGING)

### 🚀 Gửi thẻ lên hệ thống (Charging Request)
* **URL:** `http://{{domain_post}}/chargingws/v2`
* **Method:** `POST` (khuyên dùng) hoặc `GET`
* **Format:** Form params (`application/x-www-form-urlencoded`) hoặc JSON.
* **Request Params:**
  ```json
  {
    "telco": "VIETTEL", // VIETTEL, MOBIFONE, VINAPHONE, GATE, Zing...
    "code": "312821445892982", // Mã PIN dưới lớp bạc
    "serial": "10004783347874", // Số Seri thẻ cào
    "amount": "50000", // Mệnh giá khai báo (bắt buộc)
    "request_id": "323233", // Mã giao dịch duy nhất tự sinh phía đối tác
    "partner_id": "3681148751", // ID đối tác do VPay cung cấp
    "command": "charging", // Cố định
    "sign": "19db4f1670100764069dba47429a9d94" // md5(partner_key + code + serial)
  }
  ```

* **Mã lỗi phản hồi từ API:**
  * `1`: Thẻ đúng, nạp thành công đúng mệnh giá.
  * `2`: Thẻ đúng, nạp thành công nhưng sai mệnh giá khai báo.
  * `3`: Thẻ lỗi (sai mã thẻ, seri, hoặc thẻ đã sử dụng).
  * `4`: Hệ thống bảo trì.
  * `99`: Thẻ chờ xử lý (đang duyệt).
  * `100`: Gửi thẻ thất bại (các lỗi nghiệp vụ khác, kiểm tra message phản hồi).

---

### 🚀 Nhận kết quả xử lý (Callback)
VPay gửi kết quả xử lý về Callback URL được cấu hình trên hệ thống của đối tác.

#### Dạng POST JSON (Khuyên dùng)
* **JSON Body:**
  ```json
  {
    "status": 1,
    "message": "Thành công",
    "request_id": "989876",
    "declared_value": 50000,
    "value": 50000,
    "amount": 25000, // Số tiền thực nhận sau chiết khấu
    "code": "314688440422676",
    "serial": "10003395125761",
    "telco": "VIETTEL",
    "trans_id": 54180, // Mã giao dịch hệ thống VPay
    "callback_sign": "17b118fe86852c52ea126c9537617f6d" // md5(partner_key + code + serial)
  }
  ```

#### Dạng GET Redirect
* **Query String:**
  `?status=1&message=Thành công&request_id=989876&declared_value=50000&card_value=50000&value=50000&amount=25000&code=314688440422676&serial=10003395125761&telco=VIETTEL&trans_id=343424&callback_sign=17b118fe86852c52ea126c9537617f6d`

---

## 2. PHÂN HỆ MUA THẺ CÀO (BUY CARD)
* **Base URL:** `https://your-domain.com/api/cardws`
* **Method:** `POST`

### 🚀 Tạo đơn mua mã thẻ cào (Buy Card)
* **Request Params:**
  ```json
  {
    "partner_id": "0299338261",
    "command": "buycard",
    "request_id": "113", // Mã giao dịch duy nhất
    "service_code": "Viettel", // Viettel, Vinaphone, Mobifone, Zing, Gate, v.v.
    "wallet_number": "0081083966", // Số ví của bạn
    "value": "10000", // Mệnh giá thẻ cần mua
    "qty": "2", // Số lượng thẻ cần mua
    "sign": "..." // md5(partner_key + partner_id + command + request_id)
  }
  ```

* **Mã lỗi phản hồi (Error Codes):**
  * `1`: Mua thẻ thành công.
  * `2`: Thanh toán thành công nhưng chưa lấy được thẻ cào ngay.
  * `102`: Số dư tài khoản không đủ.
  * `116`: Sai chữ ký bảo mật.
  * `118`: Sản phẩm này đã hết hàng trong kho.

### 🚀 Tải lại thẻ (Re-download)
Hàm này dùng để lấy lại thông tin danh sách thẻ của đơn hàng cũ trong trường hợp lỗi mạng mà không phát sinh thêm giao dịch trừ tiền mới.
* **Request Params:**
  ```json
  {
    "partner_id": "0299338261",
    "command": "redownload",
    "request_id": "113",
    "order_code": "S61797A53BCEEF", // Mã đơn hàng trả về khi mua thẻ
    "sign": "..." // md5(partner_key + partner_id + command + request_id)
  }
  ```

---

## 3. PHÂN HỆ NẠP TIỀN ĐIỆN THOẠI (TOPUP RECHARGE)
* **Base URL:** `http://{{domain_post}}/api/rechargews`
* **Method:** `POST`
* **Content-Type:** `application/json`

### 🚀 Tạo lệnh nạp tiền Topup
* **JSON Request:**
  ```json
  {
    "partner_id": "3681148751",
    "command": "topup",
    "request_id": "116",
    "service_code": "vinatt", // Loại dịch vụ Topup
    "amount": "10000",
    "qty": "1",
    "account_info": {
      "phone": "0943793984" // Số điện thoại cần nạp tiền
    },
    "sign": "..." // md5(partner_key + partner_id + command + request_id)
  }
  ```

---

## 💻 CODE MẪU CHO AGENT TÍCH HỢP NHANH (READY-TO-USE CODE SNIPPETS)

### 🐍 Python Helper (FastAPI / Flask / Django)
```python
import hashlib
import requests

class VPayAPI:
    def __init__(self, partner_id: str, partner_key: str, domain: str):
        self.partner_id = partner_id
        self.partner_key = partner_key
        self.domain = domain.rstrip('/')

    def _generate_md5_sign(self, *args) -> str:
        """Ghép nối các tham số và trả về mã MD5 chữ thường."""
        raw_str = "".join(str(arg) for arg in args)
        return hashlib.md5(raw_str.encode('utf-8')).hexdigest()

    def charge_card(self, telco: str, code: str, serial: str, amount: int, request_id: str):
        """Gửi thẻ cào lên hệ thống đổi thẻ."""
        url = f"{self.domain}/chargingws/v2"
        sign = self._generate_md5_sign(self.partner_key, code, serial)
        
        payload = {
            "telco": telco.upper(),
            "code": code,
            "serial": serial,
            "amount": str(amount),
            "request_id": request_id,
            "partner_id": self.partner_id,
            "command": "charging",
            "sign": sign
        }
        
        response = requests.post(url, data=payload)
        return response.json()

    def verify_callback(self, code: str, serial: str, received_sign: str) -> bool:
        """Xác minh tính hợp lệ của callback kết quả thẻ từ VPay."""
        calculated_sign = self._generate_md5_sign(self.partner_key, code, serial)
        return calculated_sign == received_sign

    def buy_card(self, service_code: str, value: int, qty: int, request_id: str, wallet_number: str):
        """Gửi lệnh mua thẻ cào từ hệ thống."""
        url = f"{self.domain}/api/cardws"
        command = "buycard"
        sign = self._generate_md5_sign(self.partner_key, self.partner_id, command, request_id)
        
        payload = {
            "partner_id": self.partner_id,
            "command": command,
            "request_id": request_id,
            "service_code": service_code,
            "wallet_number": wallet_number,
            "value": str(value),
            "qty": str(qty),
            "sign": sign
        }
        response = requests.post(url, data=payload)
        return response.json()
```

### ☕ Node.js / TypeScript Helper (Express / NestJS)
```javascript
const crypto = require('crypto');
const axios = require('axios');

class VPayAPI {
  constructor(partnerId, partnerKey, domain) {
    this.partnerId = partnerId;
    this.partnerKey = partnerKey;
    this.domain = domain.replace(/\/$/, '');
  }

  generateMD5Sign(...args) {
    const rawStr = args.map(arg => String(arg)).join('');
    return crypto.createHash('md5').update(rawStr, 'utf-8').digest('hex');
  }

  async chargeCard(telco, code, serial, amount, requestId) {
    const url = `${this.domain}/chargingws/v2`;
    const sign = this.generateMD5Sign(this.partnerKey, code, serial);

    const params = new URLSearchParams();
    params.append('telco', telco.toUpperCase());
    params.append('code', code);
    params.append('serial', serial);
    params.append('amount', String(amount));
    params.append('request_id', requestId);
    params.append('partner_id', this.partnerId);
    params.append('command', 'charging');
    params.append('sign', sign);

    const response = await axios.post(url, params, {
      headers: { 'Content-Type': 'application/x-www-form-urlencoded' }
    });
    return response.data;
  }

  verifyCallback(code, serial, receivedSign) {
    const calculatedSign = this.generateMD5Sign(this.partnerKey, code, serial);
    return calculatedSign === receivedSign;
  }
}
```

---

## 🛠 HƯỚNG DẪN DÀNH CHO AGENT KHI BUILD CODE

1. **Khởi tạo Database**:
   Thiết kế bảng `card_transactions` lưu vết đổi thẻ cào cần lưu tối thiểu:
   * `id` / `request_id` (Khóa chính, duy nhất)
   * `telco` (Nhà mạng)
   * `code` & `serial` (Mã PIN & Seri)
   * `declared_value` (Mệnh giá khai báo)
   * `actual_value` (Mệnh giá thực tế nhận được)
   * `amount` (Mức tiền thực nhận sau chiết khấu)
   * `status` (Trạng thái: `pending`, `success`, `wrong_value`, `failed`)
   * `trans_id` (Mã giao dịch của VPay phục vụ đối chiếu)

2. **Quy trình xử lý Callback**:
   * **Bước 1**: Nhận request từ VPay.
   * **Bước 2**: Xác minh chữ ký `callback_sign` bằng công thức: `md5(partner_key + code + serial)`. Nếu sai chữ ký, phản hồi `400` / `fail` ngay lập tức.
   * **Bước 3**: Tìm giao dịch trong DB theo `request_id`.
   * **Bước 4**: Kiểm tra `declared_value` và `value` (thực tế). Nếu `value == declared_value`, cập nhật trạng thái giao dịch thành công. Nếu khác nhau, xử lý theo nghiệp vụ thẻ sai mệnh giá (status = `wrong_value`).
   * **Bước 5**: Cộng số dư/xu cho người dùng tương ứng với số tiền nhận được (`amount` hoặc `value`).
   * **Bước 6**: Phản hồi lại VPay `{"status": true}` hoặc `OK` để xác nhận đã nhận callback thành công.
