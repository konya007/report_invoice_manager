# 14 - Partner Portal

> Cổng dành cho khách hàng/đối tác

---

## Tổng Quan

### Mục Đích

Partner Portal cho phép khách hàng:
- ✅ Xem hóa đơn của mình
- ✅ Xem hợp đồng
- ✅ Tải xuống file
- ✅ Gửi support tickets
- ❌ KHÔNG được xem dữ liệu công ty
- ❌ KHÔNG được tạo/sửa/xóa

---

## Partner Account

### Tạo Tài Khoản

```php
// app/Services/App/CustomerService.php
public function createQuickAccountPartner(int $customerId): array {
    $customer = $this->findById($customerId);
    
    if (empty($customer->email)) {
        throw new \Exception('Khách hàng chưa có email');
    }
    
    // Check existing
    $existing = User::where('email', $customer->email)->first();
    if ($existing) {
        throw new \Exception('Email đã được sử dụng');
    }
    
    // Generate password
    $password = Str::random(12);
    
    // Create user
    $user = User::create([
        'first_name' => explode(' ', $customer->customer_name)[0],
        'last_name' => implode(' ', array_slice(explode(' ', $customer->customer_name), 1)),
        'email' => $customer->email,
        'password' => Hash::make($password),
        'company_id' => $customer->company_id,
        'customer_id' => $customer->id,  // Link to customer
        'is_active' => true,
    ]);
    
    // Send email with credentials
    $user->notify(new WelcomePartnerAccount($password));
    
    return [
        'user' => $user,
        'password' => $password,  // Show one time only
    ];
}
```

---

## Middleware

### EnsureCustomerAccount

```php
// app/Http/Middleware/EnsureCustomerAccount.php
public function handle(Request $request, Closure $next): Response {
    $user = Auth::user();
    
    if (!$user || !$user->isCustomerAccount()) {
        abort(403, 'Chỉ dành cho tài khoản đối tác');
    }
    
    return $next($request);
}

// User model
public function isCustomerAccount(): bool {
    return !is_null($this->customer_id);
}
```

---

## Routes

### Partner Routes

```php
// routes/partner.php
Route::middleware(['auth', 'ensure.customer'])->prefix('partner')->name('partner.')->group(function() {
    
    // Dashboard
    Route::get('/dashboard', PartnerDashboard::class)->name('dashboard');
    
    // Invoices
    Route::get('/invoices', PartnerInvoices::class)->name('invoices');
    Route::get('/invoices/{id}', PartnerInvoiceDetail::class)->name('invoices.detail');
    
    // Contracts
    Route::get('/contracts', PartnerContracts::class)->name('contracts');
    
    // Support
    Route::get('/support', PartnerSupport::class)->name('support');
    Route::get('/support/create', PartnerSupportCreate::class)->name('support.create');
});
```

---

## Partner Dashboard

### Component

```php
// app/Livewire/Partner/PartnerDashboard.php
class PartnerDashboard extends Component {
    public $stats = [];
    
    public function mount() {
        $user = Auth::user();
        $customerId = $user->customer_id;
        
        // Chỉ xem dữ liệu của mình
        $this->stats = [
            'total_invoices' => SaleInvoice::where('customer_id', $customerId)->count(),
            'total_amount' => SaleInvoice::where('customer_id', $customerId)
                ->where('status', 'approved')
                ->sum('grand_total'),
            'unpaid' => SaleInvoice::where('customer_id', $customerId)
                ->where('payment_status', '!=', 'paid')
                ->count(),
            'contracts' => SaleContract::where('customer_id', $customerId)->count(),
        ];
    }
    
    public function render() {
        return view('livewire.partner.dashboard')
            ->layout('layouts.partner');  // Layout riêng
    }
}
```

---

## Xem Hóa Đơn

### Partner Invoices

```php
// app/Livewire/Partner/PartnerInvoices.php
class PartnerInvoices extends Component {
    public function render() {
        $customerId = Auth::user()->customer_id;
        
        $invoices = SaleInvoice::where('customer_id', $customerId)
            ->with(['items.product'])
            ->orderByDesc('sale_date')
            ->paginate(15);
        
        return view('livewire.partner.invoices', [
            'invoices' => $invoices,
        ])->layout('layouts.partner');
    }
}
```

### Chi Tiết Hóa Đơn

```php
public function mount($id) {
    $customerId = Auth::user()->customer_id;
    
    // Security: Chỉ xem hóa đơn của mình
    $this->invoice = SaleInvoice::where('id', $id)
        ->where('customer_id', $customerId)
        ->firstOrFail();
}
```

---

## Tải File

### Secure File Download

```php
// routes/partner.php
Route::get('/download/{type}/{id}', function($type, $id) {
    $user = Auth::user();
    
    if ($type === 'invoice') {
        $invoice = SaleInvoice::where('id', $id)
            ->where('customer_id', $user->customer_id)
            ->firstOrFail();
        
        if (!$invoice->pdf_file) {
            abort(404);
        }
        
        return response()->download(
            storage_path('app/private/' . $invoice->pdf_file),
            "invoice-{$invoice->sale_number}.pdf"
        );
    }
    
    abort(404);
})->middleware(['auth', 'ensure.customer'])
  ->name('partner.download');
```

---

## Support Tickets

### Tạo Ticket

```php
// app/Livewire/Partner/PartnerSupportCreate.php
public function submit() {
    $validated = $this->validate([
        'subject' => 'required|string|max:255',
        'message' => 'required|string',
        'priority' => 'required|in:low,medium,high',
    ]);
    
    SupportTicket::create([
        'company_id' => Auth::user()->company_id,
        'customer_id' => Auth::user()->customer_id,
        'subject' => $validated['subject'],
        'message' => $validated['message'],
        'priority' => $validated['priority'],
        'status' => 'pending',
        'created_by' => Auth::id(),
    ]);
    
    return redirect()->route('partner.support')
        ->with('message', 'Gửi yêu cầu hỗ trợ thành công!');
}
```

---

## Layout Riêng

### Partner Layout

```blade
{{-- resources/views/layouts/partner.blade.php --}}
<!DOCTYPE html>
<html lang="vi">
<head>
    <meta charset="UTF-8">
    <title>Partner Portal - {{ config('app.name') }}</title>
    @vite(['resources/css/app.css', 'resources/js/app.js'])
    @livewireStyles
</head>
<body class="bg-gray-50">
    {{-- Header --}}
    <header class="bg-white shadow">
        <div class="container mx-auto px-4 py-4 flex justify-between items-center">
            <h1 class="text-xl font-bold">🏢 Portal Đối Tác</h1>
            
            <div class="flex items-center gap-4">
                <span>{{ Auth::user()->fullname }}</span>
                <a href="{{ route('logout') }}" class="text-red-600">Đăng xuất</a>
            </div>
        </div>
    </header>
    
    {{-- Navigation --}}
    <nav class="bg-gray-800 text-white">
        <div class="container mx-auto px-4">
            <ul class="flex gap-6 py-4">
                <li><a href="{{ route('partner.dashboard') }}" class="hover:text-blue-400">Dashboard</a></li>
                <li><a href="{{ route('partner.invoices') }}" class="hover:text-blue-400">Hóa đơn</a></li>
                <li><a href="{{ route('partner.contracts') }}" class="hover:text-blue-400">Hợp đồng</a></li>
                <li><a href="{{ route('partner.support') }}" class="hover:text-blue-400">Hỗ trợ</a></li>
            </ul>
        </div>
    </nav>
    
    {{-- Content --}}
    <main class="container mx-auto px-4 py-8">
        {{ $slot }}
    </main>
    
    @livewireScripts
</body>
</html>
```

---

## Giới Hạn Truy Cập

### Permission Check

```php
// Main app routes - block partners
Route::middleware(['auth', 'ensure.not_customer'])->group(function() {
    // Main app routes
    Route::get('/dashboard', Dashboard::class);
    // ...
});

// Partner routes - only partners
Route::middleware(['auth', 'ensure.customer'])->group(function() {
    // Partner portal routes
});
```

### Database Scoping

```php
// Partner chỉ thấy dữ liệu của mình
class PartnerScope implements Scope {
    public function apply(Builder $builder, Model $model) {
        if (Auth::check() && Auth::user()->isCustomerAccount()) {
            $builder->where('customer_id', Auth::user()->customer_id);
        }
    }
}
```

---

## Quick Reference

### Key Differences

| Feature | Main App | Partner Portal |
|---------|----------|----------------|
| **Users** | Company staff | Khách hàng |
| **Access** | Full system | Own data only |
| **Permissions** | Role-based | Fixed (read-only) |
| **Layout** | Admin layout | Partner layout |
| **Routes** | `/` | `/partner` |
| **Data** | All company data | Customer's data |

### Security Rules

```
✅ Partner CÓ THỂ:
  - Xem hóa đơn của mình
  - Xem hợp đồng của mình
  - Tải file liên quan
  - Tạo support tickets

❌ Partner KHÔNG THỂ:
  - Xem dữ liệu công ty
  - Tạo/sửa/xóa hóa đơn
  - Xem khách hàng khác
  - Truy cập settings
```

---

## Tiếp Theo

✅ Partner portal đã hiểu!

**Tiếp tục:**
- [Export System](15-export-system.md)
- [Statistics](16-statistics-dashboard.md)
- [Components](17-components-and-utilities.md)

---

<p align="center">
  <strong>Partner Portal Thành Thạo! 🏢</strong>
</p>
