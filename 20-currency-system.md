# Currency System - Documentation

## 📋 Tổng quan

Hệ thống tiền tệ (Currency System) quản lý việc chuyển đổi tỷ giá ngoại tệ, tự động cập nhật tỷ giá từ Vietcombank và lưu trữ lịch sử tỷ giá để phục vụ tính toán tài chính chính xác cho các hóa đơn và báo cáo.

**Tính năng chính:**
- Tự động lấy tỷ giá từ API Vietcombank.
- Lưu trữ lịch sử tỷ giá theo thời gian thực (snapshot).
- Hỗ trợ chuyển đổi tiền tệ (Convert to VND).
- Gắn tỷ giá cụ thể vào từng hóa đơn/sản phẩm để đảm bảo tính lịch sử (không bị thay đổi khi tỷ giá thị trường biến động).

---

## 🗄️ Database Schema

### Bảng `transfer_currencies`

Lưu trữ snapshot tỷ giá tại một thời điểm cụ thể.

| Column | Type | Mô tả |
|--------|------|-------|
| `id` | BIGINT | Primary Key |
| `exchange_date` | DATETIME | Thời điểm áp dụng tỷ giá |
| `source` | VARCHAR | Nguồn dữ liệu (Mặc định: 'Vietcombank') |
| `rates` | JSON | Danh sách tỷ giá (VD: `{"USD": 25000, "EUR": 27000}`) |
| `created_at` | TIMESTAMP | Thời gian tạo |

### Quan hệ
- **PurchaseInvoice / PurchaseItem**: Có trường `transfer_currency_id` liên kết đến bảng này để chốt tỷ giá tại thời điểm nhập hàng.

---

## 🚀 Cách sử dụng

### 1. Tự động cập nhật tỷ giá

Service `CurrencyExchangeService` chịu trách nhiệm giao tiếp với Vietcombank.

- **API URL:** `https://portal.vietcombank.com.vn/Usercontrols/TVPortal.TyGia/pXML.aspx`
- **Tần suất:** Dữ liệu XML từ Vietcombank thường cập nhật nhiều lần trong ngày.
- **Logic:**
  1. Fetch XML từ Vietcombank.
  2. Parse XML lấy `DateTime` và danh sách `Exrate`.
  3. Kiểm tra xem đã có bản ghi `transfer_currencies` nào trùng `exchange_date` (phút) chưa.
  4. Nếu chưa -> Tạo bản ghi mới.
  5. Nếu có -> Tái sử dụng (tránh spam database).

### 2. Chuyển đổi tiền tệ

```php
use App\Services\App\CurrencyExchangeService;

$service = new CurrencyExchangeService();

// 1. Lấy tỷ giá hiện tại (tự động fetch hoặc lấy cache)
$transferCurrency = $service->getTransferCurrencyForInvoice();

// 2. Chuyển đổi sang VND
$vndAmount = $service->convertToVND(
    amount: 100, 
    fromCurrency: 'USD', 
    transferCurrency: $transferCurrency
);
```

---

## ⚙️ Configuration

- **Timeout:** 10 giây cho request đến Vietcombank.
- **Deduplication:** Hệ thống tự động so sánh tỷ giá để không tạo bản ghi trùng lặp nếu tỷ giá không đổi, giúp tiết kiệm dung lượng database.

---

## 📚 Related Files

| File | Mô tả |
|------|-------|
| `app/Services/App/CurrencyExchangeService.php` | Service chính xử lý logic fetch và convert |
| `app/Models/TransferCurrency.php` | Model lưu trữ lịch sử tỷ giá (JSON cast) |
| `app/Services/Utils/FormatService.php` | Helper định dạng hiển thị tiền tệ |
