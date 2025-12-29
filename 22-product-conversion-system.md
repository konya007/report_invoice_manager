# Product Conversion System - Documentation

## 📋 Tổng quan

Hệ thống chuyển đổi sản phẩm (Conversion System) cho phép biến đổi một sản phẩm từ dạng này sang dạng khác (Ví dụ: Thép cuộn -> Thép tấm, hoặc Gói lớn -> Gói nhỏ). Quá trình này tự động xử lý kho và tính toán lại giá vốn cho sản phẩm mới.

**Tính năng chính:**
- Chuyển đổi 1-1 hoặc 1-n (tùy ngữ cảnh sử dụng).
- Tự động trừ kho nguyên liệu (Source).
- Tự động cộng kho thành phẩm (Destination).
- Tính toán đơn giá vốn bình quân cho thành phẩm dựa trên giá nguyên liệu.

---

## 🗄️ Database Schema

### Bảng `conversion_items`

Lưu trữ thông tin chi tiết mỗi lần chuyển đổi.

| Column | Type | Mô tả |
|--------|------|-------|
| `company_id` | BIGINT | Tenant ID |
| `purchase_item_id`| BIGINT | ID của nguyên liệu gốc (Lô hàng nhập) |
| `before_product_id`| BIGINT | ID sản phẩm trước khi đổi |
| `after_product_id` | BIGINT | ID sản phẩm sau khi đổi (Tạo mới hoặc lấy có sẵn) |
| `quantity_before` | DECIMAL | Số lượng đem đi đổi |
| `quantity_after` | DECIMAL | Số lượng thu được |
| `unit_price_before`| DECIMAL | Giá vốn nguyên liệu |
| `unit_price_after` | DECIMAL | Giá vốn thành phẩm (Được tính toán lại) |

---

## 🚀 Logic Xử lý

### 1. Quy trình chuyển đổi (`ProductConversionService`)

Khi thực hiện chuyển đổi (`createConversion`):

1. **Validate:** Kiểm tra tồn kho của lô hàng nguyên liệu (`purchase_item_id`).
2. **Calculate Price:**
   - `Total Cost` = `Quantity Before` * `Unit Price Before`
   - `Unit Price After` = `Total Cost` / `Quantity After`
   - *Nguyên tắc bảo toàn giá trị:* Tổng giá trị hàng hóa không đổi, chỉ thay đổi hình thức và đơn giá đơn vị.
3. **Transaction Creating:**
   - Tạo transaction `conversion_out` cho nguyên liệu (Giảm kho).
   - Tạo transaction `conversion_in` cho thành phẩm (Tăng kho).
   
### 2. Tạo sản phẩm mới

Nếu sản phẩm đích (`after_product_id`) chưa tồn tại, hệ thống hỗ trợ tạo nhanh sản phẩm mới với mã tự sinh (`P-YYMMDD...`) ngay trong quá trình chuyển đổi.

---

## 📚 Related Files

| File | Mô tả |
|------|-------|
| `app/Services/Utils/ProductConversionService.php` | Service chính xử lý logic chuyển đổi |
| `app/Models/ConversionItem.php` | Model lưu lịch sử chuyển đổi |
| `app/Models/InventoryTransaction.php` | Model lưu biến động kho (type `conversion_in`/`out`) |
| `app/Livewire/Main/Inventory/ProductConversionForm.php` | Giao diện người dùng |
