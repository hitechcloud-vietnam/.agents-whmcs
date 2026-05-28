# WHMCS Service Billing Module Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Guide for building billing cycle and invoice management modules.

## When to Use

- Creating custom billing cycles
- Building proration calculators
- Managing prepaid/postpaid billing

## Billing Module Patterns

```php
<?php
// modules/addons/{billingmodule}/{billingmodule}.php
if (!defined("WHMCS")) { die("Direct access denied"); }

function {billingmodule}_config(): array {
    return [
        'name' => 'Advanced Billing Manager',
        'description' => 'Custom billing cycles and proration',
        'version' => '1.0',
        'author' => 'Author',
        'default_term' => ['FriendlyName' => 'Default Billing Term', 'Type' => 'dropdown',
            'Options' => 'monthly,quarterly,semi_annual,annual'],
        'proration_method' => ['FriendlyName' => 'Proration Method', 'Type' => 'dropdown',
            'Options' => 'exact_days,30_day,billing_date'],
    ];
}

function {billingmodule}_activate(): array {
    Capsule::schema()->create('mod_billing_custom_cycles', function($t) {
        $t->increments('id');
        $t->string('name', 100);
        $t->integer('days')->unsigned();
        $t->string('billing_day', 10)->default('same');
        $t->boolean('active')->default(true);
    });

    Capsule::schema()->create('mod_billing_prorations', function($t) {
        $t->increments('id');
        $t->integer('service_id')->unsigned();
        $t->string('type', 50);
        $t->decimal('amount', 10, 2);
        $t->date('effective_date');
        $t->timestamp('created_at');
    });

    Capsule::schema()->create('mod_billing_adjustments', function($t) {
        $t->increments('id');
        $t->integer('service_id')->unsigned();
        $t->integer('invoice_id')->unsigned();
        $t->string('type', 50);
        $t->decimal('amount', 10, 2);
        $t->text('notes')->nullable();
        $t->timestamp('created_at');
    });

    return ['status' => 'success', 'description' => 'Billing module activated'];
}

function {billingmodule}_deactivate(): array {
    Capsule::schema()->dropIfExists('mod_billing_custom_cycles');
    Capsule::schema()->dropIfExists('mod_billing_prorations');
    Capsule::schema()->dropIfExists('mod_billing_adjustments');
    return ['status' => 'success', 'description' => 'Billing module deactivated'];
}
```

### Proration Calculator

```php
function calculateProration(
    float $monthlyPrice,
    \DateTime $cycleStart,
    \DateTime $cycleEnd,
    \DateTime $changeDate,
    string $method = 'exact_days'
): array {
    $cycleDays = $cycleStart->diff($cycleEnd)->days;
    $daysUsed = $cycleStart->diff($changeDate)->days;
    $daysRemaining = $changeDate->diff($cycleEnd)->days;

    switch ($method) {
        case 'exact_days':
            $credit = ($monthlyPrice / $cycleDays) * $daysRemaining;
            break;

        case '30_day':
            $credit = ($monthlyPrice / 30) * $daysRemaining;
            break;

        case 'billing_date':
            $billingDay = (int)$cycleStart->format('j');
            $credit = ($monthlyPrice / $cycleDays) * $daysRemaining;
            break;
    }

    return [
        'credit' => round($credit, 2),
        'days_credited' => $daysRemaining,
        'total_cycle_days' => $cycleDays,
    ];
}

function calculateUpgradeProration(
    float $oldPrice,
    float $newPrice,
    \DateTime $cycleStart,
    \DateTime $cycleEnd,
    \DateTime $changeDate,
    string $method = 'exact_days'
): array {
    $prorationCredit = calculateProration($oldPrice, $cycleStart, $cycleEnd, $changeDate, $method);
    $cycleDays = $cycleStart->diff($cycleEnd)->days;
    $daysInNewPlan = $changeDate->diff($cycleEnd)->days;

    $newProration = ($newPrice / $cycleDays) * $daysInNewPlan;

    $charges = [
        'credit_from_old' => $prorationCredit['credit'],
        'charge_for_new' => round($newProration, 2),
        'net_amount' => round($newProration - $prorationCredit['credit'], 2),
    ];

    return $charges;
}
```

### Billing Cycle Management

```php
function createCustomCycle(int $serviceId, int $days, string $billingDay = 'same'): void {
    Capsule::table('mod_billing_custom_cycles')->insert([
        'service_id' => $serviceId,
        'days' => $days,
        'billing_day' => $billingDay,
        'created_at' => date('Y-m-d H:i:s'),
    ]);

    // Update service billing cycle
    $cycle = match ($days) {
        7 => 'Weekly',
        14 => 'Bi-Weekly',
        30 => 'Monthly',
        90 => 'Quarterly',
        180 => 'Semi-Annual',
        365 => 'Annually',
        default => 'Custom',
    };

    Capsule::table('tblhosting')
        ->where('id', $serviceId)
        ->update(['billingcycle' => $cycle]);
}

function getNextBillingDate(int $serviceId): ?\DateTime {
    $service = Capsule::table('tblhosting')->where('id', $serviceId)->first();

    $customCycle = Capsule::table('mod_billing_custom_cycles')
        ->where('service_id', $serviceId)
        ->first();

    if ($customCycle) {
        return calculateCustomNextBilling($service, $customCycle);
    }

    // Default WHMCS logic
    return new \DateTime($service->nextduedate);
}

function calculateCustomNextBilling($service, $customCycle): \DateTime {
    $currentDue = new \DateTime($service->nextduedate);
    $cycleDays = $customCycle->days;

    while ($currentDue <= new \DateTime()) {
        $currentDue->modify("+{$cycleDays} days");
    }

    return $currentDue;
}
```

### Invoice Adjustments

```php
function applyCreditToInvoice(int $serviceId, int $invoiceId, float $amount, string $notes = ''): void {
    $invoice = Capsule::table('tblinvoices')->where('id', $invoiceId)->first();

    Capsule::table('mod_billing_adjustments')->insert([
        'service_id' => $serviceId,
        'invoice_id' => $invoiceId,
        'type' => 'credit',
        'amount' => -$amount,
        'notes' => $notes,
        'created_at' => date('Y-m-d H:i:s'),
    ]);

    // Update invoice total
    Capsule::table('tblinvoices')
        ->where('id', $invoiceId)
        ->update([
            'total' => $invoice->total - $amount,
        ]);
}

function applyProrationCredit(int $serviceId, int $invoiceId, float $creditAmount, string $reason): void {
    Capsule::table('mod_billing_adjustments')->insert([
        'service_id' => $serviceId,
        'invoice_id' => $invoiceId,
        'type' => 'proration',
        'amount' => -$creditAmount,
        'notes' => $reason,
        'created_at' => date('Y-m-d H:i:s'),
    ]);

    $invoice = Capsule::table('tblinvoices')->where('id', $invoiceId)->first();

    Capsule::table('tblinvoices')
        ->where('id', $invoiceId)
        ->update([
            'total' => $invoice->total - $creditAmount,
        ]);
}
```

### Hook Integration

```php
add_hook('AfterModuleChangePackage', 1, function($vars) {
    if ($vars['action'] === 'upgrade' || $vars['action'] === 'downgrade') {
        $service = Capsule::table('tblhosting')->where('id', $vars['serviceid'])->first();
        $newProduct = Capsule::table('tblproducts')->where('id', $vars['new_product_id'])->first();

        $proration = calculateUpgradeProration(
            $service->amount,
            $newProduct->monthlycost,
            new \DateTime($service->regdate),
            new \DateTime($service->nextduedate),
            new \DateTime()
        );

        // Create prorated invoice
        if ($proration['net_amount'] != 0) {
            createProrationInvoice($vars['serviceid'], $proration);
        }
    }
});

add_hook('InvoiceCreation', 1, function($vars) {
    // Apply any pending credits to new invoice
    $credits = Capsule::table('mod_billing_adjustments')
        ->where('service_id', $vars['service_id'])
        ->where('type', 'credit')
        ->where('invoice_id', 0)
        ->where('amount', '<', 0)
        ->get();

    foreach ($credits as $credit) {
        applyCreditToInvoice($credit->service_id, $vars['invoiceid'], abs($credit->amount), 'Auto-applied credit');
    }
});
```

---

**Related Skills:**
- whmcs-addon-builder
- whmcs-cron-automation
- whmcs-invoice-module
