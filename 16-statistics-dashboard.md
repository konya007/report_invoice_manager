# 16 - Statistics & Dashboard

> Báo cáo thống kê và dashboard

---

## Dashboard Components

### Financial Summary Service

```php
// app/Services/Utils/FinancialSummaryService.php
class FinancialSummaryService {
    
    // Tổng hợp theo khoảng thời gian
    public function summarizeRange(int $companyId, $startDate, $endDate): array {
        $sales = SaleInvoice::where('company_id', $companyId)
            ->where('status', 'approved')
            ->whereBetween('sale_date', [$startDate, $endDate])
            ->selectRaw('
                SUM(subtotal) as revenue,
                SUM(vat_amount) as sale_vat,
                COUNT(*) as sale_count
            ')
            ->first();
        
        $purchases = PurchaseInvoice::where('company_id', $companyId)
            ->where('status', 'approved')
            ->whereBetween('purchase_date', [$startDate, $endDate])
            ->selectRaw('
                SUM(subtotal) as cost,
                SUM(vat_amount) as purchase_vat,
                COUNT(*) as purchase_count
            ')
            ->first();
        
        $revenue = $sales->revenue ?? 0;
        $cost = $purchases->cost ?? 0;
        $saleVat = $sales->sale_vat ?? 0;
        $purchaseVat = $purchases->purchase_vat ?? 0;
        
        return [
            'revenue' => $revenue,
            'cost' => $cost,
            'profit' => $revenue - $cost,
            'sale_vat' => $saleVat,
            'purchase_vat' => $purchaseVat,
            'vat_balance' => $saleVat - $purchaseVat,
            'sale_count' => $sales->sale_count ?? 0,
            'purchase_count' => $purchases->purchase_count ?? 0,
        ];
    }
    
    // Tổng hợp theo tháng
    public function summarizeByMonth(int $companyId, int $year): array {
        $result = [];
        
        for ($month = 1; $month <= 12; $month++) {
            $start = Carbon::create($year, $month, 1)->startOfMonth();
            $end = Carbon::create($year, $month, 1)->endOfMonth();
            
            $result[$month] = $this->summarizeRange($companyId, $start, $end);
        }
        
        return $result;
    }
}
```

---

## Dashboard Stats

### Stats Cards

```php
// Thống kê tổng quan
public function getOverviewStats() {
    $companyId = Auth::user()->company_id;
    $startOfYear = Carbon::now()->startOfYear();
    $now = Carbon::now();
    
    $financials = app(FinancialSummaryService::class)
        ->summarizeRange($companyId, $startOfYear, $now);
    
    return [
        'revenue' => $financials['revenue'],
        'cost' => $financials['cost'],
        'profit' => $financials['profit'],
        'vat_balance' => $financials['vat_balance'],
        
        'total_customers' => Customer::where('company_id', $companyId)->count(),
        'total_suppliers' => Supplier::where('company_id', $companyId)->count(),
        'total_products' => Product::where('company_id', $companyId)->count(),
        
        'inventory_value' => Product::where('company_id', $companyId)
            ->sum(DB::raw('quantity * purchase_price')),
    ];
}
```

---

## Charts

### Monthly Revenue Chart

```php
public function getMonthlyRevenueData() {
    $companyId = Auth::user()->company_id;
    $year = now()->year;
    
    $monthly = app(FinancialSummaryService::class)
        ->summarizeByMonth($companyId, $year);
    
    return [
        'labels' => ['T1', 'T2', 'T3', 'T4', 'T5', 'T6', 'T7', 'T8', 'T9', 'T10', 'T11', 'T12'],
        'datasets' => [
            [
                'label' => 'Doanh thu',
                'data' => array_column($monthly, 'revenue'),
                'backgroundColor' => 'rgba(59, 130, 246, 0.5)',
            ],
            [
                'label' => 'Chi phí',
                'data' => array_column($monthly, 'cost'),
                'backgroundColor' => 'rgba(239, 68, 68, 0.5)',
            ],
        ],
    ];
}
```

### Display với Chart.js

```blade
<canvas id="revenueChart"></canvas>

<script>
const ctx = document.getElementById('revenueChart').getContext('2d');
const data = @json($monthlyRevenueData);

new Chart(ctx, {
    type: 'bar',
    data: data,
    options: {
        responsive: true,
        scales: {
            y: {
                beginAtZero: true,
                ticks: {
                    callback: function(value) {
                        return value.toLocaleString('vi-VN') + '₫';
                    }
                }
            }
        }
    }
});
</script>
```

---

## Top Products

```php
public function getTopSellingProducts($limit = 10) {
    return DB::table('sale_items')
        ->join('sale_invoices as inv', 'inv.id', '=', 'sale_items.sale_id')
        ->join('products as p', 'p.id', '=', 'sale_items.product_id')
        ->where('inv.company_id', Auth::user()->company_id)
        ->where('inv.status', 'approved')
        ->whereYear('inv.sale_date', now()->year)
        ->groupBy('p.id', 'p.name', 'p.sku')
        ->select([
            'p.name',
            'p.sku',
            DB::raw('SUM(sale_items.quantity) as total_sold'),
            DB::raw('SUM(sale_items.total_price) as total_revenue'),
        ])
        ->orderByDesc('total_sold')
        ->limit($limit)
        ->get();
}
```

---

## Pending Invoices

```php
public function getPendingInvoices() {
    $companyId = Auth::user()->company_id;
    
    $pendingPurchases = PurchaseInvoice::with('supplier')
        ->where('company_id', $companyId)
        ->where('status', 'pending')
        ->orderByDesc('created_at')
        ->get()
        ->map(fn($inv) => [
            'id' => $inv->id,
            'type' => 'purchase',
            'number' => $inv->purchase_number,
            'partner' => $inv->supplier->supplier_name,
            'date' => $inv->purchase_date->format('d/m/Y'),
            'amount' => $inv->grand_total,
        ]);
    
    $pendingSales = SaleInvoice::with('customer')
        ->where('company_id', $companyId)
        ->where('status', 'pending')
        ->orderByDesc('created_at')
        ->get()
        ->map(fn($inv) => [
            'id' => $inv->id,
            'type' => 'sale',
            'number' => $inv->sale_number,
            'partner' => $inv->customer->customer_name,
            'date' => $inv->sale_date->format('d/m/Y'),
            'amount' => $inv->grand_total,
        ]);
    
    return $pendingPurchases->concat($pendingSales)
        ->sortByDesc('date')
        ->values();
}
```

---

## Low Stock Alerts

```php
public function getLowStockProducts() {
    return Product::where('company_id', Auth::user()->company_id)
        ->whereNotNull('min_stock_level')
        ->whereRaw('quantity <= min_stock_level')
        ->orderBy('quantity')
        ->get()
        ->map(fn($p) => [
            'name' => $p->name,
            'sku' => $p->sku,
            'current' => $p->quantity,
            'minimum' => $p->min_stock_level,
            'status' => $p->quantity == 0 ? 'Hết hàng' : 'Sắp hết',
        ]);
}
```

---

## VAT Report

```php
public function getVATReport($month, $year) {
    $start = Carbon::create($year, $month, 1)->startOfMonth();
    $end = Carbon::create($year, $month, 1)->endOfMonth();
    
    $financials = app(FinancialSummaryService::class)
        ->summarizeRange(Auth::user()->company_id, $start, $end);
    
    return [
        'month' => $month,
        'year' => $year,
        'output_vat' => $financials['sale_vat'],      // VAT đầu ra (thu)
        'input_vat' => $financials['purchase_vat'],   // VAT đầu vào (mua)
        'vat_balance' => $financials['vat_balance'],  // Chênh lệch
        'status' => $financials['vat_balance'] >= 0 
            ? 'Cần nộp ' . number_format($financials['vat_balance'], 0, ',', '.') . '₫'
            : 'Được khấu trừ ' . number_format(abs($financials['vat_balance']), 0, ',', '.') . '₫',
    ];
}
```

---

## Real-time Updates

```php
// Dashboard component
protected $listeners = [
    'echo-private:company.{companyId},invoice.approved' => 'refreshStats',
];

public function refreshStats() {
    $this->stats = $this->getOverviewStats();
    
    $this->dispatch('toast', [
        'message' => 'Thống kê đã được cập nhật',
    ]);
}
```

---

## Quick Reference

### Common Metrics

```php
// Revenue
$revenue = SaleInvoice::where('status', 'approved')->sum('subtotal');

// Cost  
$cost = PurchaseInvoice::where('status', 'approved')->sum('subtotal');

// Profit
$profit = $revenue - $cost;

// Inventory Value
$inventoryValue = Product::sum(DB::raw('quantity * purchase_price'));

// Average Order Value
$avgOrderValue = SaleInvoice::where('status', 'approved')->avg('grand_total');
```

---

## Tiếp Theo

✅ Statistics đã hiểu!

**Còn lại:**
- [Components](17-components-and-utilities.md)
- [DataTables](18-livewire-datatables.md)

---

<p align="center">
  <strong>Statistics Thành Thạo! 📊</strong>
</p>
