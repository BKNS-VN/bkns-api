# BKNS Domain userapi — Hướng dẫn tích hợp cho lập trình viên

> Tài liệu này mô tả **cách gọi trực tiếp BKNS userapi** để quản lý tên miền, để bất kỳ
> lập trình viên nào (kể cả ngoài WHMCS) cũng có thể tự viết module/SDK cho riêng mình.
> Module mẫu tham chiếu: [`bknsdomains.php`](bknsdomains.php) (WHMCS registrar) — mọi ví dụ
> dưới đây đều trích từ chính code đã chạy thật trên môi trường OTE của BKNS.
>
> **Nguyên tắc vàng:** *không tự suy luận API.* Tài liệu này gắn nhãn độ tin cậy cho từng
> endpoint:
> - ✅ **Đã xác nhận** — có fixture phản hồi thật hoặc đã test end-to-end trên OTE.
> - ⚠️ **Chưa xác nhận** — suy ra từ nghiệp vụ, **phải hỏi BKNS** trước khi tin dùng.
>
> Phiên bản module: **1.1.0** · PHP **7.4 – 8.5** · Cập nhật: 2026-06-09.

---

## Mục lục

1. [Tổng quan & môi trường](#1-tổng-quan--môi-trường)
2. [Xác thực (JWT login)](#2-xác-thực-jwt-login)
3. [Quy ước request / response](#3-quy-ước-request--response)
4. [Bảng endpoint](#4-bảng-endpoint)
5. [Chi tiết từng endpoint](#5-chi-tiết-từng-endpoint)
6. [Đối tượng Contact (chủ thể) & trường `.vn`](#6-đối-tượng-contact-chủ-thể--trường-vn)
7. [⭐ Địa chỉ hành chính — khớp 2 file Excel](#7--địa-chỉ-hành-chính--khớp-2-file-excel)
8. [Validate `.vn` trước khi tạo order](#8-validate-vn-trước-khi-tạo-order)
9. [Thanh toán / số dư](#9-thanh-toán--số-dư)
10. [Những gì KHÔNG hỗ trợ](#10-những-gì-không-hỗ-trợ)
11. [Tự viết module tối thiểu (mẫu)](#11-tự-viết-module-tối-thiểu-mẫu)
12. [Checklist tích hợp](#12-checklist-tích-hợp)

---

## 1. Tổng quan & môi trường

BKNS userapi là REST/JSON, xác thực bằng **JWT Bearer token**. Mọi endpoint nằm dưới một
base URL; chọn base theo môi trường:

| Môi trường | Base URL | Ghi chú |
|---|---|---|
| **Production** | `https://my.bkns.net/api/` | tài khoản & credit thật |
| **Test (OTE)** | `https://id-ote.bkns.vn/api/` | thử nghiệm, **không trừ tiền thật** |

```php
const BKNSDOMAINS_API_BASE = 'https://my.bkns.net/api/';
const BKNSDOMAINS_API_TEST = 'https://id-ote.bkns.vn/api/';
```

- Mọi URL endpoint dưới đây là **tương đối** so với base (vd `domain/order` → `https://my.bkns.net/api/domain/order`).
- Body POST/PUT là JSON (`Content-Type: application/json`). Body rỗng phải gửi `{}` (object) — **không** gửi `[]` (array), vì API strict.
- GET truyền tham số qua query string.
- Tài liệu gốc của BKNS: <https://my.bkns.net/userapi>

---

## 2. Xác thực (JWT login)

### `POST login` ✅

Đăng nhập bằng **email + mật khẩu API** của tài khoản BKNS, nhận lại JWT token. Token này
đính vào header `Authorization: Bearer <token>` cho **mọi** request sau đó.

**Request**
```http
POST https://my.bkns.net/api/login
Content-Type: application/json
Accept: application/json

{
  "username": "email-api@example.com",
  "password": "matkhau-api"
}
```

**Response (200)**
```json
{ "token": "eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9..." }
```

**Quy tắc:**
- Thành công khi response có key `token` không rỗng.
- **Nên cache token** trong vòng đời tiến trình để tránh login lặp lại (module cache theo
  `md5(username|password|test_mode)`).
- Token sai/hết hạn → các request sau trả HTTP 401; xử lý bằng cách login lại.

Ví dụ PHP (curl) — trích từ `bknsdomains_login()`:
```php
$ch = curl_init('https://my.bkns.net/api/login');
curl_setopt_array($ch, [
    CURLOPT_RETURNTRANSFER => true,
    CURLOPT_TIMEOUT        => 30,
    CURLOPT_SSL_VERIFYPEER => true,
    CURLOPT_SSL_VERIFYHOST => 2,
    CURLOPT_POST           => true,
    CURLOPT_HTTPHEADER     => ['Content-Type: application/json', 'Accept: application/json'],
    CURLOPT_POSTFIELDS     => json_encode(['username' => $user, 'password' => $pass]),
]);
$raw   = curl_exec($ch);
$token = json_decode($raw, true)['token'] ?? null;
```

---

## 3. Quy ước request / response

### Header chuẩn cho request đã xác thực
```http
Authorization: Bearer <token>
Accept: application/json
Content-Type: application/json   # chỉ cho POST/PUT
```

### Cách BKNS báo thành công / thất bại

BKNS **không thống nhất 1 marker duy nhất**; client nên kiểm tra theo thứ tự sau (logic
`bknsdomains_rs_ok()`):

1. Có `error` hoặc `errors` (không rỗng) → **thất bại** ngay.
2. **HTTP status phải là 2xx.** 4xx/5xx mà body không có key `error` vẫn coi là **thất bại**
   (đừng tin mỗi body).
3. Marker thành công, xét lần lượt:
   - `token` tồn tại (login).
   - `success === true | 1 | "1"`.
   - `status ∈ {success, active, ok}` (so sánh không phân biệt hoa thường).
   - `details` là mảng (vd tra cứu domain).
   - `result ∈ {success, ok}`.
4. Nếu HTTP 2xx, không có dấu hiệu lỗi và không marker nào khớp → coi là thành công.

> 💡 **Khuyến nghị:** luôn đọc **HTTP status code** song song với body. Module gắn thêm
> `_http_code` vào mảng decode để kiểm tra ở bước 2.

### Cách đọc thông điệp lỗi (`bknsdomains_rs_error()`)

Lỗi có thể nằm ở `error` (string hoặc array), `errors` (array), hoặc `message`. Lấy theo
thứ tự đó; mỗi phần tử array có thể là string hoặc `{message: ...}`.

```json
{ "success": false, "error": "Insufficient balance" }
{ "errors": [ { "message": "domain already registered" } ] }
```

---

## 4. Bảng endpoint

| # | Method & Endpoint | Mục đích | Độ tin cậy |
|---|---|---|---|
| 1 | `POST login` | Lấy JWT token | ✅ test OTE |
| 2 | `GET balance` | Số dư / credit tài khoản | ✅ test OTE |
| 3 | `POST domain/order` | Đăng ký (`action=register`) hoặc chuyển về (`action=transfer`) | ✅ register OTE |
| 4 | `POST domain/{id}/renew` | Gia hạn theo BKNS domain **id** | ✅ test OTE |
| 5 | `GET domain/name/{domain}` | Tra cứu theo **tên miền** → danh sách bản ghi | ✅ fixture thật |
| 6 | `GET domain/{id}` | Chi tiết 1 bản ghi + **contacts** | ✅ fixture thật |
| 7 | `POST domain/nameservers` | Đổi nameserver | ⚠️ từ code |
| 8 | `POST domain/lock` | Khoá / mở khoá | ⚠️ từ code |
| 9 | `POST invoice/{id}/credit` | Trả invoice bằng credit | ✅ spec BKNS (thường tự trừ) |
| 10 | `POST domain/contacts` | Đổi thông tin chủ thể | ⚠️ **CHƯA xác nhận** |

> **Lưu ý quan trọng về thanh toán:** endpoint `domain/order` (đăng ký/transfer) và
> `domain/{id}/renew` (gia hạn) **tự động trừ credit** khi tạo order (BKNS xác nhận
> 2026-05-29). Vì vậy **không** cần gọi `invoice/{id}/credit` thủ công cho 3 nghiệp vụ này —
> chỉ cần kiểm tra số dư trước (xem [§9](#9-thanh-toán--số-dư)).

---

## 5. Chi tiết từng endpoint

### 5.1 `GET balance` ✅ — Kiểm tra số dư

**Request:** `GET https://my.bkns.net/api/balance` (chỉ cần header Authorization).

**Response (200)**
```json
{
  "success": 1,
  "details": {
    "currency": "VND",
    "acc_balance": "0.000",
    "acc_credit": "1500000.000"
  }
}
```

| Field | Ý nghĩa |
|---|---|
| `details.acc_credit` | Credit đang có trong tài khoản |
| `details.acc_balance` | Tổng invoice đang nợ (chưa trả) |
| **`available`** | = `acc_credit − acc_balance` → credit thực sự dùng được |

Gọi **trước** mỗi nghiệp vụ tốn tiền để tránh tạo order treo do thiếu credit.

---

### 5.2 `POST domain/order` ✅ — Đăng ký / Chuyển về

Một endpoint dùng cho cả hai nghiệp vụ; phân biệt bằng field **`action`** trong body
(`register` hoặc `transfer`).

**Request — đăng ký (`action=register`)**
```json
{
  "domain": "vidu.com.vn",
  "name": "vidu.com.vn",
  "tld": "com.vn",
  "years": 1,
  "action": "register",
  "pay_method": 300,
  "nameservers": ["ns1.bkdns.vn", "ns2.bkdns.vn"],
  "registrant": { "...": "xem §6" },
  "admin":      { "...": "thường clone từ registrant" },
  "tech":       { "...": "..." },
  "billing":    { "...": "..." },
  "data": { "promocode": "" }
}
```

**Request — chuyển về (`action=transfer`)** — giống trên nhưng:
- `action` = `"transfer"`,
- thêm `"epp": "<mã EPP/auth code>"`,
- không có khối `data`.

**Response (200)** — các field client cần lưu lại:
```json
{
  "success": true,
  "items": [
    { "id": 72725, "name": "vidu.com.vn", "product_id": 1234 }
  ],
  "order_num": 56789,
  "invoice_id": 98765
}
```

| Field | Dùng để |
|---|---|
| `items[].id` | **BKNS domain id** — lưu lại để gia hạn/đọc contact sau này (`subscriptionid`) |
| `items[].name` | đối chiếu đúng tên miền vừa tạo |
| `items[].product_id` | audit |
| `order_num`, `invoice_id` | đối soát / audit |

- `pay_method` = `300` (giữ theo giá trị gốc của module).
- `years` = số năm đăng ký.
- `nameservers` là mảng (tối đa 5; phần tử rỗng bị loại).

---

### 5.3 `POST domain/{id}/renew` ✅ — Gia hạn

⚠️ **id nằm trong URL path, KHÔNG nằm trong body.** Đây là BKNS domain id (lấy từ
`items[].id` lúc đăng ký, hoặc tra lại — xem §5.4).

**Request**
```http
POST https://my.bkns.net/api/domain/342342/renew

{ "years": "1", "pay_method": 300 }
```

**Response (200)**
```json
{ "success": true, "order_num": 56790, "invoice_id": 98766 }
```

> 🛑 **Chống gia hạn nhầm:** một tên miền có thể có **nhiều bản ghi** ở BKNS (vd bản cũ
> `Expired` + bản mới `Active` sau khi đăng ký lại). **Luôn tra lại id đúng theo tên**
> (`GET domain/name/{domain}`, ưu tiên bản `Active`) ngay trước khi renew — đừng tin id đã
> cache, nếu không sẽ gia hạn nhầm bản hết hạn. Module gọi `resolve_bkns_id(..., $authoritative=true)`.

---

### 5.4 `GET domain/name/{domain}` ✅ — Tra theo tên miền

Trả **danh sách** bản ghi cùng tên (có thể nhiều). Dùng cho: tìm id, đọc nameserver, đồng
bộ trạng thái/hạn.

**Request:** `GET https://my.bkns.net/api/domain/name/demo.vn`

**Response (200)** — fixture thật ([`domain-name-response.json`](domain-name-response.json)):
```json
{
  "success": true,
  "domains": [
    {
      "id": "63834", "name": "demo.vn", "status": "Expired",
      "date_created": "2023-06-01", "expires": "2025-06-01",
      "period": "2", "nameservers": ["ns1.bkdns.vn","ns2.bkdns.vn","ns3.bkdns.vn","",""],
      "autorenew": "1", "idprotection": "0"
    },
    {
      "id": "72725", "name": "demo.vn", "status": "Active",
      "date_created": "2025-11-29", "expires": "2026-11-29",
      "period": "1", "nameservers": ["ns1.bkdns.vn","ns2.bkdns.vn","ns3.bkdns.vn","",""],
      "autorenew": "1", "idprotection": "0"
    }
  ],
  "page": { "current": 0, "total": 3 }
}
```

**Cách chọn đúng bản ghi** (logic `bknsdomains_find_domain()`):
1. Lọc các phần tử `name` khớp tên (so sánh không phân biệt hoa thường, đã trim).
2. Nhiều bản ghi → **ưu tiên `status == "Active"`**, sau đó bản có `expires` **xa nhất**.
3. Nếu không khớp tên mà danh sách chỉ có **đúng 1** phần tử → dùng phần tử đó.

> Field `contacts` **KHÔNG có** ở đây — muốn đọc chủ thể phải gọi `GET domain/{id}` (§5.5).

---

### 5.5 `GET domain/{id}` ✅ — Chi tiết + contacts

**Request:** `GET https://my.bkns.net/api/domain/74848`

**Response (200)** — fixture thật ([`domain-id-response.json`](domain-id-response.json), rút gọn):
```json
{
  "success": true,
  "details": {
    "id": "72725", "name": "demo.vn", "status": "Active",
    "expires": "2026-11-29", "nameservers": ["ns1.bkdns.vn","ns2.bkdns.vn","ns3.bkdns.vn","",""],
    "contacts": {
      "registrant": {
        "type": "Private", "companyname": "", "taxid": null,
        "gender": "Male", "lastname": "Nguyễn Văn", "firstname": "Test",
        "email": "demo@gmail.com", "phonenumber": "+84-96849474948",
        "nationalid": "001094444444", "birthday": "1999-12-04",
        "country": "VN",
        "state": "Thành phố Hà Nội",
        "city":  "Phường Hoàng Mai",
        "ward":  null,
        "address1": "199 Giáp Bát", "postcode": "",
        "__nocontact": true, "__sameas": null
      },
      "admin":   { "...": "...", "__sameas": "registrant" },
      "tech":    { "...": "...", "__sameas": "registrant" },
      "billing": { "...": "...", "__sameas": "registrant" }
    }
  }
}
```

> ⭐ Để ý `state` = **tỉnh/thành**, `city` = **phường/xã** — đây là mấu chốt của
> [§7](#7--địa-chỉ-hành-chính--khớp-2-file-excel).

---

### 5.6 `POST domain/nameservers` ⚠️ — Đổi nameserver

```json
{ "domain": "vidu.com.vn", "ns1": "ns1.bkdns.vn", "ns2": "ns2.bkdns.vn" }
```
- Yêu cầu **tối thiểu 2 nameserver**.
- Để **đọc** nameserver hiện tại: dùng `GET domain/name/{domain}` rồi lấy mảng `nameservers`
  của bản ghi (lọc bỏ phần tử rỗng).

### 5.7 `POST domain/lock` ⚠️ — Khoá / mở khoá

```json
{ "domain": "vidu.com.vn", "lock": "1" }   // "1" = khoá, "0" = mở
```

### 5.8 `POST invoice/{id}/credit` ✅ spec — Trả invoice bằng credit

> Thông thường **không cần** gọi (register/renew/transfer đã tự trừ). Giữ lại cho trường
> hợp cần trả tay một invoice.

**Request:** body có `amount` thì trả đúng số đó; **bỏ trống** (`{}`) thì BKNS tự trả tối đa
bằng credit.
```json
{ "amount": 200000 }
```
**Response:** chỉ coi là xong khi `success === true` **VÀ** `invoice_status === "Paid"`.
```json
{ "success": true, "invoice_status": "Paid", "applied": 200000 }
```
Trạng thái `Partial` / `Unpaid` → coi như chưa trả xong, nạp thêm credit rồi thử lại.

---

## 6. Đối tượng Contact (chủ thể) & trường `.vn`

Mỗi order đăng ký/chuyển gửi **4 vai trò**: `registrant`, `admin`, `tech`, `billing`. Với
`.vn`, module clone cùng 1 contact (chủ thể) cho cả 4 vai trò.

### Shape contact gửi đi (BKNS `domain/order`)

| Field gửi đi | Kiểu | Bắt buộc | Ghi chú |
|---|---|---|---|
| `type` | `ind` \| `org` | ✅ | cá nhân / tổ chức (response đọc về là `Private` / `Organization`) |
| `firstname` | string | ✅ | tên |
| `lastname` | string | ✅ | họ + tên đệm |
| `companyname` | string | org | tên tổ chức (khi `type=org`) |
| `taxid` | string | org | mã số thuế (≥10 chữ số) |
| `gender` | `Male` \| `Female` | cá nhân | gửi rỗng nếu không có |
| `nationalid` | string | cá nhân | CMND/CCCD 9–12 chữ số |
| `birthday` | `YYYY-MM-DD` | cá nhân | ngày sinh |
| `email` | string | ✅ | email hợp lệ |
| `phonenumber` | string | ✅ | ≥ 8 chữ số |
| `country` | string | ✅ | mặc định `VN` |
| **`state`** | string | ✅ (.vn) | **Tỉnh/Thành phố** — xem §7 |
| **`city`** | string | ✅ (.vn) | **Phường/Xã** — xem §7 |
| `address1` | string | ✅ | số nhà, tên đường (phần địa chỉ chi tiết) |
| `postcode` | string | — | mã bưu chính (thường rỗng) |
| `position` | string | — | chức vụ |
| `__nocontact` | bool | — | **giữ `true`** → BKNS không tạo contact mới, tránh trùng email |

> **`__nocontact => true` là cố ý** — đừng bỏ. Nó ngăn BKNS sinh contact trùng cho cùng một
> email. (Xem ghi nhớ dự án *bknsdomains __nocontact flag*.)

### Trường định danh `.vn` (VNNIC)

Tên miền `.vn` cần thêm thông tin so với tên miền quốc tế:

| Hình thức | Trường bắt buộc thêm |
|---|---|
| **Cá nhân** (`ind`) | `nationalid` (CMND/CCCD 9–12 số), `gender`, `birthday` |
| **Tổ chức** (`org`) | `companyname`, `taxid` (≥10 số) |

Trong WHMCS các trường này khai bằng **Additional Domain Fields**
([`additionalfields.php`](additionalfields.php)): `Type` (Cá nhân/Công ty), `Tax`, `CMND`,
`Birthday` (YYYY-MM-DD), `Gender` (Nam/Nữ), `Position`. Nếu bạn tự viết hệ thống khác, hãy
thu thập đúng các trường tương đương.

> **Giữ dấu tiếng Việt:** tên người và địa chỉ phải giữ nguyên dấu. Trong WHMCS dùng
> `$params['original']` (bản chưa sanitize). Hệ thống khác: đảm bảo UTF-8 end-to-end, không
> bỏ dấu.

---

## 7. ⭐ Địa chỉ hành chính — khớp 2 file Excel

> Đây là phần **dễ sai nhất** khi tự code. Đọc kỹ.

Theo mô hình hành chính Việt Nam **2 cấp** (sau sắp xếp 2025: bỏ cấp quận/huyện, chỉ còn
**Tỉnh/Thành → Phường/Xã**), BKNS map contact như sau:

| Cấp hành chính | Field BKNS | Phải khớp file | Cột tên |
|---|---|---|---|
| **Tỉnh / Thành phố** | `state` | [`Tinh_thanh_Viet_Nam.xlsx`](Tinh_thanh_Viet_Nam.xlsx) | `TÊN TỈNH THÀNH` (cột B) |
| **Phường / Xã** | `city` | [`Phuong_xa_Viet_Nam.xlsx`](Phuong_xa_Viet_Nam.xlsx) | `TÊN PHƯỜNG XÃ` (cột B) |
| Số nhà / đường | `address1` | — (tự do) | — |

### Vì sao phải khớp?

Giá trị `state` và `city` **không phải nhập tự do** — BKNS/VNNIC chỉ chấp nhận **đúng tên
chuẩn** trong danh mục hành chính. Nếu gửi sai chính tả, sai dấu, hoặc tên cũ (đã sáp nhập)
thì order có thể bị từ chối hoặc lưu sai chủ thể. Vì vậy hệ thống của bạn **phải cho người
dùng chọn từ danh mục** (dropdown/autocomplete) lấy từ 2 file Excel này, **không cho gõ tay**.

### Cấu trúc 2 file

**`Tinh_thanh_Viet_Nam.xlsx`** — 34 tỉnh/thành (hàng 2→35):

| Cột A — `MÃ TỈNH THÀNH` | Cột B — `TÊN TỈNH THÀNH` |
|---|---|
| 1 | Thành phố Hà Nội |
| 4 | Tỉnh Cao Bằng |
| 8 | Tỉnh Tuyên Quang |
| … | … |

**`Phuong_xa_Viet_Nam.xlsx`** — 3321 phường/xã (hàng 2→3322):

| Cột A — `MÃ PHƯỜNG XÃ` | Cột B — `TÊN PHƯỜNG XÃ` | Cột C — `MÃ TỈNH THÀNH` | Cột D — `TÊN TỈNH THÀNH` |
|---|---|---|---|
| 4  | Phường Ba Đình | 1 | Thành phố Hà Nội |
| 8  | Phường Ngọc Hà | 1 | Thành phố Hà Nội |
| 70 | Phường Hoàn Kiếm | 1 | Thành phố Hà Nội |
| … | … | … | … |

> File phường/xã **đã chứa sẵn cột tỉnh/thành** (C, D) → quan hệ cha–con. Dùng cột C
> (`MÃ TỈNH THÀNH`) để lọc danh sách phường/xã theo tỉnh đã chọn.

### Quy tắc bắt buộc khi build form / payload

1. **Chọn Tỉnh/Thành trước** từ `Tinh_thanh_Viet_Nam.xlsx` → gán `state` = **đúng** chuỗi cột
   B (giữ nguyên dấu, vd `"Thành phố Hà Nội"`, không phải `"Ha Noi"`).
2. **Lọc Phường/Xã** trong `Phuong_xa_Viet_Nam.xlsx` theo `MÃ TỈNH THÀNH` (cột C) = mã tỉnh
   vừa chọn → người dùng chọn phường/xã → gán `city` = **đúng** chuỗi cột B (vd
   `"Phường Hoàng Mai"`).
3. `address1` = phần còn lại (số nhà, ngõ, đường) — tự do.
4. **So khớp chính xác từng ký tự** (kể cả tiền tố "Thành phố" / "Tỉnh" / "Phường" / "Xã" và
   dấu tiếng Việt). Khuyến nghị lưu cả **mã** (cột A) trong hệ thống của bạn để đối chiếu ổn
   định khi danh mục đổi tên.

### Ví dụ khớp đúng (theo fixture thật)

```jsonc
// Người dùng ở: Phường Hoàng Mai, Thành phố Hà Nội, số nhà 26D1B
{
  "country":  "VN",
  "state":    "Thành phố Hà Nội",   // ← cột B của Tinh_thanh_Viet_Nam.xlsx (mã 1)
  "city":     "Phường Hoàng Mai",   // ← cột B của Phuong_xa_Viet_Nam.xlsx (thuộc mã tỉnh 1)
  "address1": "26D1B"               // ← phần địa chỉ chi tiết, tự do
}
```

### Ánh xạ field (đừng nhầm tên)

| Ý niệm | WHMCS registrant field | BKNS field | Nguồn dữ liệu hợp lệ |
|---|---|---|---|
| Tỉnh/Thành | `state` | `state` | `Tinh_thanh_Viet_Nam.xlsx` cột B |
| Phường/Xã | `city` | `city` | `Phuong_xa_Viet_Nam.xlsx` cột B |
| Địa chỉ chi tiết | `address1` | `address1` | tự do |

> ⚠️ Tên field WHMCS gây hiểu nhầm: WHMCS gọi là `city`/`state` nhưng **giá trị** phải là
> **phường/xã** và **tỉnh/thành** theo danh mục VN. Module map thẳng
> `state→state`, `city→city`, `address1→address1` (xem `bknsdomains_build_contacts()`),
> nên trách nhiệm điền đúng danh mục thuộc về **form nhập liệu**.

---

## 8. Validate `.vn` trước khi tạo order

Để tránh tạo order rác ở BKNS, **validate phía client trước** (logic
`bknsdomains_validate_vn_registrant()`):

| Điều kiện | Áp dụng | Lỗi nếu sai |
|---|---|---|
| Email hợp lệ (`FILTER_VALIDATE_EMAIL`) | mọi hình thức | "Email chủ thể không hợp lệ…" |
| Điện thoại ≥ 8 chữ số | mọi hình thức | "Số điện thoại chủ thể không hợp lệ…" |
| `taxid` ≥ 10 chữ số | tổ chức (`org`) | "Mã số thuế không hợp lệ…" |
| `nationalid` 9–12 chữ số | cá nhân (`ind`) | "CMND/CCCD không hợp lệ…" |
| `gender` không rỗng | cá nhân | "Vui lòng chọn giới tính…" |
| `birthday` `YYYY-MM-DD`, không tương lai, không quá 100 năm | cá nhân | "Ngày sinh không hợp lệ…" |

Chỉ chạy khi tên miền kết thúc bằng `.vn` (`bknsdomains_is_vn_domain()`); tên miền quốc tế bỏ
qua các ràng buộc này.

---

## 9. Thanh toán / số dư

```
register / transfer (domain/order)   ─┐
renew (domain/{id}/renew)            ─┴─► TỰ ĐỘNG trừ credit khi tạo order
```

**Luồng chuẩn:**
1. `GET balance` → tính `available = acc_credit − acc_balance`.
2. Nếu `available <= 0` → **dừng**, báo "số dư không đủ" (đừng tạo order treo).
   - *Fail-open:* nếu `GET balance` lỗi (mạng/API), module **không chặn** mà vẫn cho đi tiếp.
3. Tạo order (`domain/order` hoặc `domain/{id}/renew`) — credit bị trừ tự động.
4. **Không** gọi `invoice/{id}/credit` cho 3 nghiệp vụ trên.

> Môi trường OTE không hỗ trợ thanh toán credit thật → khi test, bỏ qua/đừng kỳ vọng bước
> trừ tiền.

---

## 10. Những gì KHÔNG hỗ trợ

| Chức năng | Lý do |
|---|---|
| **Lấy mã EPP** (`GetEPPCode`) | BKNS userapi **không có** endpoint trả eppcode. Module cố ý không khai hàm này. |
| **Đổi thông tin chủ thể `.vn`** | VNNIC **không** cho đổi chủ thể trực tiếp qua API. `SaveContactDetails` trả thông báo yêu cầu liên hệ BKNS làm thủ tục. |
| `POST domain/contacts` (đổi contact quốc tế) | ⚠️ **CHƯA xác nhận** endpoint/payload — phải hỏi BKNS trước khi dùng cho tên miền quốc tế. |

---

## 11. Tự viết module tối thiểu (mẫu)

Khung client tối giản (ngôn ngữ bất kỳ — đây là PHP) để bạn dựng SDK riêng:

```php
final class BknsClient
{
    private string $base;
    private ?string $token = null;

    public function __construct(bool $test = false)
    {
        $this->base = $test
            ? 'https://id-ote.bkns.vn/api/'
            : 'https://my.bkns.net/api/';
    }

    /** Đăng nhập, lưu JWT. */
    public function login(string $user, string $pass): void
    {
        $rs = $this->call('POST', 'login', compact('user', 'pass') + [
            'username' => $user, 'password' => $pass,
        ], false);
        $this->token = $rs['token'] ?? null;
        if (!$this->token) {
            throw new RuntimeException('Login failed');
        }
    }

    /** Số dư khả dụng. */
    public function available(): float
    {
        $d = $this->call('GET', 'balance')['details'] ?? [];
        return (float)($d['acc_credit'] ?? 0) - (float)($d['acc_balance'] ?? 0);
    }

    /** Đăng ký tên miền. Trả BKNS domain id. */
    public function register(string $domain, int $years, array $registrant, array $ns): int
    {
        $tld = explode('.', $domain, 2)[1] ?? '';
        $rs = $this->call('POST', 'domain/order', [
            'domain' => $domain, 'name' => $domain, 'tld' => $tld,
            'years' => $years, 'action' => 'register', 'pay_method' => 300,
            'nameservers' => array_values(array_filter($ns)),
            'registrant' => $registrant,
            'admin' => $registrant, 'tech' => $registrant, 'billing' => $registrant,
        ]);
        foreach ($rs['items'] ?? [] as $it) {
            if (($it['name'] ?? '') === $domain && !empty($it['id'])) {
                return (int)$it['id'];
            }
        }
        throw new RuntimeException('Register failed: ' . json_encode($rs));
    }

    /** Tra id đúng theo tên (ưu tiên Active) — gọi TRƯỚC khi renew. */
    public function resolveId(string $domain): ?int
    {
        $list = $this->call('GET', 'domain/name/' . $domain)['domains'] ?? [];
        usort($list, fn($a, $b) =>
            (($b['status'] ?? '') === 'Active') <=> (($a['status'] ?? '') === 'Active')
            ?: strtotime($b['expires'] ?? '0') <=> strtotime($a['expires'] ?? '0'));
        return isset($list[0]['id']) ? (int)$list[0]['id'] : null;
    }

    public function renew(int $id, int $years): array
    {
        return $this->call('POST', "domain/$id/renew", [
            'years' => (string)$years, 'pay_method' => 300,
        ]);
    }

    /** HTTP wrapper. */
    private function call(string $method, string $ep, array $data = [], bool $auth = true): array
    {
        $url = $this->base . ltrim($ep, '/');
        $headers = ['Accept: application/json'];
        if ($auth) { $headers[] = 'Authorization: Bearer ' . $this->token; }

        $ch = curl_init();
        $opt = [
            CURLOPT_RETURNTRANSFER => true, CURLOPT_TIMEOUT => 30,
            CURLOPT_SSL_VERIFYPEER => true, CURLOPT_SSL_VERIFYHOST => 2,
        ];
        if ($method === 'GET') {
            if ($data) { $url .= '?' . http_build_query($data); }
        } else {
            $opt[CURLOPT_CUSTOMREQUEST] = $method;
            $opt[CURLOPT_POSTFIELDS]    = $data ? json_encode($data) : '{}';
            $headers[] = 'Content-Type: application/json';
        }
        $opt[CURLOPT_URL] = $url;
        $opt[CURLOPT_HTTPHEADER] = $headers;
        curl_setopt_array($ch, $opt);

        $raw  = curl_exec($ch);
        $code = (int)curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);

        $rs = json_decode($raw, true) ?: [];
        if ($code < 200 || $code >= 300 || !empty($rs['error'])) {
            throw new RuntimeException("API $ep failed (HTTP $code): " . ($rs['error'] ?? $raw));
        }
        return $rs;
    }
}
```

**Cách dùng:**
```php
$bk = new BknsClient(test: true);
$bk->login('email-api@example.com', 'matkhau-api');

if ($bk->available() <= 0) { exit('Số dư không đủ'); }

$registrant = [
    'type' => 'ind', 'firstname' => 'Demo', 'lastname' => 'Nguyễn Văn',
    'email' => 'demo@gmail.com', 'phonenumber' => '+84-96462222323',
    'nationalid' => '00109003333', 'gender' => 'Male', 'birthday' => '1999-05-04',
    'country' => 'VN',
    'state'   => 'Thành phố Hà Nội',   // khớp Tinh_thanh_Viet_Nam.xlsx
    'city'    => 'Phường Hoàng Mai',   // khớp Phuong_xa_Viet_Nam.xlsx
    'address1' => '199 Giáp Bát',
    '__nocontact' => true,
];

$id = $bk->register('vidu.com.vn', 1, $registrant, ['ns1.bkdns.vn', 'ns2.bkdns.vn']);
// ... sau này gia hạn:
$id = $bk->resolveId('vidu.com.vn');   // tra lại id đúng (Active) trước
$bk->renew($id, 1);
```

---

## 12. Checklist tích hợp

- [ ] Chọn đúng base URL (OTE để test, production để chạy thật).
- [ ] Login lấy token, cache lại; mọi request gắn `Authorization: Bearer`.
- [ ] Kiểm tra **HTTP 2xx + body** (không tin mỗi body).
- [ ] Body POST rỗng gửi `{}`, không gửi `[]`.
- [ ] `GET balance` trước mọi nghiệp vụ tốn tiền; **không** gọi `invoice/credit` cho register/renew/transfer.
- [ ] Thu thập đủ trường `.vn` (cá nhân: CMND/giới tính/ngày sinh; tổ chức: MST/tên).
- [ ] **`state` khớp `Tinh_thanh_Viet_Nam.xlsx` (cột B); `city` khớp `Phuong_xa_Viet_Nam.xlsx` (cột B);** dùng dropdown, không gõ tay; giữ nguyên dấu tiếng Việt.
- [ ] Lọc phường/xã theo `MÃ TỈNH THÀNH` (cột C của file phường/xã) = mã tỉnh đã chọn.
- [ ] Lưu BKNS domain **id** (`items[].id`) sau đăng ký.
- [ ] **Trước khi renew**, tra lại id đúng theo tên (ưu tiên `Active`) để khỏi gia hạn nhầm.
- [ ] Giữ `__nocontact => true`.
- [ ] Không kỳ vọng EPP / đổi chủ thể `.vn` qua API.

---

*Tài liệu trích từ module `bknsdomains` 1.1.0. Khi BKNS bổ sung/đổi endpoint, cập nhật lại
file này và gắn nhãn độ tin cậy tương ứng. Nguyên tắc: **thiếu thì hỏi, không tự suy luận.***
