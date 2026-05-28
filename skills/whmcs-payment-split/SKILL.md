# WHMCS Payment Split Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Build payment split modules for marketplace and multi-vendor scenarios.

## Payment Split Module

```php
<?php
/**
 * Payment Split Module (Addon)
 * Location: modules/addons/{module}/
 */

function {module}_config(): array {
    return [
        'name' => 'Payment Split',
        'description' => 'Split payments between vendors',
        'version' => '1.0',
    ];
}

function {module}_activate(): array {
    Capsule::schema()->create('mod_payment_splits', function($t) {
        $t->increments('id');
        $t->integer('invoice_id')->unsigned();
        $t->integer('vendor_id')->unsigned();
        $t->decimal('amount', 10, 2);
        $t->string('status', 20)->default('pending');
        $t->decimal('fee', 10, 2)->default(0);
        $t->integer('payout_id')->unsigned()->nullable();
        $t->timestamps();

        $t->index(['invoice_id', 'status']);
        $t->index(['vendor_id', 'status']);
    });

    Capsule::schema()->create('mod_vendors', function($t) {
        $t->increments('id');
        $t->string('name', 100);
        $t->string('email', 150);
        $t->decimal('commission_rate', 5, 2)->default(0);
        $t->string('payout_method', 50)->default('bank');
        $t->text('payout_details');
        $t->boolean('is_active')->default(true);
        $t->timestamps();
    });

    return ['status' => 'success'];
}

function {module}_deactivate(): array {
    Capsule::schema()->dropIfExists('mod_payment_splits');
    Capsule::schema()->dropIfExists('mod_vendors');
    return ['status' => 'success'];
}
```

## Split Logic

```php
<?php
class PaymentSplitter {
    public function splitPayment(int $invoiceId, array $items): array {
        $invoice = Capsule::table('tblinvoices')->where('id', $invoiceId)->first();
        $splits = [];

        foreach ($items as $item) {
            $vendor = $this->getVendorForItem($item['product_id'] ?? 0);

            if ($vendor) {
                $commission = $item['amount'] * ($vendor->commission_rate / 100);
                $vendorAmount = $item['amount'] - $commission;

                $splitId = Capsule::table('mod_payment_splits')->insertGetId([
                    'invoice_id' => $invoiceId,
                    'vendor_id' => $vendor->id,
                    'amount' => $vendorAmount,
                    'fee' => $commission,
                    'status' => 'pending',
                ]);

                $splits[] = [
                    'split_id' => $splitId,
                    'vendor_id' => $vendor->id,
                    'vendor_name' => $vendor->name,
                    'amount' => $vendorAmount,
                    'fee' => $commission,
                ];
            } else {
                $splits[] = [
                    'vendor_id' => 0,
                    'vendor_name' => 'Platform',
                    'amount' => $item['amount'],
                    'fee' => 0,
                ];
            }
        }

        return $splits;
    }

    public function markPaid(int $invoiceId): void {
        Capsule::table('mod_payment_splits')
            ->where('invoice_id', $invoiceId)
            ->update(['status' => 'paid']);
    }
}
```

---

**Related Skills:**
- whmcs-addon-builder
- whmcs-gateway-builder