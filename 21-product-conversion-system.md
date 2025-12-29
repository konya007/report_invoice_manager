# 21 - Hệ Thống Chuyển Đổi Sản Phẩm

> Xử lý quy trình lắp ráp (Assembly), tháo dỡ (Disassembly) và chuyển đổi đơn vị tính.

---

## 📋 Tổng quan

Trong thực tế kinh doanh, sản phẩm nhập vào và bán ra thường không giống nhau 100%.
- **Ví dụ:** Nhập "Cuộn thép 1 tấn" -> Cắt ra bán "Thép tấm 100kg".
- **Ví dụ:** Nhập "Ram giấy" -> Bán "Tờ".

Hệ thống chuyển đổi giải quyết bài toán này bằng cách tự động hóa quy trình: **Xuất kho nguyên liệu -> Nhập kho thành phẩm**.

---

## 🗄️ Database Schema

### Bảng `conversion_items`

Lưu vết (Traceability) của quá trình chuyển đổi.

| Column | Type | Mô tả |
|--------|------|-------|
| `id` | BIGINT | PK |
| `purchase_item_id` | BIGINT | **Source:** Lô nguyên liệu gốc (để truy xuất giá vốn FIFO) |
| `before_product_id`| BIGINT | ID sản phẩm NGUYÊN LIỆU |
| `after_product_id` | BIGINT | ID sản phẩm THÀNH PHẨM |
| `quantity_before` | DECIMAL | Số lượng nguyên liệu mất đi |
| `quantity_after` | DECIMAL | Số lượng thành phẩm tạo thành |
| `unit_price_after` | DECIMAL | **Giá vốn mới** (Tính toán tự động) |

### Transaction integration
Một lần chuyển đổi sẽ sinh ra 2 records trong bảng `inventory_transactions`:
1. `conversion_out`: Trừ kho nguyên liệu (Link tới `conversion_item_id`).
2. `conversion_in`: Cộng kho thành phẩm (Link tới `conversion_item_id`).

---

## 🚀 Logic Cốt Lõi (`ProductConversionService`)

### 1. Thuật toán bảo toàn giá trị (Value Preservation)

Nguyên tắc bất di bất dịch: **Tổng giá trị vốn không đổi**, chỉ có đơn giá đơn vị thay đổi.

**Công thức:**
```
Total Cost = Quantity_In * Unit_Cost_In
Unit_Cost_Out = Total Cost / Quantity_Out
```

**Code implementation:**
`app/Services/Utils/ProductConversionService.php`

```php
public function calculateUnitPrice(int $sourceItemId, float $qtyIn, float $qtyOut): float
{
    $sourceItem = PurchaseItem::findOrFail($sourceItemId);
    $costBase = $sourceItem->unit_cost_gross; // Giá vốn gốc (hoặc giá nhập)
    
    // Tính tổng giá trị đem đi chuyển đổi
    $totalValue = $costBase * $qtyIn;
    
    // Chia đều cho số lượng thành phẩm
    return round($totalValue / $qtyOut, 4);
}
```

### 2. Tự động sinh mã sản phẩm (`generateProductCode`)

Khi chuyển đổi sang một sản phẩm hoàn toàn mới (chưa có trong danh mục), hệ thống hỗ trợ tạo nhanh ngay tại form chuyển đổi.

- **Pattern:** `P-{YYMMDDHHmmss}{Random3}`
- **Ví dụ:** `P-241225093011999`
- **Mục đích:** Đảm bảo unique code và có timestamp để biết thời điểm tạo.

### 3. Quy trình thực hiện (Transactional)

Mọi thao tác được gói trong `DB::transaction()` để đảm bảo tính toàn vẹn dữ liệu:

1. Validate tồn kho nguyên liệu (Có đủ để chuyển đổi không?).
2. Tạo/Lấy sản phẩm đích.
3. Tính giá vốn mới.
4. Tạo record `conversion_items`.
5. Tạo transaction `conversion_out` (Trừ kho).
6. Tạo transaction `conversion_in` (Cộng kho).

Nếu bất kỳ bước nào lỗi -> Rollback toàn bộ.

---

## 💡 Use Cases

### Case 1: Chia nhỏ (Splitting)
- **Input:** 1 Hộp bánh (Giá vốn 100k).
- **Output:** 10 Cái bánh lẻ.
- **Kết quả:** Kho +10 cái bánh, giá vốn 10k/cái.

### Case 2: Đóng gói (Bundling)
- **Input:** 10 Cái bánh lẻ (Giá vốn 10k/cái).
- **Output:** 1 Hộp bánh.
- **Kết quả:** Kho +1 hộp bánh, giá vốn 100k/hộp.

---

## 📚 Related Files

| File | Mô tả |
|------|-------|
| `app/Services/Utils/ProductConversionService.php` | Core Logic |
| `app/Livewire/Main/Inventory/ProductConversionForm.php` | UI xử lý form |
| `app/Models/ConversionItem.php` | Model |
