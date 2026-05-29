# WHMCS Eloquent Models

## Overview

Eloquent ORM provides an object-oriented interface for database operations in WHMCS modules.

## Defining Models

```php
<?php
namespaceWHMCS\Module\YourModule\Model;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;
use Illuminate\Database\Eloquent\Relations\HasMany;

class ServiceRecord extends Model
{
    protected $table = 'tblhosting';
    protected $primaryKey = 'id';
    public $timestamps = true;
    protected $connection = 'mysql';
    
    protected $fillable = [
        'userid',
        'packageid',
        'server',
        'regdate',
        'domain',
        'username',
        'password',
        'status',
    ];
    
    protected $hidden = [
        'password',
    ];
    
    protected $casts = [
        'regdate' => 'date',
        'nextduedate' => 'date',
        'created_at' => 'datetime',
        'updated_at' => 'datetime',
    ];
}
```

## Relationships

### BelongsTo

```php
<?php
class ServiceRecord extends Model
{
    public function client(): BelongsTo
    {
        return $this->belongsTo(Client::class, 'userid', 'id');
    }
    
    public function product(): BelongsTo
    {
        return $this->belongsTo(Product::class, 'packageid', 'id');
    }
    
    public function server(): BelongsTo
    {
        return $this->belongsTo(Server::class, 'server', 'id');
    }
}

// Usage
$service = ServiceRecord::with(['client', 'product'])->find(1);
echo $service->client->fullName;
echo $service->product->name;
```

### HasMany

```php
<?php
class Client extends Model
{
    protected $table = 'tblclients';
    protected $primaryKey = 'id';
    
    public function invoices(): HasMany
    {
        return $this->hasMany(Invoice::class, 'userid', 'id');
    }
    
    public function services(): HasMany
    {
        return $this->hasMany(ServiceRecord::class, 'userid', 'id');
    }
    
    public function domains(): HasMany
    {
        return $this->hasMany(Domain::class, 'userid', 'id');
    }
    
    public function activeServices(): HasMany
    {
        return $this->hasMany(ServiceRecord::class, 'userid', 'id')
            ->where('domainstatus', 'Active');
    }
}

// Usage
$client = Client::find(1);
$invoices = $client->invoices;
$activeServices = $client->activeServices;
```

### BelongsToMany (Pivot)

```php
<?php
class Product extends Model
{
    protected $table = 'tblproducts';
    
    public function addons(): BelongsToMany
    {
        return $this->belongsToMany(
            Addon::class,
            'tblproduct_addons',
            'product_id',
            'addon_id'
        )->withPivot(['billingcycle', 'tax']);
    }
}

// Usage
$product = Product::find(1);
$addons = $product->addons;
```

## Query Scopes

### Local Scopes

```php
<?php
class Invoice extends Model
{
    protected $table = 'tblinvoices';
    
    // Scope methods
    public function scopePaid($query)
    {
        return $query->where('status', 'Paid');
    }
    
    public function scopeUnpaid($query)
    {
        return $query->where('status', 'Unpaid');
    }
    
    public function scopeOverdue($query)
    {
        return $query->where('status', 'Unpaid')
            ->where('duedate', '<', date('Y-m-d'));
    }
    
    public function scopeForDateRange($query, string $start, string $end)
    {
        return $query->whereBetween('datecreated', [$start, $end]);
    }
    
    public function scopeByUser($query, int $userId)
    {
        return $query->where('userid', $userId);
    }
}

// Usage
$paidInvoices = Invoice::paid()->get();
$overdueInvoices = Invoice::overdue()->forDateRange('2024-01-01', '2024-12-31')->get();
```

## Accessors and Mutators

### Accessors

```php
<?php
class Client extends Model
{
    // Get full name attribute
    public function getFullNameAttribute(): string
    {
        return trim($this->firstname . ' ' . $this->lastname);
    }
    
    // Format date
    public function getCreatedAtFormattedAttribute(): string
    {
        return $this->created_at ? $this->created_at->format('M d, Y') : '';
    }
    
    // Boolean accessor
    public function getIsActiveAttribute(): bool
    {
        return $this->status === 'Active';
    }
}

// Usage
$client = Client::find(1);
echo $client->fullName; // "John Doe"
echo $client->is_active ? 'Yes' : 'No';
```

### Mutators

```php
<?php
class ServiceRecord extends Model
{
    // Hash password on set
    public function setPasswordAttribute(string $value): void
    {
        $this->attributes['password'] = encrypt($value);
    }
    
    // Get decrypted password
    public function getPasswordAttribute(string $value): string
    {
        return decrypt($value);
    }
    
    // Format amount on set
    public function setAmountAttribute(float $value): void
    {
        $this->attributes['amount'] = round($value, 2);
    }
}
```

## Collections

```php
<?php
// Get collection
$clients = Client::all();

// Filter collection
$activeClients = $clients->filter(function ($client) {
    return $client->status === 'Active';
});

// Map collection
$emails = $clients->map(function ($client) {
    return $client->email;
});

// Group collection
$byCountry = $clients->groupBy('country');

// Chunk collection
Client::chunk(100, function ($clients) {
    foreach ($clients as $client) {
        processClient($client);
    }
});

// Pluck specific column
$names = Client::pluck('firstname', 'id'); // [1 => 'John', 2 => 'Jane']
```

## Events

```php
<?php
class Invoice extends Model
{
    protected $dispatchesEvents = [
        'created' => InvoiceCreated::class,
        'updated' => InvoiceUpdated::class,
        'deleted' => InvoiceDeleted::class,
    ];
}

// Or use boot method
class Invoice extends Model
{
    protected static function boot()
    {
        parent::boot();
        
        static::created(function ($invoice) {
            logActivity("Invoice {$invoice->id} created");
        });
        
        static::updating(function ($invoice) {
            // Store old values
            $invoice->oldStatus = $invoice->getOriginal('status');
        });
        
        static::updated(function ($invoice) {
            if ($invoice->oldStatus !== $invoice->status) {
                logActivity("Invoice status changed from {$invoice->oldStatus} to {$invoice->status}");
            }
        });
    }
}
```

## Full Model Example

```php
<?php
namespace WHMCS\Module\YourModule\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;
use Illuminate\Database\Eloquent\Relations\HasMany;

class Ticket extends Model
{
    protected $table = 'tbtickets';
    protected $primaryKey = 'id';
    public $timestamps = false;
    
    protected $fillable = [
        'userid',
        'did',
        'subject',
        'priority',
        'status',
        'adminunread',
        'clientunread',
    ];
    
    protected $casts = [
        'created' => 'datetime',
        'lastreply' => 'datetime',
    ];
    
    // Relationships
    public function client(): BelongsTo
    {
        return $this->belongsTo(Client::class, 'userid', 'id');
    }
    
    public function department(): BelongsTo
    {
        return $this->belongsTo(TicketDepartment::class, 'did', 'id');
    }
    
    public function replies(): HasMany
    {
        return $this->hasMany(TicketReply::class, 'tid', 'id');
    }
    
    // Scopes
    public function scopeOpen($query)
    {
        return $query->whereNotIn('status', ['Closed']);
    }
    
    public function scopeByPriority($query, string $priority)
    {
        return $query->where('priority', $priority);
    }
    
    public function scopeUnread($query)
    {
        return $query->where('clientunread', 1);
    }
    
    // Accessors
    public function getIsOpenAttribute(): bool
    {
        return !in_array($this->status, ['Closed']);
    }
    
    public function getTicketNumberAttribute(): string
    {
        return 'TKT-' . str_pad($this->id, 8, '0', STR_PAD_LEFT);
    }
}
```

## Related Documentation

- [WHMCS Capsule Queries](/docs/whmcs-capsule-queries.md)
- [WHMCS Capsule Schema](/docs/whmcs-capsule-schema.md)