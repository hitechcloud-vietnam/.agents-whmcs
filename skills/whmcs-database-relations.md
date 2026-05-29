# WHMCS Database Relations

## Skill Description
Implement Eloquent-style relationship management for WHMCS modules using Laravel's Capsule ORM to define and query relationships between entities.

## Prerequisites
- WHMCS 7.0+ installation
- PHP 7.4+
- Laravel Capsule ORM
- Basic understanding of relational databases

## Step-by-Step Implementation

### 1. Base Model
```php
<?php
// includes/database/Model.php

namespace WHMCS\Module\YourModule\Database;

use Illuminate\Database\Eloquent\Model as EloquentModel;

abstract class Model extends EloquentModel
{
    protected $connection = 'default';
    protected $table = '';
    protected $primaryKey = 'id';
    protected $keyType = 'int';
    public $timestamps = true;
    const CREATED_AT = 'created_at';
    const UPDATED_AT = 'updated_at';

    public function __construct(array $attributes = [])
    {
        parent::__construct($attributes);
    }

    public function newFromBuilder($attributes = [], $connection = null)
    {
        $model = parent::newFromBuilder($attributes, $connection);
        return $model;
    }

    public function scopeWhereStatus($query, string $status)
    {
        return $query->where('status', $status);
    }

    public function scopeWhereUser($query, int $userId)
    {
        return $query->where('userid', $userId);
    }
}
```

### 2. Relationship Models
```php
<?php
// includes/database/Models/Client.php

namespace WHMCS\Module\YourModule\Database\Models;

use WHMCS\Module\YourModule\Database\Model;

class Client extends Model
{
    protected $table = 'tblclients';

    public function services()
    {
        return $this->hasMany(Service::class, 'userid');
    }

    public function invoices()
    {
        return $this->hasMany(Invoice::class, 'userid');
    }

    public function orders()
    {
        return $this->hasMany(Order::class, 'userid');
    }

    public function tickets()
    {
        return $this->hasMany(Ticket::class, 'userid');
    }

    public function domains()
    {
        return $this->hasMany(Domain::class, 'userid');
    }

    public function credits()
    {
        return $this->hasMany(Credit::class, 'clientid');
    }

    public function activeServices()
    {
        return $this->services()->where('domainstatus', 'Active');
    }

    public function pendingInvoices()
    {
        return $this->invoices()->whereIn('status', ['Unpaid', 'Overdue']);
    }

    public function getTotalSpent(): float
    {
        return $this->invoices()
            ->where('status', 'Paid')
            ->sum('total');
    }
}
```

```php
<?php
// includes/database/Models/Service.php

namespace WHMCS\Module\YourModule\Database\Models;

use WHMCS\Module\YourModule\Database\Model;

class Service extends Model
{
    protected $table = 'tblhosting';

    public function client()
    {
        return $this->belongsTo(Client::class, 'userid');
    }

    public function product()
    {
        return $this->belongsTo(Product::class, 'packageid');
    }

    public function addons()
    {
        return $this->hasMany(ServiceAddon::class, 'hostingid');
    }

    public function customFields()
    {
        return $this->hasMany(CustomFieldValue::class, 'relid')
            ->where('fieldtype', '!=', 'link');
    }

    public function scopeActive($query)
    {
        return $query->where('domainstatus', 'Active');
    }

    public function scopeSuspended($query)
    {
        return $query->where('domainstatus', 'Suspended');
    }

    public function scopeTerminated($query)
    {
        return $query->where('domainstatus', 'Terminated');
    }

    public function getDaysUntilDue(): int
    {
        return max(0, (int) ((strtotime($this->nextduedate) - time()) / 86400));
    }

    public function isOverdue(): bool
    {
        return strtotime($this->nextduedate) < time() && $this->domainstatus === 'Active';
    }
}
```

```php
<?php
// includes/database/Models/Invoice.php

namespace WHMCS\Module\YourModule\Database\Models;

use WHMCS\Module\YourModule\Database\Model;

class Invoice extends Model
{
    protected $table = 'tblinvoices';

    public function client()
    {
        return $this->belongsTo(Client::class, 'userid');
    }

    public function items()
    {
        return $this->hasMany(InvoiceItem::class, 'invoiceid');
    }

    public function transactions()
    {
        return $this->hasMany(Transaction::class, 'invoiceid');
    }

    public function scopeUnpaid($query)
    {
        return $query->where('status', 'Unpaid');
    }

    public function scopeOverdue($query)
    {
        return $query->where('status', 'Overdue')
            ->orWhere(function ($q) {
                $q->where('status', 'Unpaid')
                  ->where('duedate', '<', date('Y-m-d'));
            });
    }

    public function scopePaid($query)
    {
        return $query->where('status', 'Paid');
    }

    public function isPaid(): bool
    {
        return $this->status === 'Paid';
    }

    public function isOverdue(): bool
    {
        return in_array($this->status, ['Unpaid', 'Overdue'])
            && strtotime($this->duedate) < time();
    }

    public function getAmountPaid(): float
    {
        return $this->transactions()->sum('amountin');
    }

    public function getBalance(): float
    {
        return $this->total - $this->getAmountPaid();
    }
}
```

```php
<?php
// includes/database/Models/Product.php

namespace WHMCS\Module\YourModule\Database\Models;

use WHMCS\Module\YourModule\Database\Model;

class Product extends Model
{
    protected $table = 'tblproducts';

    public function services()
    {
        return $this->hasMany(Service::class, 'packageid');
    }

    public function group()
    {
        return $this->belongsTo(ProductGroup::class, 'gid');
    }

    public function scopeFree($query)
    {
        return $query->where('freedomain', 1);
    }

    public function scopeRecurring($query)
    {
        return $query->whereNotIn('billingcycle', ['One Time', 'Free']);
    }

    public function getMonthlyPrice(): float
    {
        return $this->monthly ?? $this->getPriceForCycle('Monthly');
    }

    public function getSetupFee(): float
    {
        return $this->setupfee ?? 0;
    }
}
```

### 3. Relationship Loader
```php
<?php
// includes/database/RelationshipLoader.php

namespace WHMCS\Module\YourModule\Database;

class RelationshipLoader
{
    public static function eagerLoad(array $models, array $relations): void
    {
        foreach ($models as $model) {
            foreach ($relations as $relation) {
                if (method_exists($model, $relation)) {
                    $model->$relation;
                }
            }
        }
    }

    public static function loadServicesWithRelations(array $serviceIds): array
    {
        global $db;

        // Get services
        $placeholders = implode(',', array_fill(0, count($serviceIds), '?'));
        $services = $db->select(
            "SELECT * FROM tblhosting WHERE id IN ({$placeholders})",
            $serviceIds
        );

        $serviceIds = array_column($services, 'id');

        // Eager load related data
        $products = self::loadProducts(array_unique(array_column($services, 'packageid')));
        $clients = self::loadClients(array_unique(array_column($services, 'userid')));

        // Attach relations to services
        foreach ($services as &$service) {
            $service['product'] = $products[$service['packageid']] ?? null;
            $service['client'] = $clients[$service['userid']] ?? null;
        }

        return $services;
    }

    private static function loadProducts(array $productIds): array
    {
        if (empty($productIds)) {
            return [];
        }

        global $db;

        $placeholders = implode(',', array_fill(0, count($productIds), '?'));
        $products = $db->select(
            "SELECT * FROM tblproducts WHERE id IN ({$placeholders})",
            $productIds
        );

        return array_column($products, null, 'id');
    }

    private static function loadClients(array $clientIds): array
    {
        if (empty($clientIds)) {
            return [];
        }

        global $db;

        $placeholders = implode(',', array_fill(0, count($clientIds), '?'));
        $clients = $db->select(
            "SELECT id, firstname, lastname, email, companyname FROM tblclients WHERE id IN ({$placeholders})",
            $clientIds
        );

        return array_column($clients, null, 'id');
    }

    public static function getClientWithAllRelations(int $clientId): ?array
    {
        global $db;

        // Get client
        $clients = $db->select("SELECT * FROM tblclients WHERE id = ?", [$clientId]);

        if (empty($clients)) {
            return null;
        }

        $client = $clients[0];

        // Get related data
        $client['services'] = $db->select(
            "SELECT h.*, p.name as product_name
             FROM tblhosting h
             LEFT JOIN tblproducts p ON h.packageid = p.id
             WHERE h.userid = ?
             ORDER BY h.id DESC",
            [$clientId]
        );

        $client['invoices'] = $db->select(
            "SELECT * FROM tblinvoices WHERE userid = ? ORDER BY id DESC",
            [$clientId]
        );

        $client['orders'] = $db->select(
            "SELECT * FROM tblorders WHERE userid = ? ORDER BY id DESC",
            [$clientId]
        );

        $client['tickets'] = $db->select(
            "SELECT * FROM tbltickets WHERE userid = ? ORDER BY id DESC LIMIT 10",
            [$clientId]
        );

        $client['credits'] = $db->select(
            "SELECT * FROM tblcredit WHERE clientid = ? ORDER BY id DESC",
            [$clientId]
        );

        $client['domains'] = $db->select(
            "SELECT * FROM tbldomains WHERE userid = ? ORDER BY id DESC",
            [$clientId]
        );

        return $client;
    }
}
```

### 4. Usage Examples
```php
<?php
// Example: Using relationships in WHMCS module

namespace WHMCS\Module\YourModule\Services;

use WHMCS\Module\YourModule\Database\RelationshipLoader;
use WHMCS\Module\YourModule\Database\Models\Client;
use WHMCS\Module\YourModule\Database\Models\Invoice;

class ReportingService
{
    public function getClientSummary(int $clientId): array
    {
        $client = new Client();
        $clientData = $client->find($clientId);

        if (!$clientData) {
            return null;
        }

        // Get related data
        $services = $clientData->services()->get();
        $activeServices = $clientData->activeServices()->get();
        $pendingInvoices = $clientData->pendingInvoices()->get();
        $totalSpent = $clientData->getTotalSpent();

        return [
            'client' => $clientData->toArray(),
            'stats' => [
                'total_services' => count($services),
                'active_services' => count($activeServices),
                'pending_invoices' => count($pendingInvoices),
                'total_spent' => $totalSpent
            ]
        ];
    }

    public function getOverdueServices(): array
    {
        global $db;

        $db->query(
            "SELECT h.*, c.firstname, c.lastname, c.email,
                    p.name as product_name
             FROM tblhosting h
             JOIN tblclients c ON h.userid = c.id
             JOIN tblproducts p ON h.packageid = p.id
             WHERE h.domainstatus = 'Active'
             AND h.nextduedate < CURDATE()
             ORDER BY h.nextduedate ASC"
        );

        return $db->fetchAll();
    }

    public function getClientRevenueReport(int $clientId, string $startDate, string $endDate): array
    {
        $invoices = Invoice::where('userid', $clientId)
            ->where('status', 'Paid')
            ->where('date', '>=', $startDate)
            ->where('date', '<=', $endDate)
            ->get();

        $totalRevenue = 0;
        $invoiceCount = count($invoices);

        foreach ($invoices as $invoice) {
            $totalRevenue += $invoice->total;
        }

        return [
            'client_id' => $clientId,
            'period' => [
                'start' => $startDate,
                'end' => $endDate
            ],
            'total_revenue' => $totalRevenue,
            'invoice_count' => $invoiceCount,
            'average_invoice' => $invoiceCount > 0 ? $totalRevenue / $invoiceCount : 0
        ];
    }
}
```

## Common Pitfalls and Solutions

| Pitfall | Solution |
|---------|----------|
| N+1 queries | Use eager loading (with()) |
| Missing relations | Always check if relation returns null |
| Circular references | Use lazy loading with caching |
| Large result sets | Implement pagination |
| Invalid foreign keys | Validate before saving |

## Security Considerations

1. **Validate foreign keys** - Always validate related IDs
2. **Access control** - Check user can access related data
3. **SQL injection** - Use parameterized queries
4. **Data exposure** - Filter sensitive fields from relations

## Testing Checklist

- [ ] Test hasMany relationships
- [ ] Test belongsTo relationships
- [ ] Test hasOne relationships
- [ ] Test eager loading performance
- [ ] Test relationship chaining
- [ ] Test scope usage
- [ ] Test model events
- [ ] Test with empty results

## Reference Links

- [Laravel Eloquent Relationships](https://laravel.com/docs/eloquent-relationships)
- [Eager Loading](https://laravel.com/docs/eloquent-relationships#eager-loading)
- [Querying Relations](https://laravel.com/docs/eloquent-relationships#querying-relations)
