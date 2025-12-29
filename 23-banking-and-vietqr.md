# Banking & VietQR System - Documentation

## 📋 Tổng quan

Hệ thống Ngân hàng tích hợp VietQR API để cung cấp danh sách ngân hàng chính xác, cập nhật nhất tại Việt Nam. Hệ thống hỗ trợ tìm kiếm, tra cứu và tự động cache để tối ưu hiệu suất.

**Tính năng chính:**
- Lấy danh sách ngân hàng chuẩn từ VietQR (hơn 50+ ngân hàng).
- Tìm kiếm thông minh theo Tên, Mã, Tên viết tắt.
- Cache dữ liệu để giảm phụ thuộc vào đường truyền mạng và API bên ngoài.

---

## 🚀 Cách sử dụng

### 1. Lấy danh sách ngân hàng

```php
use App\Services\Utils\VietQRService;

$service = new VietQRService();
$banks = $service->getBanks(); // Returns Collection
```

### 2. Tìm kiếm ngân hàng

Hỗ trợ tìm kiếm mờ (fuzzy search) và tìm kiếm chính xác.

```php
// Tìm chính xác theo mã (VD: VCB, MB, ICB)
$bank = $service->findByCode('VCB');

// Tìm theo tên (VD: "Vietcom", "Quân đội")
$results = $service->searchByName('Ngoại thương');
```

---

## ⚙️ Configuration & Caching

### API & Cache Logic
- **Endpoint:** `https://api.vietqr.io/v2/banks`
- **Cache Key:** `vietqr.banks`
- **TTL (Time-To-Live):** Mặc định 24 giờ (`1440` phút).
- **Fallback:** Nếu API lỗi, hệ thống sẽ trả về dữ liệu cũ từ cache hoặc danh sách rỗng, đảm bảo app không bị crash.

### Customizing TTL
Người dùng cấp công ty có thể cấu hình thời gian làm mới cache thông qua `system_settings.cache_refresh_minutes` trong cài đặt công ty.

---

## 📚 Related Files

| File | Mô tả |
|------|-------|
| `app/Services/Utils/VietQRService.php` | Service chính xử lý API và Cache |
| `resources/views/livewire/main/settings/bank-settings.blade.php` | Giao diện cấu hình ngân hàng (nếu có) |
