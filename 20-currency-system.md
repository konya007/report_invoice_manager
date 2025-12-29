# 20 - Hệ Thống Tiền Tệ & Tỷ Giá

> Quản lý đa tiền tệ và tự động đồng bộ tỷ giá ngân hàng.

---

## 📋 Tổng quan

Hệ thống hỗ trợ nhập/xuất hàng hóa bằng nhiều loại tiền tệ (USD, EUR, JPY...) nhưng luôn quy đổi và hạch toán về đồng tiền cơ sở (VND) để báo cáo tài chính.

**Điểm đặc biệt:**
- **Real-time Sync:** Tự động lấy tỷ giá Vietcombank 10 phút/lần.
- **Snapshot History:** Lưu trữ tỷ giá tại thời điểm giao dịch. Nếu thị trường biến động sau đó, giá trị hóa đơn cũ **không đổi**.
- **Smart Deduplication:** Chỉ lưu bản ghi mới nếu tỷ giá thay đổi, tiết kiệm 90% dung lượng DB.

---

## 🗄️ Database Schema

### Bảng `transfer_currencies`

Lưu trữ lịch sử tỷ giá.

| Column | Type | Mô tả |
|--------|------|-------|
| `id` | BIGINT | PK |
| `exchange_date` | DATETIME | Thời điểm lấy tỷ giá Snapshot |
| `source` | VARCHAR | Nguồn (Default: 'Vietcombank') |
| `rates` | JSON | Lưu trữ NoSQL dạng: `{"USD": 25450, "EUR": 27100}` |

### Tại sao dùng JSON?
- Linh hoạt: Có thể thêm bất kỳ loại tiền tệ mới nào mà không cần sửa cấu trúc bảng (Migration).
- Nhanh: Truy xuất toàn bộ bảng tỷ giá trong 1 query.

---

## 🚀 Core Services

### 1. `CurrencyExchangeService`

Mạch máu của hệ thống tiền tệ. Nằm tại `app/Services/App/CurrencyExchangeService.php`.

#### a. Fetching Logic (Vietcombank Integration)
- **Endpoint:** `https://portal.vietcombank.com.vn/Usercontrols/TVPortal.TyGia/pXML.aspx`
- **Format:** XML
- **Parser:** Sử dụng `DOMDocument` và `XPath` để parse XML.

```xml
<ExrateList>
    <DateTime>12/24/2024 8:30:00 AM</DateTime>
    <Exrate CurrencyCode="USD" CurrencyName="DO LA MY" Buy="25140" Transfer="25170" Sell="25510"/>
    ...
</ExrateList>
```

Hệ thống sẽ lấy giá trị **Transfer** (Chuyển khoản) làm chuẩn để tính toán.

#### b. Deduplication Strategy (Chống trùng lặp)
Trước khi lưu tỷ giá mới, hệ thống kiểm tra:
1. Tìm bản ghi mới nhất trong ngày.
2. So sánh mảng `rates` của bản ghi đó với dữ liệu vừa fetch.
3. Nếu **GIỐNG HỆT** (sai số < 0.01) -> Bỏ qua, trả về bản ghi cũ.
4. Nếu **KHÁC** -> Tạo bản ghi `TransferCurrency` mới.

#### c. Conversion Logic

```php
public function convertToVND(float $amount, string $currency, ?TransferCurrency $transferRate): float
{
    if ($currency === 'VND') return $amount;
    
    // Fallback: Nếu không có tỷ giá chỉ định, lấy tỷ giá mới nhất
    $rateObj = $transferRate ?? $this->getLatestRate();
    $rate = $rateObj->rates[$currency] ?? 0;
    
    return $amount * $rate;
}
```

---

## 🛠 Integration Points

### 1. Hóa đơn mua (Purchase Invoice)
- Khi tạo hóa đơn USD, hệ thống tự động gọi `getTransferCurrencyForInvoice()`.
- ID của bản ghi tỷ giá được lưu vào `purchase_invoices.transfer_currency_id`.
- **Bảo toàn dữ liệu:** Dù tỷ giá ngày mai tăng gấp đôi, hóa đơn hôm nay vẫn dùng ID cũ -> Giá trị nhập kho không bị sai lệch.

### 2. Báo cáo tài chính
- Doanh thu/Lợi nhuận được tính bằng cách: `SUM(amount * saved_rate)`.

---

## ⚙️ Configuration

Cấu hình trong `.env` (Hiện tại đang hardcode URL trong service, có thể refactor ra file config).

- **Timeout:** 10s (Tránh treo ứng dụng nếu Vietcombank sập).
- **Cache:** Có thể bật cache Redis nếu tần suất gọi quá cao.

---

## 📚 Related Files

| File | Type | Mô tả |
|------|------|-------|
| `app/Services/App/CurrencyExchangeService.php` | Service | Logic chính fetch & parse |
| `app/Models/TransferCurrency.php` | Model | Eloquent model với JSON cast |
| `app/Services/Utils/FormatService.php` | Helper | Format tiền tệ (VD: 1,000,000 ₫) |
