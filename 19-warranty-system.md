# 19 - Hệ Thống Bảo Hành & Hạn Sử Dụng

> Quản lý chu trình bảo hành và hạn sử dụng sản phẩm theo từng lô nhập hàng.

---

## 📋 Tổng quan

Hệ thống bảo hành được thiết kế để theo dõi **vòng đời sau bán hàng** của sản phẩm. Không giống như các hệ thống quản lý bảo hành theo Serial Number từng cái, hệ thống này quản lý theo **Lô hàng nhập (Purchase Item)**.

### Tại sao quản lý theo Purchase Item?
- **Đơn giản hóa:** Không cần nhập Serial cho hàng nghìn sản phẩm giá rẻ.
- **Linh hoạt:** Cùng một mã sản phẩm (SKU) có thể có thời hạn bảo hành khác nhau tùy theo đợt nhập hàng.
- **Tự động hóa:** Ngày hết hạn được tính tự động từ ngày mua của khách hàng cộng với chính sách của lô hàng đó.

---

## 🗄️ Database Schema

Thông tin cấu hình bảo hành được lưu trữ trực tiếp trong bảng `purchase_items`.

### Bảng `purchase_items`

| Column | Type | Default | Mô tả |
|--------|------|---------|-------|
| `warranty_months` | INT | NULL | Số tháng bảo hành (VD: 12, 24). Null = Không bảo hành. |
| `expiry_months` | INT | NULL | Hạn sử dụng (Shelf life) tính từ ngày nhập. |

**Mối quan hệ:**
- Một `PurchaseItem` thuộc về một `PurchaseInvoice`.
- Khi bán hàng (`SaleItem`), hệ thống sẽ truy xuất lại `PurchaseItem` gốc (thông qua FIFO) để xác định thời hạn bảo hành cho khách.

---

## 🚀 Logic & Thuật Toán

### 1. Cập nhật thông tin (`WarrantyUpdateModal`)

Việc cập nhật thông tin bảo hành được thực hiện tách biệt với luồng nhập kho, giúp kế toán kho có thể cập nhật sau khi hàng đã về.

**Core Logic:**
File: `app/Livewire/Main/Products/WarrantyUpdateModal.php`

```php
public function saveWarranty(): void
{
    // 1. Validate inputs
    $this->validate([
        'warrantyMonths' => ['nullable', 'integer', 'min:0', 'max:1200'],
        'expiryMonths' => ['nullable', 'integer', 'min:0', 'max:1200'],
    ]);

    // 2. Load & Check Owner
    $purchaseItem = PurchaseItem::findOrFail($this->purchaseItemId);
    $this->authorize('update', $purchaseItem); 

    // 3. Update
    $purchaseItem->warranty_months = $this->warrantyMonths;
    $purchaseItem->expiry_months = $this->expiryMonths;
    $purchaseItem->save();

    // 4. Feedback
    $this->dispatch('refresh');
}
```

### 2. Tính toán ngày hết hạn (Calculation)

Hệ thống sử dụng thư viện `Carbon` để tính toán chính xác ngày hết hạn, xử lý các trường hợp năm nhuận hoặc tháng có số ngày khác nhau.

```php
// Input: Purchase Date (Ngày khách mua)
// Logic:
$warrantyExpiryDate = Carbon::parse($purchaseDate)->addMonths($warrantyMonths);
```

**Ví dụ:**
- Khách mua: 31/01/2024
- Bảo hành: 1 tháng
- Hết hạn: 29/02/2024 (Tự động handle năm nhuận)

---

## ✨ Features & UI

### 1. Modal Cập nhật
- **Giao diện:** Tách biệt, popup modal.
- **Preview:** Tự động hiển thị "Ngày hết hạn dự kiến" ngay khi nhập số tháng (Live calculation).
- **Validation:** Chặn nhập số âm hoặc số quá lớn (> 100 năm).

### 2. Hiển thị trên hóa đơn
- Khi in hóa đơn bán hàng, thông tin bảo hành sẽ được hiển thị dòng dưới tên sản phẩm:
  > *Bảo hành: 12 tháng (đến 20/12/2025)*

---

## 🔧 Troubleshooting

### Vấn đề: Không lưu được số tháng bảo hành?
- **Kiểm tra:** User có quyền edit purchase invoice không?
- **Kiểm tra:** Purchase Invoice đã bị khóa sổ (locked) chưa? (Hiện tại hệ thống cho phép sửa bảo hành ngay cả khi đã approved).

### Vấn đề: Ngày hết hạn hiển thị sai?
- **Nguyên nhân:** Format ngày tháng đầu vào (`d/m/Y` vs `Y-m-d`).
- **Fix:** Kiểm tra `Carbon::createFromFormat` trong code.

---

## 📚 Related Files

| File | Type | Mô tả |
|------|------|-------|
| `app/Models/PurchaseItem.php` | Model | Chứa filed `warranty_months` |
| `app/Livewire/Main/Products/WarrantyUpdateModal.php` | Livewire | Logic cập nhật |
| `resources/views/livewire/main/products/warranty-update-modal.blade.php` | View | UI Modal |
