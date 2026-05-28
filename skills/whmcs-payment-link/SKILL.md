# WHMCS Payment Link Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Build payment link generation for offline payments and manual invoices.

## Payment Link Module

```php
<?php
/**
 * Payment Link Module (Addon)
 * Location: modules/addons/{module}/
 */

function {module}_activate(): array {
    Capsule::schema()->create('mod_payment_links', function($t) {
        $t->increments('id');
        $t->string('link_code', 50)->unique();
        $t->integer('invoice_id')->unsigned()->nullable();
        $t->integer('user_id')->unsigned()->nullable();
        $t->decimal('amount', 10, 2)->nullable();
        $t->string('description', 255)->nullable();
        $t->string('status', 20)->default('active');
        $t->integer('max_uses')->unsigned()->nullable();
        $t->integer('use_count')->unsigned()->default(0);
        $t->timestamp('expires_at')->nullable();
        $t->timestamps();

        $t->index('link_code');
        $t->index('status');
    });

    return ['status' => 'success'];
}

function {module}_deactivate(): array {
    Capsule::schema()->dropIfExists('mod_payment_links');
    return ['status' => 'success'];
}
```

## Payment Link Generator

```php
<?php
class PaymentLinkGenerator {
    public function generateLink(array $options): string {
        $linkCode = $this->generateCode();

        $data = [
            'link_code' => $linkCode,
            'invoice_id' => $options['invoice_id'] ?? null,
            'user_id' => $options['user_id'] ?? null,
            'amount' => $options['amount'] ?? null,
            'description' => $options['description'] ?? 'Payment Link',
            'status' => 'active',
            'max_uses' => $options['max_uses'] ?? 1,
            'expires_at' => $options['expires_at'] ?? null,
        ];

        Capsule::table('mod_payment_links')->insert($data);

        $baseUrl = Capsule::table('tblconfiguration')
            ->where('setting', 'SystemURL')
            ->value('value');

        return rtrim($baseUrl, '/') . '/pay.php?code=' . $linkCode;
    }

    public function validateLink(string $code): array {
        $link = Capsule::table('mod_payment_links')
            ->where('link_code', $code)
            ->first();

        if (!$link) {
            return ['valid' => false, 'error' => 'Invalid payment link'];
        }

        if ($link->status !== 'active') {
            return ['valid' => false, 'error' => 'Payment link is no longer active'];
        }

        if ($link->expires_at && $link->expires_at < date('Y-m-d H:i:s')) {
            return ['valid' => false, 'error' => 'Payment link has expired'];
        }

        if ($link->max_uses && $link->use_count >= $link->max_uses) {
            return ['valid' => false, 'error' => 'Payment link usage limit reached'];
        }

        return ['valid' => true, 'link' => $link];
    }

    public function recordUse(string $code, int $invoiceId, string $transactionId): void {
        Capsule::table('mod_payment_links')
            ->where('link_code', $code)
            ->increment('use_count');

        if ($this->shouldDeactivate($code)) {
            Capsule::table('mod_payment_links')
                ->where('link_code', $code)
                ->update(['status' => 'used']);
        }
    }

    private function generateCode(int $length = 32): string {
        return bin2hex(random_bytes($length / 2));
    }

    private function shouldDeactivate(string $code): bool {
        $link = Capsule::table('mod_payment_links')->where('link_code', $code)->first();
        return $link->max_uses && $link->use_count >= $link->max_uses;
    }
}
```

---

**Related Skills:**
- whmcs-addon-builder
- whmcs-gateway-builder