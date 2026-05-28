# WHMCS Recurring Billing Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Build advanced recurring billing management with complex cycles.

## Recurring Billing Module

```php
<?php
/**
 * Recurring Billing Module (Addon)
 * Location: modules/addons/{module}/
 */

function {module}_activate(): array {
    Capsule::schema()->create('mod_recurring_profiles', function($t) {
        $t->increments('id');
        $t->string('profile_code', 50)->unique();
        $t->integer('user_id')->unsigned();
        $t->string('billing_type', 30);
        $t->decimal('amount', 10, 2);
        $t->string('frequency', 20);
        $t->integer('interval_value')->default(1);
        $t->date('start_date');
        $t->date('end_date')->nullable();
        $t->integer('max_cycles')->unsigned()->nullable();
        $t->integer('cycles_completed')->unsigned()->default(0);
        $t->string('next_billing_date', 10);
        $t->string('status', 20)->default('active');
        $t->timestamps();

        $t->index(['user_id', 'status']);
        $t->index('next_billing_date');
    });

    Capsule::schema()->create('mod_recurring_payments', function($t) {
        $t->increments('id');
        $t->integer('profile_id')->unsigned();
        $t->integer('invoice_id')->unsigned()->nullable();
        $t->decimal('amount', 10, 2);
        $t->string('status', 20);
        $t->timestamp('processed_at')->nullable();
        $t->timestamps();

        $t->index('profile_id');
    });

    return ['status' => 'success'];
}

function {module}_deactivate(): array {
    Capsule::schema()->dropIfExists('mod_recurring_profiles');
    Capsule::schema()->dropIfExists('mod_recurring_payments');
    return ['status' => 'success'];
}
```

## Billing Logic

```php
<?php
class RecurringBilling {
    public function processRecurring(): array {
        $profiles = Capsule::table('mod_recurring_profiles')
            ->where('status', 'active')
            ->where('next_billing_date', '<=', date('Y-m-d'))
            ->get();

        $results = ['processed' => 0, 'failed' => 0, 'completed' => 0];

        foreach ($profiles as $profile) {
            try {
                $result = $this->processProfile($profile);

                if ($result['success']) {
                    $results['processed']++;
                } else {
                    $results['failed']++;
                }

                if ($result['completed']) {
                    $results['completed']++;
                }
            } catch (\Exception $e) {
                $this->logError($profile->id, $e->getMessage());
                $results['failed']++;
            }
        }

        return $results;
    }

    private function processProfile(object $profile): array {
        $nextDate = $this->calculateNextDate($profile->next_billing_date, $profile->frequency, $profile->interval_value);

        $invoiceId = $this->createInvoice($profile);

        $paymentResult = $this->processPayment($profile, $invoiceId);

        $this->recordPayment($profile->id, $invoiceId, $profile->amount, $paymentResult['success'] ? 'paid' : 'failed');

        $completedCycles = $profile->cycles_completed + 1;
        $isComplete = $profile->max_cycles && $completedCycles >= $profile->max_cycles;

        Capsule::table('mod_recurring_profiles')
            ->where('id', $profile->id)
            ->update([
                'next_billing_date' => $nextDate,
                'cycles_completed' => $completedCycles,
                'status' => $isComplete ? 'completed' : 'active',
            ]);

        return ['success' => $paymentResult['success'], 'completed' => $isComplete];
    }

    private function calculateNextDate(string $current, string $frequency, int $interval): string {
        $format = 'Y-m-d';

        return match ($frequency) {
            'daily' => date($format, strtotime($current . " +{$interval} days")),
            'weekly' => date($format, strtotime($current . " +{$interval} weeks")),
            'monthly' => date($format, strtotime($current . " +{$interval} months")),
            'quarterly' => date($format, strtotime($current . " +" . (3 * $interval) . " months")),
            'semi_annually' => date($format, strtotime($current . " +" . (6 * $interval) . " months")),
            'annually' => date($format, strtotime($current . " +{$interval} years")),
            default => date($format, strtotime($current . " +{$interval} months")),
        };
    }
}
```

---

**Related Skills:**
- whmcs-addon-builder
- whmcs-subscription-billing
- whmcs-cron-automation