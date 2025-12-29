# Warranty System - Documentation

## 📋 Tổng quan

Hệ thống bảo hành và hạn sử dụng cho phép theo dõi thời gian bảo hành (Warranty) và hạn sử dụng (Expiry) cho từng mặt hàng trong hóa đơn mua/bán.

**Tính năng chính:**
- Thiết lập số tháng bảo hành cho từng sản phẩm.
- Thiết lập số tháng hạn sử dụng (Date code/Expiry).
- Tự động tính ngày hết hạn dựa trên ngày mua hàng.
- Cảnh báo hoặc hiển thị thông tin bảo hành trên hóa đơn.

---

## 🗄️ Database Schema

Thông tin bảo hành được lưu trực tiếp trong bảng `purchase_items` (đối với hàng nhập) và có thể được tham chiếu lại khi bán.

### Bảng `purchase_items`

| Column | Type | Mô tả |
|--------|------|-------|
| `warranty_months` | INT | Số tháng bảo hành (Nullable) |
| `expiry_months` | INT | Số tháng hạn sử dụng (Nullable) |

**Logic tính toán:**
- **Ngày hết hạn bảo hành** = `Ngày mua hàng (Purchase Date)` + `warranty_months`
- **Ngày hết hạn sử dụng** = `Ngày mua hàng (Purchase Date)` + `expiry_months`

---

## 🚀 Cách sử dụng

### 1. Cập nhật thông tin bảo hành

Thông tin bảo hành được cập nhật thông qua **Modal Cập nhật Bảo hành** (`WarrantyUpdateModal`).

**Quy trình:**
1. Người dùng mở chi tiết hóa đơn mua hàng hoặc danh sách sản phẩm.
2. Chọn hành động "Cập nhật bảo hành" cho một dòng sản phẩm (`PurchaseItem`).
3. Nhập số tháng bảo hành và số tháng hạn sử dụng.
4. Hệ thống tự động hiển thị ngày hết hạn dự kiến.
5. Lưu lại thông tin.

### 2. Logic xử lý (Frontend)

File: `app/Livewire/Main/Products/WarrantyUpdateModal.php`

- **Input:** 
  - `warrantyMonths`: Số nguyên (0-1200)
  - `expiryMonths`: Số nguyên (0-1200)
- **Validation:**
  - `min:0`, `max:1200` (100 năm)
  - Kiểm tra quyền truy cập công ty (`company_id`).
- **Calculation:**
  - Sử dụng `Carbon` để cộng số tháng vào `purchase_date` của hóa đơn gốc.

```php
// Ví dụ logic tính toán
$purchaseDate = \Carbon\Carbon::parse($this->purchaseDate);
$warrantyExpiry = $purchaseDate->copy()->addMonths($this->warrantyMonths);
```

---

## ⚙️ Configuration

- **Giới hạn tối đa:** 1200 tháng (tương đương 100 năm).
- **Quyền hạn:** Yêu cầu quyền sửa đổi sản phẩm hoặc hóa đơn mua hàng.

---

## 📚 Related Files

| File | Mô tả |
|------|-------|
| `app/Livewire/Main/Products/WarrantyUpdateModal.php` | Component xử lý logic cập nhật |
| `resources/views/livewire/main/products/warranty-update-modal.blade.php` | Giao diện Modal |
| `app/Models/PurchaseItem.php` | Model lưu trữ dữ liệu |
