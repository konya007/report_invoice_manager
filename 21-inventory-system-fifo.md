# Inventory System - Documentation

## 📋 Tổng quan

Hệ thống kho (Inventory System) hoạt động dựa trên nguyên tắc **Transaction-based Ledger** thay vì chỉ lưu số lượng tồn kho tĩnh. Mọi biến động kho đều được ghi nhận qua các giao dịch (Transactions).

**Nguyên lý cốt lõi:**
- **FIFO (First-In, First-Out):** Hàng nhập trước xuất trước. Hệ thống tự động phân bổ hàng xuất từ các lô nhập cũ nhất còn tồn.
- **Truy xuất nguồn gốc:** Mọi hàng xuất ra đều biết chính xác từ lô nhập (`purchase_item_id`) nào.
- **Chính xác giá vốn:** Do phân bổ chính xác theo lô nhập, giá vốn hàng bán (COGS) được tính chính xác theo từng lô.

---

## 🗄️ Database Schema

### Bảng `inventory_transactions`

Bảng trung tâm lưu tất cả lịch sử xuất nhập.

| Column | Type | Mô tả |
|--------|------|-------|
| `id` | BIGINT | Primary Key |
| `company_id` | BIGINT | Multi-tenancy |
| `product_id` | BIGINT | Sản phẩm |
| `transaction_type`| VARCHAR | Loại giao dịch (Xem bên dưới) |
| `quantity` | DECIMAL(4) | Số lượng (Luôn dương, dấu phụ thuộc loại GD khi tính toán) |
| `purchase_item_id`| BIGINT | **Reference quan trọng nhất** - Link tới lô hàng nhập gốc |
| `sale_item_id` | BIGINT | Link tới item bán (nếu là xuất bán) |
| `approved_at` | DATETIME | Thời điểm giao dịch có hiệu lực |

### Transaction Types

| Type | Ý nghĩa | Tác động tồn kho |
|------|---------|------------------|
| `purchase` | Nhập hàng mua | Tăng (+) |
| `sale` | Xuất bán | Giảm (-) |
| `conversion_in` | Nhập do chuyển đổi/lắp ráp | Tăng (+) |
| `conversion_out` | Xuất do chuyển đổi/tháo dỡ | Giảm (-) |
| `undetermined` | Nhập không xác định (tự SX) | Tăng (+) |
| `deleted` | Xóa sổ (hủy) | Giảm (-) |

---

## 🚀 Logic Xử lý (Core Algorithms)

### 1. Tính tồn kho (`InventoryStockService`)

Tồn kho không được lưu cứng mà được tính toán (aggregate) từ bảng transactions:

`Stock = SUM(Inside Types) - SUM(Outside Types)`

- **Inside:** `purchase`, `conversion_in`, `undetermined`
- **Outside:** `sale`, `conversion_out`, `deleted`

```php
// SQL Logic tương đương
SELECT product_id, 
       SUM(CASE 
           WHEN type IN ('purchase', 'conversion_in') THEN quantity 
           ELSE -quantity 
       END) as stock
FROM inventory_transactions
GROUP BY product_id
```

### 2. Phân bổ hàng xuất - FIFO (`SaleItemAllocationService` / `InventoryTransactionLogger`)

Khi xuất bán (`logSaleItem`), hệ thống thực hiện:
1. Tìm tất cả các giao dịch nhập (`purchase`/`conversion_in`) của sản phẩm đó, sắp xếp theo thời gian (`approved_at ASC`).
2. Tính số lượng còn lại của từng giao dịch nhập (Số lượng nhập - Số lượng đã xuất của lô đó).
3. Trừ dần số lượng cần xuất vào các lô nhập khả dụng (Allocate).
4. Tạo các transaction `sale` mới link tới đúng `purchase_item_id` tương ứng.

**Ví dụ:**
- Lô A (01/01): Nhập 10
- Lô B (05/01): Nhập 20
- **Bán 15 cái:**
  -> Hệ thống lấy 10 từ Lô A (hết Lô A)
  -> Lấy 5 từ Lô B (Lô B còn 15)
  -> Tạo 2 dòng transaction `sale`: 1 dòng 10 (link Lô A), 1 dòng 5 (link Lô B).

---

## 📚 Related Files

| File | Mô tả |
|------|-------|
| `app/Services/Utils/InventoryStockService.php` | Tính toán tồn kho, báo cáo tồn kho chi tiết |
| `app/Services/Utils/InventoryTransactionLogger.php` | Ghi nhận giao dịch, xử lý logic FIFO |
| `app/Models/InventoryTransaction.php` | Model chính |
| `app/Services/Utils/SaleItemAllocationService.php` | Helper phân bổ hàng (nếu tách riêng) |
