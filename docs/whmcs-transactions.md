# WHMCS Transaction Management

## Overview

Transaction management ensures data consistency when performing multiple database operations.

## Basic Transactions

### Auto-Commit Transaction

```php
<?php
use WHMCS\Database\Capsule;

// Simple transaction - auto-commits on success
$result = Capsule::transaction(function () {
    // Create client
    $clientId = Capsule::table('tblclients')->insertGetId([
        'firstname' => 'John',
        'lastname' => 'Doe',
        'email' => 'john@example.com',
        'created_at' => date('Y-m-d H:i:s'),
    ]);
    
    // Create service
    Capsule::table('tblhosting')->insert([
        'userid' => $clientId,
        'packageid' => 1,
        'domain' => 'example.com',
        'regdate' => date('Y-m-d'),
        'domainstatus' => 'Pending',
    ]);
    
    // Create invoice
    $invoiceId = Capsule::table('tblinvoices')->insertGetId([
        'userid' => $clientId,
        'date' => date('Y-m-d'),
        'duedate' => date('Y-m-d', strtotime('+7 days')),
        'total' => 10.00,
        'status' => 'Unpaid',
    ]);
    
    // Add invoice item
    Capsule::table('tblinvoiceitems')->insert([
        'invoice_id' => $invoiceId,
        'userid' => $clientId,
        'description' => 'Hosting Service',
        'amount' => 10.00,
    ]);
    
    return $clientId;
});
```

### Manual Transaction Control

```php
<?php
$connection = Capsule::connection();
$pdo = $connection->getPdo();

try {
    $pdo->beginTransaction();
    
    // Operations
    $orderId = createOrder();
    $invoiceId = createInvoice($orderId);
    processPayment($invoiceId);
    
    $pdo->commit();
    
} catch (Exception $e) {
    $pdo->rollBack();
    logActivity("Transaction failed: " . $e->getMessage());
    throw $e;
}
```

## Savepoints

```php
<?php
function complexTransaction(): void
{
    $pdo = Capsule::connection()->getPdo();
    $pdo->beginTransaction();
    
    try {
        // Main operations
        $clientId = createClient();
        
        // Savepoint for service creation
        $pdo->exec("SAVEPOINT before_service");
        
        try {
            foreach ($services as $service) {
                createService($clientId, $service);
            }
        } catch (Exception $e) {
            // Rollback to savepoint
            $pdo->exec("ROLLBACK TO SAVEPOINT before_service");
            // Continue with client but skip failed services
            logActivity("Some services failed, continuing with partial order");
        }
        
        // Release savepoint
        $pdo->exec("RELEASE SAVEPOINT before_service");
        
        // Finalize
        $invoiceId = createInvoice($clientId);
        
        $pdo->commit();
        
    } catch (Exception $e) {
        $pdo->rollBack();
        throw $e;
    }
}
```

## Nested Transactions

```php
<?php
class TransactionManager
{
    private int $depth = 0;
    private bool $inTransaction = false;
    
    public function begin(): void
    {
        if ($this->depth === 0) {
            Capsule::connection()->getPdo()->beginTransaction();
            $this->inTransaction = true;
        }
        $this->depth++;
    }
    
    public function commit(): void
    {
        $this->depth--;
        
        if ($this->depth === 0 && $this->inTransaction) {
            Capsule::connection()->getPdo()->commit();
            $this->inTransaction = false;
        }
    }
    
    public function rollback(): void
    {
        if ($this->depth > 0 && $this->inTransaction) {
            Capsule::connection()->getPdo()->rollBack();
            $this->inTransaction = false;
        }
        $this->depth = 0;
    }
}

// Usage
$tx = new TransactionManager();

$tx->begin();
try {
    createClient();
    
    $tx->begin();
    try {
        createService();
    } catch (Exception $e) {
        // Service failed but continue
    }
    $tx->commit();
    
    createInvoice();
    
    $tx->commit();
} catch (Exception $e) {
    $tx->rollback();
    throw $e;
}
```

## Transaction with Locking

```php
<?php
function reserveInventory(array $items): bool
{
    return Capsule::transaction(function () use ($items) {
        foreach ($items as $item) {
            // Lock the row for update
            $product = Capsule::table('tblproducts')
                ->where('id', $item['product_id'])
                ->lockForUpdate()
                ->first();
            
            if ($product->stock < $item['quantity']) {
                throw new Exception("Insufficient stock for product {$item['product_id']}");
            }
            
            // Reduce stock
            Capsule::table('tblproducts')
                ->where('id', $item['product_id'])
                ->decrement('stock', $item['quantity']);
        }
        
        return true;
    });
}
```

## Optimistic Locking

```php
<?php
class OptimisticLock
{
    public static function update(array $data, int $expectedVersion): bool
    {
        $updated = Capsule::table('mod_records')
            ->where('id', $data['id'])
            ->where('version', $expectedVersion)
            ->update([
                'data' => $data['data'],
                'version' => $expectedVersion + 1,
                'updated_at' => date('Y-m-d H:i:s'),
            ]);
        
        if ($updated === 0) {
            throw new OptimisticLockException(
                'Record was modified by another process'
            );
        }
        
        return true;
    }
    
    public static function updateWithRetry(callable $operation, int $maxRetries = 3): mixed
    {
        $attempts = 0;
        
        while ($attempts < $maxRetries) {
            try {
                return $operation();
            } catch (OptimisticLockException $e) {
                $attempts++;
                
                if ($attempts >= $maxRetries) {
                    throw $e;
                }
                
                // Refresh data and retry
                usleep(100000 * $attempts); // Exponential backoff
            }
        }
        
        throw new Exception('Max retries exceeded');
    }
}
```

## Best Practices

1. **Keep transactions short** - Minimize lock time
2. **Avoid nested transactions** - Use savepoints instead
3. **Handle failures** - Always rollback on error
4. **Check isolation levels** - Use appropriate isolation
5. **Log transactions** - Track for debugging

## Related Documentation

- [WHMCS Capsule Queries](/docs/whmcs-capsule-queries.md)
- [WHMCS Database Optimization](/docs/whmcs-database-optimization.md)