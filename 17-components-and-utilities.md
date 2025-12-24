# 17 - Components & Utilities

> Các components tái sử dụng và tiện ích

---

## Blade Components

### Alert Component

```blade
{{-- resources/views/components/ui/alert.blade.php --}}
@props(['type' => 'info', 'close' => false, 'timeout' => 0])

<div 
    x-data="{ show: true }"
    x-show="show"
    x-transition
    @if($timeout > 0)
        x-init="setTimeout(() => show = false, {{ $timeout }})"
    @endif
    {{ $attributes->merge(['class' => 'p-4 rounded-lg mb-4']) }}
    :class="{
        'bg-blue-50 text-blue-800': '{{ $type }}' === 'info',
        'bg-green-50 text-green-800': '{{ $type }}' === 'success',
        'bg-yellow-50 text-yellow-800': '{{ $type }}' === 'warning',
        'bg-red-50 text-red-800': '{{ $type }}' === 'error',
    }">
    
    <div class="flex items-start">
        <div class="flex-1">
            {{ $slot }}
        </div>
        
        @if($close)
            <button @click="show = false" class="ml-4">
                <i class="bi bi-x-lg"></i>
            </button>
        @endif
    </div>
</div>

{{-- Sử dụng --}}
<x-ui.alert type="success" :close="true" :timeout="5000">
    Lưu thành công!
</x-ui.alert>
```

### Modal Component

```blade
{{-- resources/views/components/ui/modal.blade.php --}}
@props(['name', 'title' => '', 'maxWidth' => '2xl'])

<div 
    x-data="{ show: false }"
    x-on:open-modal.window="if ($event.detail === '{{ $name }}') show = true"
    x-on:close-modal.window="if ($event.detail === '{{ $name }}') show = false"
    x-show="show"
    class="fixed inset-0 z-50 overflow-y-auto"
    style="display: none;">
    
    {{-- Backdrop --}}
    <div 
        x-show="show"
        x-on:click="show = false"
        x-transition:enter="ease-out duration-300"
        x-transition:enter-start="opacity-0"
        x-transition:enter-end="opacity-100"
        class="fixed inset-0 bg-black bg-opacity-50">
    </div>
    
    {{-- Modal --}}
    <div class="flex items-center justify-center min-h-screen p-4">
        <div 
            x-show="show"
            x-transition
            @click.outside="show = false"
            class="bg-white rounded-lg shadow-xl max-w-{{ $maxWidth }} w-full">
            
            {{-- Header --}}
            @if($title)
            <div class="px-6 py-4 border-b">
                <h3 class="text-lg font-semibold">{{ $title }}</h3>
            </div>
            @endif
            
            {{-- Body --}}
            <div class="px-6 py-4">
                {{ $slot }}
            </div>
        </div>
    </div>
</div>

{{-- Sử dụng --}}
<button x-data @click="$dispatch('open-modal', 'confirm-delete')">Xóa</button>

<x-ui.modal name="confirm-delete" title="Xác nhận xóa">
    <p>Bạn có chắc muốn xóa?</p>
    <div class="mt-4 flex gap-2">
        <button x-data @click="$dispatch('close-modal', 'confirm-delete')">Hủy</button>
        <button wire:click="delete">Xóa</button>
    </div>
</x-ui.modal>
```

---

## Helper Functions

### Number Formatting

```php
// app/Support/helpers.php

if (!function_exists('format_currency')) {
    function format_currency($amount, $currency = 'VND'): string {
        if ($currency === 'VND') {
            return number_format($amount, 0, ',', '.') . '₫';
        }
        
        return $currency . ' ' . number_format($amount, 2, '.', ',');
    }
}

if (!function_exists('format_date')) {
    function format_date($date, $format = 'd/m/Y'): string {
        if (!$date) return '';
        
        return Carbon::parse($date)->format($format);
    }
}

// Sử dụng
{{ format_currency(1000000) }}  // 1.000.000₫
{{ format_date($invoice->sale_date) }}  // 24/12/2025
```

---

## Traits

### ValidatesCompanyScopedUnique

```php
// app/Livewire/Traits/ValidatesCompanyScopedUnique.php
trait ValidatesCompanyScopedUnique {
    
    protected function getCompanyScopedUniqueRule(
        string $table, 
        string $column, 
        $ignore = null
    ): string {
        $companyId = Auth::user()->company_id;
        
        $rule = "unique:{$table},{$column},NULL,id,company_id,{$companyId}";
        
        if ($ignore) {
            $rule .= ",{$ignore}";
        }
        
        return $rule;
    }
}

// Sử dụng trong Livewire component
use ValidatesCompanyScopedUnique;

protected function rules() {
    return [
        'taxId' => [
            'required',
            $this->getCompanyScopedUniqueRule('customers', 'tax_id', $this->customerId),
        ],
    ];
}
```

---

## Utilities

### Currency Helper

```php
// app/Services/Utils/CurrencyHelper.php
class CurrencyHelper {
    
    public function convertToDisplayCurrency(
        float $amount,
        string $fromCurrency,
        ?TransferCurrency $transferCurrency,
        int $companyId
    ): float {
        // Nếu đã là VND, return luôn
        if ($fromCurrency === 'VND') {
            return $amount;
        }
        
        // Lấy tỷ giá
        if ($transferCurrency) {
            $rate = $transferCurrency->rate;
        } else {
            // Fallback: tìm tỷ giá gần nhất
            $rate = $this->getLatestRate($fromCurrency, 'VND', $companyId);
        }
        
        return $amount * $rate;
    }
    
    private function getLatestRate($from, $to, $companyId) {
        $transfer = TransferCurrency::where('company_id', $companyId)
            ->where('from_currency', $from)
            ->where('to_currency', $to)
            ->latest('date')
            ->first();
        
        return $transfer?->rate ?? 1;
    }
}
```

---

## Quick Reference

### Components Location

```
resources/views/components/
├── ui/
│   ├── alert.blade.php
│   ├── modal.blade.php
│   ├── button.blade.php
│   ├── input.blade.php
│   └── select.blade.php
└── layouts/
    ├── main.blade.php
    └── partner.blade.php
```

### Usage

```blade
{{-- Alert --}}
<x-ui.alert type="success">Thành công!</x-ui.alert>

{{-- Modal --}}
<x-ui.modal name="my-modal" title="Title">
    Content
</x-ui.modal>

{{-- Button --}}
<x-ui.button color="primary" wire:click="save">
    Lưu
</x-ui.button>
```

---

## Tiếp Theo

✅ Components & Utilities đã hiểu!

**File cuối:**
- [DataTables Guide](18-livewire-datatables.md)

---

<p align="center">
  <strong>Components Thành Thạo! 🧩</strong>
</p>
