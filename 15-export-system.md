# 15 - Hệ Thống Export

> Xuất Excel, PDF cho báo cáo

---

## Excel Export

### Maatwebsite Excel

```bash
composer require maatwebsite/excel
```

### Export Class

```php
// app/Exports/CustomersExport.php
namespace App\Exports;

use App\Models\Customer;
use Maatwebsite\Excel\Concerns\FromQuery;
use Maatwebsite\Excel\Concerns\WithHeadings;
use Maatwebsite\Excel\Concerns\WithMapping;
use Illuminate\Support\Facades\Auth;

class CustomersExport implements FromQuery, WithHeadings, WithMapping {
    
    public function query() {
        return Customer::query()
            ->where('company_id', Auth::user()->company_id)
            ->orderBy('created_at', 'desc');
    }
    
    public function headings(): array {
        return [
            'ID',
            'Tên khách hàng',
            'Email',
            'Số điện thoại',
            'Địa chỉ',
            'Mã số thuế',
            'Ngân hàng',
            'Số TK',
            'Ngày tạo',
        ];
    }
    
    public function map($customer): array {
        return [
            $customer->id,
            $customer->customer_name,
            $customer->email,
            $customer->phone,
            $customer->address,
            $customer->tax_id,
            $customer->bank_name,
            $customer->bank_account,
            $customer->created_at->format('d/m/Y H:i'),
        ];
    }
}
```

### Controller/Livewire

```php
use Maatwebsite\Excel\Facades\Excel;
use App\Exports\CustomersExport;

public function exportExcel() {
    return Excel::download(
        new CustomersExport(),
        'danh-sach-khach-hang-' . now()->format('Y-m-d') . '.xlsx'
    );
}
```

---

## PDF Export

### DomPDF

```bash
composer require barryvossen/laravel-dompdf
```

### PDF Controller

```php
use Barryvdh\DomPDF\Facade\Pdf;

public function exportPDF($id) {
    $invoice = SaleInvoice::with(['customer', 'items.product'])
        ->findOrFail($id);
    
    $pdf = Pdf::loadView('exports.invoice-pdf', [
        'invoice' => $invoice,
    ]);
    
    return $pdf->download("hoa-don-{$invoice->sale_number}.pdf");
}

// Hoặc hiển thị inline
public function viewPDF($id) {
    $invoice = SaleInvoice::with(['customer', 'items.product'])
        ->findOrFail($id);
    
    $pdf = Pdf::loadView('exports.invoice-pdf', ['invoice' => $invoice]);
    
    return $pdf->stream("hoa-don-{$invoice->sale_number}.pdf");
}
```

### PDF Template

```blade
{{-- resources/views/exports/invoice-pdf.blade.php --}}
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>Hóa đơn {{ $invoice->sale_number }}</title>
    <style>
        body { font-family: DejaVu Sans, sans-serif; font-size: 12px; }
        table { width: 100%; border-collapse: collapse; }
        th, td { padding: 8px; border: 1px solid #ddd; text-align: left; }
        th { background-color: #f3f4f6; }
        .header { text-align: center; margin-bottom: 20px; }
        .company-info { margin-bottom: 20px; }
        .totals { margin-top: 20px; text-align: right; }
    </style>
</head>
<body>
    {{-- Header --}}
    <div class="header">
        <h1>HÓA ĐƠN BÁN HÀNG</h1>
        <p>Số: {{ $invoice->sale_number }}</p>
        <p>Ngày: {{ $invoice->sale_date->format('d/m/Y') }}</p>
    </div>
    
    {{-- Company Info --}}
    <div class="company-info">
        <strong>Khách hàng:</strong> {{ $invoice->customer->customer_name }}<br>
        <strong>Địa chỉ:</strong> {{ $invoice->customer->address }}<br>
        <strong>MST:</strong> {{ $invoice->customer->tax_id }}
    </div>
    
    {{-- Items Table --}}
    <table>
        <thead>
            <tr>
                <th>STT</th>
                <th>Sản phẩm</th>
                <th>Số lượng</th>
                <th>Đơn giá</th>
                <th>Thành tiền</th>
            </tr>
        </thead>
        <tbody>
            @foreach($invoice->items as $index => $item)
            <tr>
                <td>{{ $index + 1 }}</td>
                <td>{{ $item->product->name }}</td>
                <td>{{ $item->quantity }} {{ $item->product->unit }}</td>
                <td>{{ number_format($item->unit_price, 0, ',', '.') }}₫</td>
                <td>{{ number_format($item->total_price, 0, ',', '.') }}₫</td>
            </tr>
            @endforeach
        </tbody>
    </table>
    
    {{-- Totals --}}
    <div class="totals">
        <p><strong>Tạm tính:</strong> {{ number_format($invoice->subtotal, 0, ',', '.') }}₫</p>
        <p><strong>VAT:</strong> {{ number_format($invoice->vat_amount, 0, ',', '.') }}₫</p>
        <p style="font-size: 14px;"><strong>TỔNG CỘNG:</strong> {{ number_format($invoice->grand_total, 0, ',', '.') }}₫</p>
    </div>
    
    {{-- Footer --}}
    <div style="margin-top: 50px; text-align: center;">
        <p><em>Cảm ơn quý khách!</em></p>
    </div>
</body>
</html>
```

---

## Export với Filters

### Export Class với Parameters

```php
class InvoicesExport implements FromQuery, WithHeadings {
    private $fromDate;
    private $toDate;
    private $status;
    
    public function __construct($fromDate, $toDate, $status = null) {
        $this->fromDate = $fromDate;
        $this->toDate = $toDate;
        $this->status = $status;
    }
    
    public function query() {
        $query = SaleInvoice::query()
            ->where('company_id', Auth::user()->company_id)
            ->whereBetween('sale_date', [$this->fromDate, $this->toDate]);
        
        if ($this->status) {
            $query->where('status', $this->status);
        }
        
        return $query;
    }
    
    //...
}

// Usage
public function exportWithFilters() {
    return Excel::download(
        new InvoicesExport($this->fromDate, $this->toDate, $this->status),
        'invoices.xlsx'
    );
}
```

---

## Advanced Excel

### Với Styles

```php
use Maatwebsite\Excel\Concerns\WithStyles;
use PhpOffice\PhpSpreadsheet\Worksheet\Worksheet;

class CustomersExport implements FromQuery, WithHeadings, WithStyles {
    
    public function styles(Worksheet $sheet) {
        return [
            // Style cho header row
            1 => [
                'font' => ['bold' => true, 'size' => 12],
                'fill' => [
                    'fillType' => \PhpOffice\PhpSpreadsheet\Style\Fill::FILL_SOLID,
                    'startColor' => ['rgb' => '3B82F6']
                ],
                'font' => ['color' => ['rgb' => 'FFFFFF']],
            ],
        ];
    }
}
```

### Multiple Sheets

```php
use Maatwebsite\Excel\Concerns\WithMultipleSheets;

class ReportExport implements WithMultipleSheets {
    
    public function sheets(): array {
        return [
            new CustomersSheet(),
            new InvoicesSheet(),
            new ProductsSheet(),
        ];
    }
}
```

---

## Bulk Export

### Queue Export

```php
use Maatwebsite\Excel\Concerns\Exportable;
use Illuminate\Contracts\Queue\ShouldQueue;

class LargeInvoicesExport implements FromQuery, ShouldQueue {
    use Exportable;
    
    public function query() {
        // Large dataset
        return SaleInvoice::query()->where('company_id', Auth::user()->company_id);
    }
}

// Usage - export sẽ chạy background
(new LargeInvoicesExport)->queue('invoices.xlsx')->onQueue('exports');

// Notification khi xong
dispatch(new NotifyExportComplete($user, 'invoices.xlsx'));
```

---

## Quick Reference

### Excel Commands

```php
// Download
return Excel::download(new Export, 'file.xlsx');

// Store to disk
Excel::store(new Export, 'file.xlsx', 'exports');

// Queue (large files)
(new Export)->queue('file.xlsx');

// Stream response
return Excel::raw(new Export, \Maatwebsite\Excel\Excel::XLSX);
```

### PDF Commands

```php
// Download
$pdf->download('file.pdf');

// Stream (view in browser)
$pdf->stream('file.pdf');

// Save to storage
$pdf->save(storage_path('app/pdfs/file.pdf'));

// Output string
$content = $pdf->output();
```

---

## Tiếp Theo

✅ Export system đã hiểu!

**Hoàn tất:**
- [Statistics](16-statistics-dashboard.md)
- [Components](17-components-and-utilities.md)
- [DataTables](18-livewire-datatables.md)

---

<p align="center">
  <strong>Export Thành Thạo! 📄</strong>
</p>
