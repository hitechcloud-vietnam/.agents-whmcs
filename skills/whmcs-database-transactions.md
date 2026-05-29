# WHMCS Database Transactions

## Skill Description
Implement proper database transaction handling for WHMCS modules using Laravel's Capsule ORM to ensure data integrity and ACID compliance.

## Prerequisites
- WHMCS 7.0+ installation
- PHP 7.4+
- Basic understanding of database transactions
- Laravel Capsule ORM knowledge

## Step-by-Step Implementation

### 1. Transaction Manager
```php
<?php
// includes/database/TransactionManager.php

namespace WHMCS\Module\YourModule\Database;

class TransactionManager
{
    private bool $inTransaction = false;

    public function begin(): bool
    {
        if ($this->inTransaction) {
            return false;
        }

        global $db;

        try {
            $db->query('START TRANSACTION');
            $this->inTransaction = true;
            return true;
        } catch (\Exception $e) {
            logActivity('Transaction start failed: ' . $e->getMessage());
            return false;
        }
    }

    public function commit(): bool
    {
        if (!$this->inTransaction) {
            return false;
        }

        global $db;

        try {
            $db->query('COMMIT');
            $this->inTransaction = false;
            return true;
        } catch (\Exception $e) {
            $this->rollback();
            logActivity('Transaction commit failed: ' . $e->getMessage());
            return false;
        }
    }

    public function rollback(): bool
    {
        if (!$this->inTransaction) {
            return false;
        }

        global $db;

        try {
            $db->query('ROLLBACK');
            $this->inTransaction = false;
            return true;
        } catch (\Exception $e) {
            logActivity('Transaction rollback failed: ' . $e->getMessage());
            $this->inTransaction = false;
            return false;
        }
    }

    public function isInTransaction(): bool
    {
        return $this->inTransaction;
    }

    public function wrap(callable $callback): mixed
    {
        $this->begin();

        try {
            $result = $callback();
            $this->commit();
            return $result;
        } catch (\Exception $e) {
            $this->rollback();
            throw $e;
        }
    }
}
```

### 2. Safe Transaction Helper
```php
<?php
// includes/database/DatabaseTransactions.php

namespace WHMCS\Module\YourModule\Database;

class DatabaseTransactions
{
    private TransactionManager $transactionManager;

    public function __construct()
    {
        $this->transactionManager = new TransactionManager();
    }

    public function atomic(callable $callback): mixed
    {
        return $this->transactionManager->wrap($callback);
    }

    public function createInvoiceWithItems(array $invoiceData, array $items): array
    {
        return $this->atomic(function () use ($invoiceData, $items) {
            global $db;

            // Create invoice
            $invoiceId = $db->insert('tblinvoices', [
                'userid' => $invoiceData['user_id'],
                'invoicenum' => $invoiceData['invoice_number'] ?? '',
                'date' => date('Y-m-d'),
                'duedate' => $invoiceData['due_date'] ?? date('Y-m-d', strtotime('+14 days')),
                'subtotal' => 0,
                'total' => 0,
                'status' => 'Unpaid',
                'notes' => $invoiceData['notes'] ?? '',
                'created_at' => date('Y-m-d H:i:s')
            ]);

            $subtotal = 0;

            // Add invoice items
            foreach ($items as $item) {
                $db->insert('tblinvoiceitems', [
                    'invoiceid' => $invoiceId,
                    'userid' => $invoiceData['user_id'],
                    'description' => $item['description'],
                    'amount' => $item['amount'],
                    'taxed' => $item['taxed'] ?? 0,
                    'type' => $item['type'] ?? 'Item'
                ]);

                $subtotal += $item['amount'];
            }

            // Update invoice totals
            $tax = $this->calculateTax($subtotal);
            $total = $subtotal + $tax;

            $db->update('tblinvoices', [
                'subtotal' => $subtotal,
                'total' => $total,
                'tax' => $tax
            ], 'id = ?', [$invoiceId]);

            return [
                'invoice_id' => $invoiceId,
                'subtotal' => $subtotal,
                'tax' => $tax,
                'total' => $total
            ];
        });
    }

    public function transferService(
        int $serviceId,
        int $fromUserId,
        int $toUserId,
        bool $prorata = true
    ): bool {
        return $this->atomic(function () use ($serviceId, $fromUserId, $toUserId, $prorata) {
            global $db;

            // Get service details
            $service = $db->select(
                "SELECT * FROM tblhosting WHERE id = ? AND userid = ?",
                [$serviceId, $fromUserId]
            );

            if (empty($service)) {
                throw new \Exception('Service not found');
            }

            $service = $service[0];

            // Create credits for original owner if prorata
            if ($prorata && $service['billingcycle'] !== 'One Time') {
                $remainingDays = $this->calculateRemainingDays($service);
                $dailyRate = $service['amount'] / 30;
                $creditAmount = $remainingDays * $dailyRate;

                $this->addCredit($fromUserId, $creditAmount, "Service transfer credit - {$service['domain']}");
            }

            // Transfer service
            $db->update('tblhosting', [
                'userid' => $toUserId,
                'updated_at' => date('Y-m-d H:i:s')
            ], 'id = ?', [$serviceId]);

            // Transfer addons
            $db->update('tblhostingaddons', [
                'userid' => $toUserId
            ], 'hostingid = ?', [$serviceId]);

            // Transfer domain if linked
            if (!empty($service['domain'])) {
                $db->update('tbldomains', [
                    'userid' => $toUserId
                ], 'domain = ?', [$service['domain']]);
            }

            return true;
        });
    }

    public function bulkUpdateStatus(array $serviceIds, string $newStatus): int
    {
        return $this->atomic(function () use ($serviceIds, $newStatus) {
            global $db;

            $updated = 0;

            foreach ($serviceIds as $id) {
                $db->update('tblhosting', [
                    'domainstatus' => $newStatus,
                    'updated_at' => date('Y-m-d H:i:s')
                ], 'id = ?', [$id]);

                $updated += $db->affectedRows();

                // Log the status change
                $db->insert('tblactivitylog', [
                    'date' => date('Y-m-d H:i:s'),
                    'description' => "Service {$id} status changed to {$newStatus}",
                    'userid' => 0,
                    'ipaddr' => $_SERVER['REMOTE_ADDR'] ?? ''
                ]);
            }

            return $updated;
        });
    }

    private function calculateTax(float $amount, float $rate = 0): float
    {
        return round($amount * ($rate / 100), 2);
    }

    private function calculateRemainingDays(array $service): int
    {
        $nextDue = strtotime($service['nextduedate']);
        $today = time();

        return max(0, (int) (($nextDue - $today) / 86400));
    }

    private function addCredit(int $userId, float $amount, string $description): void
    {
        global $db;

        $db->insert('tblcredit', [
            'clientid' => $userId,
            'amount' => $amount,
            'description' => $description,
            'date' => date('Y-m-d H:i:s')
        ]);

        // Update client credit total
        $db->query(
            "UPDATE tblclients SET credit = credit + ? WHERE id = ?",
            [$amount, $userId]
        );
    }
}
```

### 3. Transaction Logging Trait
```php
<?php
// includes/database/LogsTransactions.php

namespace WHMCS\Module\YourModule\Traits;

trait LogsTransactions
{
    private array $transactionLog = [];

    protected function logTransaction(string $action, array $data): void
    {
        $this->transactionLog[] = [
            'action' => $action,
            'data' => $data,
            'timestamp' => date('Y-m-d H:i:s'),
            'backtrace' => debug_backtrace(DEBUG_BACKTRACE_IGNORE_ARGS, 5)
        ];
    }

    protected function getTransactionLog(): array
    {
        return $this->transactionLog;
    }

    protected function clearTransactionLog(): void
    {
        $this->transactionLog = [];
    }

    protected function logToDatabase(string $action, array $details, ?int $userId = null): void
    {
        global $db;

        $db->insert('mod_yourmodule_transaction_log', [
            'action' => $action,
            'details' => json_encode($details),
            'user_id' => $userId,
            'ip_address' => $_SERVER['REMOTE_ADDR'] ?? '',
            'created_at' => date('Y-m-d H:i:s')
        ]);
    }
}
```

### 4. Database Locking Helper
```php
<?php
// includes/database/DatabaseLock.php

namespace WHMCS\Module\YourModule\Database;

class DatabaseLock
{
    private static ?\PDO $pdo = null;

    private static function getPdo(): \PDO
    {
        if (self::$pdo === null) {
            $config = include __DIR__ . '/../../config.php';
            self::$pdo = new \PDO(
                $config['db_host'],
                $config['db_username'],
                $config['db_password'],
                [\PDO::ATTR_ERRMODE => \PDO::ERRMODE_EXCEPTION]
            );
        }

        return self::$pdo;
    }

    public static function acquire(string $lockName, int $timeout = 10): bool
    {
        $pdo = self::getPdo();

        try {
            $stmt = $pdo->prepare('SELECT GET_LOCK(?, ?)');
            $stmt->execute([$lockName, $timeout]);
            $result = $stmt->fetchColumn();

            return $result === 1;
        } catch (\Exception $e) {
            logActivity('Failed to acquire lock: ' . $e->getMessage());
            return false;
        }
    }

    public static function release(string $lockName): bool
    {
        $pdo = self::getPdo();

        try {
            $stmt = $pdo->prepare('SELECT RELEASE_LOCK(?)');
            $stmt->execute([$lockName]);
            $result = $stmt->fetchColumn();

            return $result === 1;
        } catch (\Exception $e) {
            logActivity('Failed to release lock: ' . $e->getMessage());
            return false;
        }
    }

    public static function withLock(string $lockName, callable $callback, int $timeout = 10): mixed
    {
        if (!self::acquire($lockName, $timeout)) {
            throw new \Exception("Could not acquire lock: {$lockName}");
        }

        try {
            return $callback();
        } finally {
            self::release($lockName);
        }
    }
}
```

## Common Pitfalls and Solutions

| Pitfall | Solution |
|---------|----------|
| Nested transactions | Use savepoints for nested transactions |
| Transaction left open | Use wrap() method with automatic rollback |
| Deadlocks | Implement retry logic with exponential backoff |
| Lock timeout | Set appropriate lock timeout values |
| Uncommitted reads | Ensure proper isolation level |

## Security Considerations

1. **Validate transaction inputs** - Never trust user data in transactions
2. **Log transaction failures** - Track failed transactions for debugging
3. **Use proper permissions** - Limit database user permissions
4. **Secure connection** - Use SSL for database connections
5. **Prevent SQL injection** - Always use prepared statements

## Testing Checklist

- [ ] Test successful transaction commit
- [ ] Test transaction rollback on exception
- [ ] Test nested transaction with savepoints
- [ ] Test concurrent transaction handling
- [ ] Test lock acquisition and release
- [ ] Test transaction log recording
- [ ] Test atomic operations
- [ ] Test partial failure scenarios

## Reference Links

- [MySQL Transaction Documentation](https://dev.mysql.com/doc/refman/8.0/en/commit.html)
- [Laravel Database Transactions](https://laravel.com/docs/database#database-transactions)
- [Database ACID Properties](https://en.wikipedia.org/wiki/ACID)
