# WHMCS Auto Bill Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Build automatic billing modules with flexible scheduling.

## Auto Bill Module

```php
<?php
class AutoBillingScheduler {
    public function scheduleBilling(int $userId, array $config): int {
        return Capsule::table('mod_autobill_schedules')->insertGetId([
            'user_id' => $userId,
            'payment_method' => $config['payment_method'],
            'billing_day' => $config['billing_day'] ?? 1,
            'max_amount' => $config['max_amount'] ?? null,
            'notification_days' => $config['notification_days'] ?? 3,
            'auto_pay' => $config['auto_pay'] ?? true,
            'status' => 'active',
            'created_at' => date('Y-m-d H:i:s'),
        ]);
    }

    public function processAutoBilling(): array {
        $schedules = $this->getDueSchedules();
        $results = ['processed' => 0, 'paid' => 0, 'failed' => 0, 'skipped' => 0];

        foreach ($schedules as $schedule) {
            $invoices = $this->getUnpaidInvoices($schedule->user_id);

            if (empty($invoices)) {
                $results['skipped']++;
                continue;
            }

            $total = array_sum(array_column($invoices, 'total'));

            if ($schedule->max_amount && $total > $schedule->max_amount) {
                $this->sendHighAmountNotification($schedule, $total);
                $results['skipped']++;
                continue;
            }

            if (!$schedule->auto_pay) {
                $this->sendPaymentReminder($schedule, $total);
                $results['processed']++;
                continue;
            }

            $result = $this->processPayment($schedule, $invoices);

            if ($result['success']) {
                $results['paid']++;
            } else {
                $results['failed']++;
                $this->handleFailedPayment($schedule, $result['error']);
            }
        }

        return $results;
    }
}
```

---

**Related Skills:**
- whmcs-addon-builder
- whmcs-recurring-billing