# WHMCS Gift Card System Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Implement gift card management with issuance, redemption, and balance tracking.

## Database Schema

```php
<?php
// modules/addons/gift_cards/gift_cards.php

use WHMCS\Database\Capsule;

function gift_cards_config(): array {
    return [
        'name' => 'Gift Cards',
        'description' => 'Gift card management system',
        'version' => '1.0',
    ];
}

function gift_cards_activate(): array {
    Capsule::schema()->create('mod_gift_cards', function($t) {
        $t->increments('id');
        $t->string('card_number', 50)->unique();
        $t->string('card_code', 20)->unique();
        $t->decimal('initial_value', 10, 2);
        $t->decimal('current_balance', 10, 2);
        $t->decimal('min_balance', 10, 2)->default(0);
        $t->string('status', 20)->default('active');
        $t->string('type', 20)->default('single_use');
        $t->integer('purchaser_id')->unsigned()->nullable();
        $t->integer('recipient_id')->unsigned()->nullable();
        $t->string('recipient_email', 100)->nullable();
        $t->string('recipient_name', 100)->nullable();
        $t->text('message')->nullable();
        $t->date('valid_from')->nullable();
        $t->date('valid_until')->nullable();
        $t->timestamp('redeemed_at')->nullable();
        $t->timestamp('created_at')->useCurrent();
    });

    Capsule::schema()->create('mod_gift_card_transactions', function($t) {
        $t->increments('id');
        $t->integer('gift_card_id')->unsigned();
        $t->string('type', 30);
        $t->decimal('amount', 10, 2);
        $t->decimal('balance_before', 10, 2);
        $t->decimal('balance_after', 10, 2);
        $t->integer('order_id')->unsigned()->nullable();
        $t->integer('invoice_id')->unsigned()->nullable();
        $t->string('payment_method', 50)->nullable();
        $t->text('notes')->nullable();
        $t->timestamp('created_at')->useCurrent();
    });

    Capsule::schema()->create('mod_gift_card_templates', function($t) {
        $t->increments('id');
        $t->string('name', 100);
        $t->text('description')->nullable();
        $t->decimal('min_value', 10, 2)->default(10);
        $t->decimal('max_value', 10, 2)->default(1000);
        $t->string('denominations', 500)->nullable();
        $t->string('background_color', 7)->default('#4A90D9');
        $t->string('text_color', 7)->default('#FFFFFF');
        $t->text('custom_message')->nullable();
        $t->boolean('is_active')->default(true);
        $t->timestamps();
    });

    Capsule::schema()->create('mod_gift_card_purchases', function($t) {
        $t->increments('id');
        $t->integer('gift_card_id')->unsigned();
        $t->integer('buyer_id')->unsigned();
        $t->integer('order_id')->unsigned()->nullable();
        $t->decimal('amount', 10, 2);
        $t->string('delivery_method', 30)->default('email');
        $t->string('delivery_email', 100)->nullable();
        $t->timestamp('scheduled_delivery')->nullable();
        $t->timestamp('delivered_at')->nullable();
        $t->timestamps();
    });

    return ['status' => 'success'];
}

function gift_cards_deactivate(): array {
    Capsule::schema()->dropIfExists('mod_gift_card_purchases');
    Capsule::schema()->dropIfExists('mod_gift_card_templates');
    Capsule::schema()->dropIfExists('mod_gift_card_transactions');
    Capsule::schema()->dropIfExists('mod_gift_cards');
    return ['status' => 'success'];
}
```

## Gift Card Manager

```php
<?php
class GiftCardManager {
    public function generateCard(float $value, array $options = []): array {
        $cardNumber = $this->generateCardNumber();
        $cardCode = $this->generateCardCode();

        $cardId = Capsule::table('mod_gift_cards')->insertGetId([
            'card_number' => $cardNumber,
            'card_code' => $cardCode,
            'initial_value' => $value,
            'current_balance' => $value,
            'status' => 'active',
            'type' => $options['type'] ?? 'single_use',
            'purchaser_id' => $options['purchaser_id'] ?? null,
            'recipient_email' => $options['recipient_email'] ?? null,
            'recipient_name' => $options['recipient_name'] ?? null,
            'message' => $options['message'] ?? null,
            'valid_from' => $options['valid_from'] ?? date('Y-m-d'),
            'valid_until' => $options['valid_until'] ?? null,
        ]);

        $this->recordTransaction($cardId, 'issue', $value, 0, $value);

        return [
            'success' => true,
            'card_id' => $cardId,
            'card_number' => $cardNumber,
            'card_code' => $cardCode,
        ];
    }

    private function generateCardNumber(): string {
        do {
            $number = 'GC' . str_pad(mt_rand(1, 9999999999), 10, '0', STR_PAD_LEFT);

            $exists = Capsule::table('mod_gift_cards')
                ->where('card_number', $number)
                ->exists();
        } while ($exists);

        return $number;
    }

    private function generateCardCode(): string {
        return strtoupper(substr(md5(random_bytes(16)), 0, 12));
    }

    public function validateCard(string $identifier): array {
        $card = Capsule::table('mod_gift_cards')
            ->where('card_number', $identifier)
            ->orWhere('card_code', $identifier)
            ->first();

        if (!$card) {
            return ['valid' => false, 'error' => 'Card not found'];
        }

        if ($card->status !== 'active') {
            return ['valid' => false, 'error' => 'Card is ' . $card->status];
        }

        if ($card->valid_from && $card->valid_from > date('Y-m-d')) {
            return ['valid' => false, 'error' => 'Card is not yet valid'];
        }

        if ($card->valid_until && $card->valid_until < date('Y-m-d')) {
            return ['valid' => false, 'error' => 'Card has expired'];
        }

        if ($card->current_balance <= 0) {
            return ['valid' => false, 'error' => 'Card has no balance'];
        }

        return [
            'valid' => true,
            'card' => $card,
        ];
    }

    public function redeemCard(string $identifier, float $amount, int $userId, ?int $orderId = null): array {
        $validation = $this->validateCard($identifier);

        if (!$validation['valid']) {
            return $validation;
        }

        $card = $validation['card'];

        if ($amount > $card->current_balance) {
            $amount = $card->current_balance;
        }

        $newBalance = $card->current_balance - $amount;

        Capsule::table('mod_gift_cards')
            ->where('id', $card->id)
            ->update([
                'current_balance' => $newBalance,
                'recipient_id' => $userId,
            ]);

        if ($newBalance <= 0 || $card->type === 'single_use') {
            Capsule::table('mod_gift_cards')
                ->where('id', $card->id)
                ->update([
                    'status' => 'redeemed',
                    'redeemed_at' => date('Y-m-d H:i:s'),
                ]);
        }

        $this->recordTransaction($card->id, 'redeem', -$amount, $card->current_balance, $newBalance, $orderId);

        return [
            'success' => true,
            'amount_applied' => $amount,
            'remaining_balance' => $newBalance,
        ];
    }

    public function addBalance(string $identifier, float $amount, ?string $paymentMethod = null): array {
        $card = Capsule::table('mod_gift_cards')
            ->where('card_number', $identifier)
            ->orWhere('card_code', $identifier)
            ->first();

        if (!$card) {
            return ['success' => false, 'error' => 'Card not found'];
        }

        $newBalance = $card->current_balance + $amount;

        Capsule::table('mod_gift_cards')
            ->where('id', $card->id)
            ->update(['current_balance' => $newBalance]);

        $this->recordTransaction($card->id, 'reload', $amount, $card->current_balance, $newBalance, null, null, $paymentMethod);

        return [
            'success' => true,
            'new_balance' => $newBalance,
        ];
    }

    private function recordTransaction(
        int $cardId,
        string $type,
        float $amount,
        float $balanceBefore,
        float $balanceAfter,
        ?int $orderId = null,
        ?int $invoiceId = null,
        ?string $paymentMethod = null
    ): void {
        Capsule::table('mod_gift_card_transactions')->insert([
            'gift_card_id' => $cardId,
            'type' => $type,
            'amount' => $amount,
            'balance_before' => $balanceBefore,
            'balance_after' => $balanceAfter,
            'order_id' => $orderId,
            'invoice_id' => $invoiceId,
            'payment_method' => $paymentMethod,
        ]);
    }

    public function getCardByCode(string $code): ?object {
        return Capsule::table('mod_gift_cards')
            ->where('card_code', $code)
            ->orWhere('card_number', $code)
            ->first();
    }

    public function getCardHistory(int $cardId): array {
        return Capsule::table('mod_gift_card_transactions')
            ->where('gift_card_id', $cardId)
            ->orderBy('created_at', 'desc')
            ->get();
    }

    public function checkBalance(string $identifier): array {
        $validation = $this->validateCard($identifier);

        if (!$validation['valid']) {
            return $validation;
        }

        return [
            'valid' => true,
            'card' => $validation['card'],
            'balance' => $validation['card']->current_balance,
            'valid_until' => $validation['card']->valid_until,
        ];
    }

    public function bulkGenerate(int $count, float $value, array $options = []): array {
        $cards = [];

        for ($i = 0; $i < $count; $i++) {
            $result = $this->generateCard($value, $options);
            $cards[] = $result;
        }

        return [
            'success' => true,
            'count' => count($cards),
            'cards' => $cards,
        ];
    }

    public function getAnalytics(): array {
        $stats = [
            'total_cards' => Capsule::table('mod_gift_cards')->count(),
            'active_cards' => Capsule::table('mod_gift_cards')->where('status', 'active')->count(),
            'total_value_issued' => Capsule::table('mod_gift_cards')->sum('initial_value'),
            'total_balance' => Capsule::table('mod_gift_cards')->where('status', 'active')->sum('current_balance'),
            'total_redeemed' => Capsule::table('mod_gift_cards')->sum('initial_value') -
                               Capsule::table('mod_gift_cards')->sum('current_balance'),
        ];

        return $stats;
    }
}
```

## Cart Integration

```php
<?php
add_hook('ShoppingCartValidateCheckout', 1, function($vars) {
    $giftCardCode = $_SESSION['cart']['gift_card_code'] ?? null;

    if (!$giftCardCode) return null;

    $manager = new GiftCardManager();
    $result = $manager->checkBalance($giftCardCode);

    if (!$result['valid']) {
        unset($_SESSION['cart']['gift_card_code']);
        return ['allowed' => false, 'errormessage' => $result['error']];
    }

    return null;
});

add_hook('AfterCartCalculateTotals', 1, function($vars) {
    $giftCardCode = $_SESSION['cart']['gift_card_code'] ?? null;

    if (!$giftCardCode) return;

    $manager = new GiftCardManager();
    $balance = $manager->checkBalance($giftCardCode);

    if ($balance['valid']) {
        $maxCredit = min($balance['balance'], $vars['total']);
        $_SESSION['cart']['gift_card_credit'] = $maxCredit;
    }
});

add_hook('OrderPaymentComplete', 1, function($vars) {
    $giftCardCode = $_SESSION['cart']['gift_card_code'] ?? null;
    $credit = $_SESSION['cart']['gift_card_credit'] ?? 0;

    if ($giftCardCode && $credit > 0) {
        $manager = new GiftCardManager();
        $manager->redeemCard($giftCardCode, $credit, $vars['userid'], $vars['orderid']);

        unset($_SESSION['cart']['gift_card_code']);
        unset($_SESSION['cart']['gift_card_credit']);
    }
});
```

## Client Area

```php
<?php
function gift_cards_clientarea(array $vars): array {
    $userId = $_SESSION['uid'];
    $manager = new GiftCardManager();

    $userCards = Capsule::table('mod_gift_cards')
        ->where(function($q) use ($userId) {
            $q->where('purchaser_id', $userId)
              ->orWhere('recipient_id', $userId);
        })
        ->get();

    $balance = $_POST['check_balance'] ?? null;
    $balanceResult = null;

    if ($balance) {
        $balanceResult = $manager->checkBalance($balance);
    }

    return [
        'pagetitle' => 'Gift Cards',
        'templatefile' => 'gift_cards',
        'vars' => [
            'user_cards' => $userCards,
            'balance_result' => $balanceResult,
        ],
    ];
}
```

---

**Related Skills:**
- whmcs-wallet-system
- whmcs-promotional-codes
- whmcs-store-credit