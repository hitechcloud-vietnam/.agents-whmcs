# WHMCS Store Credit Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Build store credit modules for customer balance management.

## Credit Module Structure

```php
<?php
/**
 * Store Credit Module (Addon)
 * Location: modules/addons/{module}/
 */

function {module}_config(): array {
    return [
        'name' => 'Store Credit',
        'description' => 'Customer credit balance management',
        'version' => '1.0',
    ];
}

function {module}_activate(): array {
    Capsule::schema()->create('mod_credit_accounts', function($t) {
        $t->increments('id');
        $t->integer('user_id')->unsigned()->unique();
        $t->decimal('balance', 12, 2)->default(0);
        $t->decimal('total_added', 12, 2)->default(0);
        $t->decimal('total_used', 12, 2)->default(0);
        $t->timestamps();
    });

    Capsule::schema()->create('mod_credit_transactions', function($t) {
        $t->increments('id');
        $t->integer('account_id')->unsigned();
        $t->string('type', 20);
        $t->decimal('amount', 12, 2);
        $t->string('reference_type', 50)->nullable();
        $t->integer('reference_id')->unsigned()->nullable();
        $t->string('description', 255);
        $t->integer('admin_id')->unsigned()->nullable();
        $t->timestamps();

        $t->index('account_id');
        $t->index(['type', 'created_at']);
    });

    return ['status' => 'success', 'description' => 'Credit module activated'];
}

function {module}_deactivate(): array {
    Capsule::schema()->dropIfExists('mod_credit_accounts');
    Capsule::schema()->dropIfExists('mod_credit_transactions');
    return ['status' => 'success'];
}
```

## Credit Operations

```php
<?php
class CreditManager {
    public function getBalance(int $userId): float {
        $account = Capsule::table('mod_credit_accounts')
            ->where('user_id', $userId)
            ->first();

        return $account ? (float) $account->balance : 0;
    }

    public function addCredit(int $userId, float $amount, string $description, ?int $adminId = null): array {
        if ($amount <= 0) {
            return ['success' => false, 'error' => 'Amount must be positive'];
        }

        $account = $this->getOrCreateAccount($userId);

        Capsule::connection()->transaction(function() use ($account, $amount, $description, $adminId) {
            Capsule::table('mod_credit_accounts')
                ->where('id', $account->id)
                ->update([
                    'balance' => $account->balance + $amount,
                    'total_added' => $account->total_added + $amount,
                ]);

            Capsule::table('mod_credit_transactions')->insert([
                'account_id' => $account->id,
                'type' => 'credit',
                'amount' => $amount,
                'description' => $description,
                'admin_id' => $adminId,
            ]);
        });

        return ['success' => true, 'new_balance' => $this->getBalance($userId)];
    }

    public function useCredit(int $userId, float $amount, string $referenceType, int $referenceId): array {
        if ($amount <= 0) {
            return ['success' => false, 'error' => 'Amount must be positive'];
        }

        $balance = $this->getBalance($userId);
        if ($balance < $amount) {
            return ['success' => false, 'error' => 'Insufficient balance'];
        }

        $account = Capsule::table('mod_credit_accounts')
            ->where('user_id', $userId)
            ->first();

        Capsule::connection()->transaction(function() use ($account, $amount, $referenceType, $referenceId) {
            Capsule::table('mod_credit_accounts')
                ->where('id', $account->id)
                ->update([
                    'balance' => $account->balance - $amount,
                    'total_used' => $account->total_used + $amount,
                ]);

            Capsule::table('mod_credit_transactions')->insert([
                'account_id' => $account->id,
                'type' => 'debit',
                'amount' => $amount,
                'reference_type' => $referenceType,
                'reference_id' => $referenceId,
                'description' => "Used for {$referenceType} #{$referenceId}",
            ]);
        });

        return ['success' => true, 'new_balance' => $this->getBalance($userId)];
    }

    public function getTransactions(int $userId, int $limit = 50): array {
        $account = Capsule::table('mod_credit_accounts')
            ->where('user_id', $userId)
            ->first();

        if (!$account) return [];

        return Capsule::table('mod_credit_transactions')
            ->where('account_id', $account->id)
            ->orderBy('created_at', 'desc')
            ->limit($limit)
            ->get()
            ->toArray();
    }
}
```

## Invoice Integration

```php
add_hook('InvoiceCreated', 1, function($vars) {
    $invoice = Capsule::table('tblinvoices')
        ->where('id', $vars['invoiceid'])
        ->first();

    $creditManager = new CreditManager();
    $balance = $creditManager->getBalance($invoice->userid);

    if ($balance > 0 && $invoice->total > 0) {
        $creditToApply = min($balance, $invoice->total);

        $creditManager->useCredit(
            $invoice->userid,
            $creditToApply,
            'invoice',
            $invoice->id
        );

        addInvoicePayment($invoice->id, 0, $creditToApply, 0, 'credit');
    }
});
```

---

**Related Skills:**
- whmcs-addon-builder
- whmcs-service-billing