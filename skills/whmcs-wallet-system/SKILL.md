# WHMCS Wallet System Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Digital wallet for clients to deposit funds, manage balances, and pay invoices.

## Database Schema

```php
<?php
// modules/addons/wallet_system/wallet_system.php

use WHMCS\Database\Capsule;

function wallet_system_config(): array {
    return [
        'name' => 'Wallet System',
        'description' => 'Digital wallet for client funds',
        'version' => '1.0',
    ];
}

function wallet_system_activate(): array {
    Capsule::schema()->create('mod_wallet_balances', function($t) {
        $t->increments('id');
        $t->integer('user_id')->unsigned()->unique();
        $t->decimal('balance', 12, 2)->default(0);
        $t->decimal('pending_balance', 12, 2)->default(0);
        $t->decimal('total_deposits', 12, 2)->default(0);
        $t->decimal('total_withdrawals', 12, 2)->default(0);
        $t->timestamp('last_activity_at')->nullable();
        $t->timestamps();
    });

    Capsule::schema()->create('mod_wallet_transactions', function($t) {
        $t->increments('id');
        $t->integer('user_id')->unsigned();
        $t->string('type', 30);
        $t->decimal('amount', 10, 2);
        $t->decimal('balance_before', 12, 2);
        $t->decimal('balance_after', 12, 2);
        $t->string('payment_method', 50)->nullable();
        $t->string('reference_type', 50)->nullable();
        $t->integer('reference_id')->unsigned()->nullable();
        $t->text('description')->nullable();
        $t->string('status', 20)->default('completed');
        $t->string('transaction_ref', 100)->unique();
        $t->timestamp('processed_at')->nullable();
        $t->timestamps();
    });

    Capsule::schema()->create('mod_wallet_deposits', function($t) {
        $t->increments('id');
        $t->integer('user_id')->unsigned();
        $t->string('transaction_ref', 100)->unique();
        $t->decimal('amount', 10, 2);
        $t->string('payment_method', 50);
        $t->string('status', 20)->default('pending');
        $t->string('gateway_ref', 100)->nullable();
        $t->text('gateway_response')->nullable();
        $t->timestamp('completed_at')->nullable();
        $t->timestamps();
    });

    Capsule::schema()->create('mod_wallet_withdrawals', function($t) {
        $t->increments('id');
        $t->integer('user_id')->unsigned();
        $t->string('transaction_ref', 100)->unique();
        $t->decimal('amount', 10, 2);
        $t->string('withdrawal_method', 50);
        $t->text('withdrawal_details')->nullable();
        $t->string('status', 20)->default('pending');
        $t->text('admin_notes')->nullable();
        $t->timestamp('processed_at')->nullable();
        $t->timestamps();
    });

    Capsule::schema()->create('mod_wallet_limits', function($t) {
        $t->increments('id');
        $t->integer('user_id')->unsigned();
        $t->decimal('daily_limit', 10, 2)->default(1000);
        $t->decimal('monthly_limit', 10, 2)->default(5000);
        $t->decimal('min_withdrawal', 10, 2)->default(20);
        $t->boolean('require_verification')->default(false);
        $t->timestamps();
    });

    return ['status' => 'success'];
}

function wallet_system_deactivate(): array {
    Capsule::schema()->dropIfExists('mod_wallet_limits');
    Capsule::schema()->dropIfExists('mod_wallet_withdrawals');
    Capsule::schema()->dropIfExists('mod_wallet_deposits');
    Capsule::schema()->dropIfExists('mod_wallet_transactions');
    Capsule::schema()->dropIfExists('mod_wallet_balances');
    return ['status' => 'success'];
}
```

## Wallet Manager

```php
<?php
class WalletManager {
    public function getBalance(int $userId): object {
        $wallet = Capsule::table('mod_wallet_balances')
            ->where('user_id', $userId)
            ->first();

        if (!$wallet) {
            $wallet = (object)[
                'balance' => 0,
                'pending_balance' => 0,
                'total_deposits' => 0,
                'total_withdrawals' => 0,
            ];

            Capsule::table('mod_wallet_balances')->insert([
                'user_id' => $userId,
                'balance' => 0,
            ]);
        }

        return $wallet;
    }

    public function deposit(int $userId, float $amount, string $paymentMethod, array $gatewayData = []): array {
        if ($amount <= 0) {
            return ['success' => false, 'error' => 'Invalid amount'];
        }

        $transactionRef = $this->generateTransactionRef('DEP');

        $depositId = Capsule::table('mod_wallet_deposits')->insertGetId([
            'user_id' => $userId,
            'transaction_ref' => $transactionRef,
            'amount' => $amount,
            'payment_method' => $paymentMethod,
            'status' => 'pending',
            'gateway_response' => json_encode($gatewayData),
        ]);

        return [
            'success' => true,
            'deposit_id' => $depositId,
            'transaction_ref' => $transactionRef,
        ];
    }

    public function confirmDeposit(string $transactionRef): array {
        $deposit = Capsule::table('mod_wallet_deposits')
            ->where('transaction_ref', $transactionRef)
            ->where('status', 'pending')
            ->first();

        if (!$deposit) {
            return ['success' => false, 'error' => 'Deposit not found'];
        }

        $userId = $deposit->user_id;
        $amount = $deposit->amount;

        $currentBalance = $this->getBalance($userId)->balance;

        // Update wallet balance
        Capsule::table('mod_wallet_balances')
            ->where('user_id', $userId)
            ->update([
                'balance' => $currentBalance + $amount,
                'total_deposits' => Capsule::raw('total_deposits + ' . $amount),
                'last_activity_at' => date('Y-m-d H:i:s'),
            ]);

        // Record transaction
        Capsule::table('mod_wallet_transactions')->insert([
            'user_id' => $userId,
            'type' => 'deposit',
            'amount' => $amount,
            'balance_before' => $currentBalance,
            'balance_after' => $currentBalance + $amount,
            'payment_method' => $deposit->payment_method,
            'reference_type' => 'deposit',
            'reference_id' => $deposit->id,
            'status' => 'completed',
            'transaction_ref' => $transactionRef,
            'processed_at' => date('Y-m-d H:i:s'),
        ]);

        // Update deposit status
        Capsule::table('mod_wallet_deposits')
            ->where('id', $deposit->id)
            ->update([
                'status' => 'completed',
                'completed_at' => date('Y-m-d H:i:s'),
            ]);

        return [
            'success' => true,
            'new_balance' => $currentBalance + $amount,
        ];
    }

    public function withdraw(int $userId, float $amount, string $method, array $details = []): array {
        if ($amount <= 0) {
            return ['success' => false, 'error' => 'Invalid amount'];
        }

        $limits = $this->getLimits($userId);

        if ($amount < $limits->min_withdrawal) {
            return ['success' => false, 'error' => "Minimum withdrawal: $" . number_format($limits->min_withdrawal, 2)];
        }

        $balance = $this->getBalance($userId);

        if ($amount > $balance->balance) {
            return ['success' => false, 'error' => 'Insufficient balance'];
        }

        // Check daily/monthly limits
        $dailyTotal = $this->getPeriodTotal($userId, 'withdrawal', 'daily');
        $monthlyTotal = $this->getPeriodTotal($userId, 'withdrawal', 'monthly');

        if ($dailyTotal + $amount > $limits->daily_limit) {
            return ['success' => false, 'error' => 'Daily withdrawal limit exceeded'];
        }

        if ($monthlyTotal + $amount > $limits->monthly_limit) {
            return ['success' => false, 'error' => 'Monthly withdrawal limit exceeded'];
        }

        $transactionRef = $this->generateTransactionRef('WDR');

        Capsule::table('mod_wallet_withdrawals')->insert([
            'user_id' => $userId,
            'transaction_ref' => $transactionRef,
            'amount' => $amount,
            'withdrawal_method' => $method,
            'withdrawal_details' => json_encode($details),
            'status' => 'pending',
        ]);

        return [
            'success' => true,
            'transaction_ref' => $transactionRef,
            'message' => 'Withdrawal request submitted',
        ];
    }

    public function deduct(int $userId, float $amount, string $description, ?int $invoiceId = null): array {
        $balance = $this->getBalance($userId);

        if ($amount > $balance->balance) {
            return ['success' => false, 'error' => 'Insufficient balance'];
        }

        $transactionRef = $this->generateTransactionRef('DED');

        Capsule::table('mod_wallet_balances')
            ->where('user_id', $userId)
            ->update([
                'balance' => $balance->balance - $amount,
                'total_withdrawals' => Capsule::raw('total_withdrawals + ' . $amount),
                'last_activity_at' => date('Y-m-d H:i:s'),
            ]);

        Capsule::table('mod_wallet_transactions')->insert([
            'user_id' => $userId,
            'type' => 'deduction',
            'amount' => -$amount,
            'balance_before' => $balance->balance,
            'balance_after' => $balance->balance - $amount,
            'description' => $description,
            'reference_type' => 'invoice',
            'reference_id' => $invoiceId,
            'status' => 'completed',
            'transaction_ref' => $transactionRef,
            'processed_at' => date('Y-m-d H:i:s'),
        ]);

        return [
            'success' => true,
            'new_balance' => $balance->balance - $amount,
        ];
    }

    public function payInvoice(int $userId, int $invoiceId): array {
        $invoice = Capsule::table('tblinvoices')->find($invoiceId);

        if (!$invoice || $invoice->status !== 'Unpaid') {
            return ['success' => false, 'error' => 'Invalid invoice'];
        }

        $balance = $this->getBalance($userId);

        if ($invoice->total > $balance->balance) {
            return ['success' => false, 'error' => 'Insufficient wallet balance'];
        }

        $result = $this->deduct($userId, $invoice->total, "Payment for invoice #{$invoiceId}", $invoiceId);

        if ($result['success']) {
            localAPI('addInvoicePayment', [
                'invoiceid' => $invoiceId,
                'amount' => $invoice->total,
                'paymentmethod' => 'wallet',
            ]);
        }

        return $result;
    }

    private function generateTransactionRef(string $prefix): string {
        return $prefix . date('Ymd') . strtoupper(substr(md5(random_bytes(8)), 0, 8));
    }

    private function getLimits(int $userId): object {
        $limits = Capsule::table('mod_wallet_limits')
            ->where('user_id', $userId)
            ->first();

        if (!$limits) {
            $limits = (object)[
                'daily_limit' => 1000,
                'monthly_limit' => 5000,
                'min_withdrawal' => 20,
            ];
        }

        return $limits;
    }

    private function getPeriodTotal(int $userId, string $type, string $period): float {
        $date = match($period) {
            'daily' => date('Y-m-d'),
            'monthly' => date('Y-m'),
            default => date('Y-m-d'),
        };

        $startDate = match($period) {
            'daily' => date('Y-m-d 00:00:00'),
            'monthly' => date('Y-m-01 00:00:00'),
            default => date('Y-m-d 00:00:00'),
        };

        $result = Capsule::table('mod_wallet_transactions')
            ->where('user_id', $userId)
            ->where('type', $type)
            ->where('created_at', '>=', $startDate)
            ->where('status', 'completed')
            ->sum('amount');

        return $result ?? 0;
    }

    public function getTransactionHistory(int $userId, int $limit = 50, string $type = null): array {
        $query = Capsule::table('mod_wallet_transactions')
            ->where('user_id', $userId)
            ->orderBy('created_at', 'desc')
            ->limit($limit);

        if ($type) {
            $query->where('type', $type);
        }

        return $query->get();
    }

    public function autoTopup(int $userId, float $threshold, float $amount): array {
        $balance = $this->getBalance($userId);

        if ($balance->balance >= $threshold) {
            return ['success' => false, 'error' => 'Balance above threshold'];
        }

        return $this->deposit($userId, $amount, 'auto_topup');
    }
}
```

## Payment Gateway Integration

```php
<?php
function wallet_system_link(array $params): string {
    $userId = $_SESSION['uid'] ?? 0;
    $manager = new WalletManager();

    $balance = $manager->getBalance($userId);

    $html = '<div class="wallet-gateway">';
    $html .= '<p>Current Wallet Balance: <strong>$' . number_format($balance->balance, 2) . '</strong></p>';
    $html .= '<button type="submit" name="paymentmethod" value="wallet" class="btn btn-primary">';
    $html .= 'Pay with Wallet</button>';
    $html .= '</div>';

    return $html;
}
```

## Client Area

```php
<?php
function wallet_system_clientarea(array $vars): array {
    $userId = $_SESSION['uid'];
    $manager = new WalletManager();

    $balance = $manager->getBalance($userId);
    $transactions = $manager->getTransactionHistory($userId, 20);
    $pendingWithdrawals = Capsule::table('mod_wallet_withdrawals')
        ->where('user_id', $userId)
        ->where('status', 'pending')
        ->get();

    return [
        'pagetitle' => 'My Wallet',
        'templatefile' => 'wallet',
        'vars' => [
            'balance' => $balance->balance,
            'pending_balance' => $balance->pending_balance,
            'total_deposits' => $balance->total_deposits,
            'total_withdrawals' => $balance->total_withdrawals,
            'transactions' => $transactions,
            'pending_withdrawals' => $pendingWithdrawals,
        ],
    ];
}
```

---

**Related Skills:**
- whmcs-gift-cards
- whmcs-store-credit
- whmcs-payment-split