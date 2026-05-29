# WHMCS Transaction Management Workflow

## Overview
This workflow implements proper transaction management for WHMCS to ensure data integrity.

## Prerequisites
- WHMCS with database access
- Understanding of ACID properties
- PHP 7.0+ for transactions

## Step-by-Step Process

### Step 1: Understand Transaction Patterns
```
TRANSACTION PATTERNS:
1. Simple Transaction - Single operation
2. Nested Transaction - Multiple related operations
3. Saga Pattern - Distributed transactions
4. Outbox Pattern - Reliable messaging
```

### Step 2: Create Transaction Helper
```php
<?php
// /includes/transactions/TransactionHelper.php

namespace WHMCS\Transactions;

class TransactionHelper
{
    /**
     * Execute callback within transaction
     */
    public static function run(callable $callback, array $options = []): mixed
    {
        $options = array_merge([
            'isolation_level' => null,
            'retries' => 3,
            'retry_on_deadlock' => true
        ], $options);

        $attempt = 0;

        while ($attempt < $options['retries']) {
            $attempt++;

            try {
                return Capsule::connection()->transaction(function() use ($callback) {
                    return $callback();
                });
            } catch (QueryException $e) {
                if ($options['retry_on_deadlock'] && self::isDeadlockError($e)) {
                    logActivity("Transaction deadlock, retry attempt {$attempt}");

                    // Exponential backoff
                    usleep(pow(2, $attempt) * 100000);
                    continue;
                }

                throw $e;
            }
        }

        throw new Exception("Transaction failed after {$options['retries']} retries");
    }

    private static function isDeadlockError(QueryException $e): bool
    {
        $message = strtolower($e->getMessage());
        return strpos($message, 'deadlock') !== false
            || strpos($message, 'lock wait timeout') !== false;
    }

    /**
     * Execute with row-level locking
     */
    public static function runWithLock(string $table, $id, callable $callback, string $lockType = 'FOR UPDATE')
    {
        return Capsule::connection()->transaction(function() use ($table, $id, $callback, $lockType) {
            // Acquire lock
            Capsule::select("SELECT * FROM {$table} WHERE id = ? {$lockType}", [$id]);

            return $callback();
        });
    }
}
```

### Step 3: Create Service Provisioning Transaction
```php
<?php
// /includes/transactions/ServiceProvisioningTransaction.php

class ServiceProvisioningTransaction
{
    public static function execute(array $orderData): array
    {
        return TransactionHelper::run(function() use ($orderData) {
            // 1. Create client if not exists
            $clientId = self::ensureClient($orderData);

            // 2. Create service record
            $serviceId = self::createService($clientId, $orderData);

            // 3. Create invoice
            $invoiceId = self::createInvoice($clientId, $serviceId, $orderData);

            // 4. Record transaction
            self::recordTransaction($serviceId, $invoiceId);

            // 5. Trigger provisioning
            $provisionResult = self::provisionService($serviceId);

            if (!$provisionResult['success']) {
                throw new Exception("Provisioning failed: " . $provisionResult['error']);
            }

            // 6. Update service status
            Capsule::table('tblhosting')
                ->where('id', $serviceId)
                ->update(['domainstatus' => 'Active']);

            return [
                'success' => true,
                'client_id' => $clientId,
                'service_id' => $serviceId,
                'invoice_id' => $invoiceId
            ];
        });
    }

    private static function ensureClient(array $data): int
    {
        // Check if client exists
        $existing = Capsule::table('tblclients')
            ->where('email', $data['email'])
            ->first();

        if ($existing) {
            return $existing->id;
        }

        // Create new client
        $result = localApi('AddClient', [
            'firstname' => $data['firstname'],
            'lastname' => $data['lastname'],
            'email' => $data['email'],
            'password2' => $data['password'] ?? generateRandomPassword()
        ]);

        return $result['clientid'];
    }

    private static function createService(int $clientId, array $data): int
    {
        $serviceId = Capsule::table('tblhosting')->insertGetId([
            'userid' => $clientId,
            'packageid' => $data['product_id'],
            'server' => $data['server_id'] ?? 0,
            'regdate' => date('Y-m-d'),
            'domainstatus' => 'Pending',
            'billingcycle' => $data['billing_cycle'],
            'nextduedate' => calculateNextDueDate($data['billing_cycle']),
            'amount' => $data['price']
        ]);

        return $serviceId;
    }

    private static function createInvoice(int $clientId, int $serviceId, array $data): int
    {
        $result = localApi('CreateInvoice', [
            'userid' => $clientId,
            'sendinvoice' => false
        ]);

        $invoiceId = $result['invoiceid'];

        // Add service item
        localApi('AddInvoiceItem', [
            'invoiceid' => $invoiceId,
            'type' => 'Hosting',
            'relid' => $serviceId,
            'description' => $data['description'],
            'amount' => $data['price']
        ]);

        return $invoiceId;
    }

    private static function recordTransaction(int $serviceId, int $invoiceId)
    {
        Capsule::table('mod_service_transactions')->insert([
            'service_id' => $serviceId,
            'invoice_id' => $invoiceId,
            'action' => 'provisioning_initiated',
            'created_at' => date('Y-m-d H:i:s')
        ]);
    }

    private static function provisionService(int $serviceId): array
    {
        $service = Capsule::table('tblhosting')
            ->where('id', $serviceId)
            ->first();

        // Call provisioning module
        return ServerAPI::createAccount($service->server, [
            'serviceid' => $serviceId
        ]);
    }
}
```

### Step 4: Create Billing Transaction
```php
<?php
// /includes/transactions/BillingTransaction.php

class BillingTransaction
{
    /**
     * Process payment within transaction
     */
    public static function processPayment(int $invoiceId, string $paymentMethod, float $amount): array
    {
        return TransactionHelper::run(function() use ($invoiceId, $paymentMethod, $amount) {
            // 1. Get invoice details
            $invoice = Capsule::table('tblinvoices')
                ->where('id', $invoiceId)
                ->first();

            if (!$invoice) {
                throw new Exception("Invoice not found: {$invoiceId}");
            }

            if ($invoice->status === 'Paid') {
                throw new Exception("Invoice already paid");
            }

            // 2. Process payment with gateway
            $paymentResult = PaymentGateway::process($paymentMethod, [
                'invoice_id' => $invoiceId,
                'amount' => $amount,
                'client_id' => $invoice->userid
            ]);

            if (!$paymentResult['success']) {
                throw new Exception("Payment failed: " . $paymentResult['error']);
            }

            // 3. Update invoice status
            Capsule::table('tblinvoices')
                ->where('id', $invoiceId)
                ->update([
                    'status' => 'Paid',
                    'datepaid' => date('Y-m-d H:i:s'),
                    'paymentmethod' => $paymentMethod
                ]);

            // 4. Apply credit if any overpayment
            $overpayment = $amount - $invoice->total;
            if ($overpayment > 0) {
                self::applyCredit($invoice->userid, $overpayment);
            }

            // 5. Activate services
            self::activateServices($invoiceId);

            // 6. Record transaction
            Capsule::table('tblaccounts')->insert([
                'userid' => $invoice->userid,
                'invoiceid' => $invoiceId,
                'description' => "Invoice #{$invoiceId} Payment",
                'amountin' => $amount,
                'amountout' => 0,
                'fees' => $paymentResult['fee'] ?? 0,
                'paymentmethod' => $paymentMethod,
                'date' => date('Y-m-d H:i:s')
            ]);

            // 7. Update affiliate commission
            self::creditAffiliate($invoiceId);

            return [
                'success' => true,
                'transaction_id' => $paymentResult['transaction_id'],
                'amount' => $amount
            ];
        });
    }

    private static function applyCredit(int $clientId, float $amount)
    {
        Capsule::table('tblclients')
            ->where('id', $clientId)
            ->increment('credit', $amount);
    }

    private static function activateServices(int $invoiceId)
    {
        // Get services on invoice
        $items = Capsule::table('tblinvoiceitems')
            ->where('invoiceid', $invoiceId)
            ->where('type', 'Hosting')
            ->get();

        foreach ($items as $item) {
            Capsule::table('tblhosting')
                ->where('id', $item->relid)
                ->update([
                    'domainstatus' => 'Active',
                    'nextduedate' => getNextDueDate()
                ]);
        }
    }

    private static function creditAffiliate(int $invoiceId)
    {
        $invoice = Capsule::table('tblinvoices')
            ->where('id', $invoiceId)
            ->first();

        $affiliateId = Capsule::table('tblaffiliates')
            ->where('clientid', $invoice->userid)
            ->value('id');

        if ($affiliateId) {
            $commission = $invoice->total * 0.1; // 10% commission

            Capsule::table('tblaffiliates')
                ->where('id', $affiliateId)
                ->increment('balance', $commission);
        }
    }
}
```

### Step 5: Implement Rollback Hooks
```php
<?php
// /includes/transactions/RollbackManager.php

class RollbackManager
{
    private $stack = [];

    /**
     * Register action for potential rollback
     */
    public function register(string $action, callable $rollback, array $data = []): string
    {
        $id = uniqid('action_');

        $this->stack[$id] = [
            'action' => $action,
            'rollback' => $rollback,
            'data' => $data,
            'executed' => false
        ];

        return $id;
    }

    /**
     * Mark action as successfully executed
     */
    public function markExecuted(string $id)
    {
        if (isset($this->stack[$id])) {
            $this->stack[$id]['executed'] = true;
        }
    }

    /**
     * Rollback all executed actions
     */
    public function rollback(): array
    {
        $results = [];
        $errors = [];

        // Rollback in reverse order
        $executedActions = array_filter($this->stack, fn($a) => $a['executed']);

        foreach (array_reverse($executedActions, true) as $id => $action) {
            try {
                $result = $action['rollback']($action['data']);
                $results[$action['action']] = $result;

                logActivity("Rollback successful: {$action['action']}");
            } catch (Exception $e) {
                $errors[$action['action']] = $e->getMessage();

                logActivity("Rollback failed: {$action['action']} - " . $e->getMessage());
            }
        }

        $this->stack = [];

        return [
            'rolled_back' => $results,
            'failed' => $errors
        ];
    }

    /**
     * Commit and clear
     */
    public function commit()
    {
        $this->stack = [];
    }
}

// Usage in transaction
$rollback = new RollbackManager();

try {
    Capsule::connection()->transaction(function() use ($rollback) {
        // Action 1: Create client
        $clientId = createClient($data);
        $rollback->register('create_client', fn($d) => deleteClient($d['client_id']), ['client_id' => $clientId]);

        // Action 2: Create service
        $serviceId = createService($clientId, $productId);
        $rollback->register('create_service', fn($d) => deleteService($d['service_id']), ['service_id' => $serviceId]);

        // ... more actions ...

        // All successful - mark executed
        foreach (array_keys($rollback->stack) as $id) {
            $rollback->markExecuted($id);
        }
    });

    $rollback->commit();
} catch (Exception $e) {
    $rollback->rollback();
    throw $e;
}
```

### Step 6: Create Idempotency Keys
```php
<?php
// /includes/transactions/IdempotencyManager.php

class IdempotencyManager
{
    private $keyPrefix = 'idemp_';

    /**
     * Execute with idempotency check
     */
    public function execute(string $key, callable $operation): array
    {
        // Check if already processed
        $existing = $this->getResult($key);

        if ($existing) {
            return [
                'idempotent' => true,
                'result' => json_decode($existing->result, true),
                'original_timestamp' => $existing->created_at
            ];
        }

        // Execute operation
        $result = $operation();

        // Store result
        $this->storeResult($key, $result);

        return [
            'idempotent' => false,
            'result' => $result
        ];
    }

    /**
     * Store idempotency record
     */
    public function storeResult(string $key, $result, int $ttl = 86400): bool
    {
        try {
            Capsule::table('mod_idempotency_keys')->insert([
                'idempotency_key' => $this->keyPrefix . $key,
                'result' => json_encode($result),
                'created_at' => date('Y-m-d H:i:s'),
                'expires_at' => date('Y-m-d H:i:s', time() + $ttl)
            ]);

            return true;
        } catch (Exception $e) {
            // Duplicate key - another process already stored result
            return true;
        }
    }

    /**
     * Get stored result
     */
    public function getResult(string $key): ?object
    {
        return Capsule::table('mod_idempotency_keys')
            ->where('idempotency_key', $this->keyPrefix . $key)
            ->where('expires_at', '>', date('Y-m-d H:i:s'))
            ->first();
    }
}

// Usage
$idempotency = new IdempotencyManager();

$result = $idempotency->execute("payment_{$invoiceId}_{$transactionId}", function() use ($invoiceId) {
    return BillingTransaction::processPayment($invoiceId, $paymentMethod, $amount);
});

if ($result['idempotent']) {
    logActivity("Duplicate payment request detected, returning cached result");
}

return $result['result'];
```

### Step 7: Monitor Transaction Health
```php
<?php
// Transaction monitoring

add_hook('DailyCronJob', 1, function($vars) {
    // Check for long-running transactions
    $longTransactions = Capsule::select("
        SELECT *
        FROM information_schema.innodb_trx
        WHERE trx_started < DATE_SUB(NOW(), INTERVAL 5 MINUTE)
    ");

    if (count($longTransactions) > 0) {
        sendAdminEmail('Long-Running Transactions Detected', [
            'count' => count($longTransactions),
            'transactions' => $longTransactions
        ]);
    }

    // Clean up expired idempotency keys
    Capsule::table('mod_idempotency_keys')
        ->where('expires_at', '<', date('Y-m-d H:i:s'))
        ->delete();
});
```

## Transaction Best Practices

1. **Keep transactions short** - Long locks cause problems
2. **Avoid nested transactions** - Use savepoints if needed
3. **Order operations** - Always acquire locks in same order
4. **Handle deadlocks** - Implement retry logic
5. **Use idempotency keys** - Prevent duplicate processing
6. **Log everything** - For debugging failed transactions

## Transaction Isolation Levels

| Level | Description | Use Case |
|-------|-------------|----------|
| READ UNCOMMITTED | Dirty reads allowed | Never recommended |
| READ COMMITTED | No dirty reads | Standard WHMCS |
| REPEATABLE READ | Consistent reads | When needed |
| SERIALIZABLE | Full locking | Rarely needed |

## Related Workflows
- [WHMCS Rollback Automation](./whmcs-rollback-automation.md)
- [WHMCS Error Recovery](./whmcs-error-recovery.md)
- [WHMCS Retry Logic](./whmcs-retry-logic.md)