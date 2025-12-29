# 22 - Hệ Thống Ngân Hàng & VietQR Integration

> Tích hợp API VietQR để chuẩn hóa dữ liệu ngân hàng và tạo mã QR thanh toán.

---

## 📋 Tổng quan

Hệ thống không lưu cứng danh sách ngân hàng trong database mà đồng bộ trực tiếp từ **VietQR API** (Cổng thanh toán quốc gia NAPAS). Điều này đảm bảo:
- Luôn cập nhật ngân hàng mới nhất.
- Logo, tên viết tắt, mã BIN chính xác tuyệt đối.
- Giảm thiểu maintenance thủ công.

---

## 🚀 Core Components

### 1. `VietQRService`

Service chịu trách nhiệm giao tiếp API và caching.
File: `app/Services/Utils/VietQRService.php`

#### a. Architecture
```mermaid
graph LR
    App -->|Get Banks| VietQRService
    VietQRService -->|Check| Cache[Redis/File Cache]
    Cache --Miss--> VietQRService
    VietQRService -->|Fetch API| VietQR[api.vietqr.io]
    VietQR -->|JSON| VietQRService
    VietQRService -->|Store (24h)| Cache
    VietQRService -->|Return| App
```

#### b. API Endpoint
- **URL:** `https://api.vietqr.io/v2/banks`
- **Method:** `GET`
- **Response Structure:**
  ```json
  {
    "code": "200",
    "desc": "Thành công",
    "data": [
      {
        "id": 17,
        "name": "Ngan hang TMCP Cong thuong Viet Nam",
        "code": "ICB",
        "bin": "970415",
        "shortName": "VietinBank",
        "logo": "https://img.vietqr.io/image/ICB-vietinbank-logo.png",
        "transferSupported": 1,
        "lookupSupported": 1
      },
      ...
    ]
  }
  ```

### 2. Intelligent Caching

Để tránh phụ thuộc 100% vào bên thứ 3 (tránh lỗi khi mạng chập chờn), hệ thống áp dụng Cache Layer 2 lớp:

1. **Layer 1 (Memory/Redis):** Cache dữ liệu trong 24 giờ (`1440` phút).
2. **Layer 2 (Fallback):** Nếu API chết, hệ thống sẽ cố gắng trả về dữ liệu cũ trong cache (Stale-while-revalidate) hoặc mảng rỗng có kiểm soát thay vì throw Exception làm sập app.

**Config:**
- `system_settings.cache_refresh_minutes`: Admin có thể điều chỉnh thời gian cache trong Cài đặt hệ thống.

### 3. Fuzzy Search Engine

Hệ thống tích hợp bộ tìm kiếm mờ ngay trong Service để hỗ trợ Dropdown chọn ngân hàng.

**Logic:** `searchByName($keyword)`
- Chuẩn hóa từ khóa (lowercase, trim).
- Quét qua 4 trường thông tin: `name`, `shortName`, `code`, `bin`.
- So sánh tương đối (`str_contains`).

**Ví dụ:** Tìm "quân đội" sẽ ra "MBBank" (Ngân hàng TMCP Quân Đội).

---

## 💡 Usage Examples

### Trong Livewire Component

```php
use App\Services\Utils\VietQRService;

public function mount(VietQRService $service)
{
    // Lấy list để bind vào Select box
    $this->banks = $service->getBanks();
}

public function updatedSearchBank($keyword)
{
    // Tìm kiếm live
    $this->banks = app(VietQRService::class)->searchByName($keyword);
}
```

### Tạo mã QR (Future Integration)
Hiện tại service tập trung vào lấy danh sách ngân hàng. Nền tảng đã sẵn sàng để tích hợp tạo mã QR thanh toán (VietQR Gen) bằng cách kết hợp:
- `BIN` ngân hàng.
- Số tài khoản.
- Số tiền.
- Nội dung.

---

## 📚 Related Files

| File | Mô tả |
|------|-------|
| `app/Services/Utils/VietQRService.php` | Service chính |
| `config/setting.php` | Default config cho cache time |
| `resources/views/livewire/main/settings/bank-settings.blade.php` | UI cài đặt |
