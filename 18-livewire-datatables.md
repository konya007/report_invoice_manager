# 18 - Livewire DataTables

> Laravel Livewire Tables - Advanced datatables

---

## Setup

```bash
composer require rappasoft/laravel-livewire-tables
```

---

## Basic Table

### Table Component

```php
// app/Livewire/Main/Customers/CustomerTable.php
namespace App\Livewire\Main\Customers;

use App\Models\Customer;
use App\Services\App\CustomerService;
use Rappasoft\LaravelLivewireTables\DataTableComponent;
use Rappasoft\LaravelLivewireTables\Views\Column;

class CustomerTable extends DataTableComponent {
    protected $model = Customer::class;
    
    public function configure(): void {
        $this->setPrimaryKey('id')
            ->setSearchEnabled()
            ->setPerPageAccepted([10, 25, 50, 100])
            ->setDefaultSort('created_at', 'desc')
            ->setFilterLayoutSlideDown();
    }
    
    public function builder(): Builder {
        $service = app(CustomerService::class);
        return $service->getQueryBuilder();
    }
    
    public function columns(): array {
        return [
            Column::make('ID', 'id')
                ->sortable(),
                
            Column::make('Tên khách hàng', 'customer_name')
                ->sortable()
                ->searchable()
                ->format(fn($value, $row) => 
                    '<a href="'.route('customers.detail', $row->id).'" 
                        class="text-blue-600 hover:underline" wire:navigate>'
                        .htmlspecialchars($value).'</a>'
                )
                ->html(),
                
            Column::make('Email', 'email')
                ->sortable()
                ->searchable(),
                
            Column::make('Điện thoại', 'phone')
                ->searchable(),
                
            Column::make('MST', 'tax_id')
                ->sortable()
                ->searchable(),
                
            Column::make('Ngày tạo', 'created_at')
                ->sortable()
                ->format(fn($value) => $value ? $value->format('d/m/Y') : ''),
                
            Column::make('Thao tác')
                ->label(fn($row) => view('components.table-actions', [
                    'editRoute' => route('customers.edit', $row->id),
                    'deleteAction' => 'deleteCustomer('.$row->id.')',
                ]))
                ->html(),
        ];
    }
}
```

---

## Advanced Features

### Filters

```php
use Rappasoft\LaravelLivewireTables\Views\Filters\SelectFilter;
use Rappasoft\LaravelLivewireTables\Views\Filters\DateFilter;

public function filters(): array {
    return [
        SelectFilter::make('Trạng thái')
            ->setFilterPillTitle('Trạng thái: ')
            ->options([
                '' => 'Tất cả',
                'active' => 'Đang hoạt động',
                'inactive' => 'Ngừng hoạt động',
            ])
            ->filter(function($builder, $value) {
                if ($value === 'active') {
                    $builder->where('is_active', true);
                } elseif ($value === 'inactive') {
                    $builder->where('is_active', false);
                }
            }),
            
        DateFilter::make('Từ ngày')
            ->filter(function($builder, $value) {
                $builder->whereDate('created_at', '>=', $value);
            }),
    ];
}
```

### Bulk Actions

```php
use Rappasoft\LaravelLivewireTables\Views\Filters\BulkAction;

public function configure(): void {
    $this->setPrimaryKey('id')
        ->setBulkActions([
            'exportSelected' => 'Xuất Excel',
            'deleteSelected' => 'Xóa',
        ]);
}

public function bulkActions(): array {
    return [
        'exportSelected' => 'Xuất Excel',
        'deleteSelected' => 'Xóa',
    ];
}

public function exportSelected() {
    $ids = $this->getSelected();
    
    // Export logic
    return Excel::download(
        new CustomersExport($ids),
        'customers.xlsx'
    );
}

public function deleteSelected() {
    $ids = $this->getSelected();
    Customer::whereIn('id', $ids)->delete();
    
    $this->clearSelected();
    $this->dispatch('refresh');
}
```

---

## Customization

### Custom Row Classes

```php
public function setTableRowClass($row): ?string {
    if (!$row->is_active) {
        return 'opacity-50';
    }
    
    return null;
}
```

### Custom Row Attributes

```php
public function setTableRowAttributes($row): array {
    return [
        'wire:key' => 'customer-' . $row->id,
        'class' => $row->is_active ? '' : 'bg-gray-100',
    ];
}
```

---

## Pagination

```php
public function configure(): void {
    $this->setPaginationEnabled()
        ->setPerPageAccepted([10, 25, 50, 100])
        ->setPerPage(25);
}
```

---

## Searching

```php
// Enable global search
public function configure(): void {
    $this->setSearchEnabled()
        ->setSearchDebounce(500); // Wait 500ms after typing
}

// Make columns searchable
Column::make('Tên', 'customer_name')
    ->searchable();
```

---

## Quick Reference

### Configuration Options

```php
$this->setPrimaryKey('id')
    ->setSearchEnabled()
    ->setPerPageAccepted([10, 25, 50])
    ->setDefaultSort('created_at', 'desc')
    ->setFilterLayoutSlideDown()
    ->setBulkActionsEnabled()
    ->setColumnSelectEnabled()
    ->setOfflineIndicatorEnabled(false);
```

### Column Methods

```php
Column::make('Tên', 'customer_name')
    ->sortable()
    ->searchable()
    ->format(fn($value) => ucfirst($value))
    ->html()
    ->collapseOnMobile()
    ->excludeFromColumnSelect();
```

---

## Hoàn Thành!

✅ **19/19 Files Documentation Hoàn Thành!**

Toàn bộ tài liệu kỹ thuật đã được tạo đầy đủ.

---

<p align="center">
  <strong>🎉 Hoàn Thành Toàn Bộ Documentation! 🎉</strong><br>
  <sub>19 files - 4 phases - Comprehensive technical docs</sub>
</p>
