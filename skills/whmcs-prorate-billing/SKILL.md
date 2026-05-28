# WHMCS Prorated Billing Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Build prorated billing modules for mid-cycle upgrades and downgrades.

## Prorated Billing Logic

```php
<?php
class ProratedBilling {
    public function calculateProration(int $serviceId, string $newPlanCode, string $changeDate): array {
        $service = Capsule::table('tblhosting')->where('id', $serviceId)->first();
        $newPlan = Capsule::table('tblproducts')->where('id', $newPlanCode)->first();
        $oldPlan = Capsule::table('tblproducts')->where('id', $service->packageid)->first();

        $currentPeriodEnd = $service->nextduedate;
        $daysRemaining = $this->getDaysRemaining($changeDate, $currentPeriodEnd);
        $totalDays = $this->getTotalDays($changeDate, $currentPeriodEnd);

        $oldDailyRate = ($oldPlan->monthly ?? 0) / 30;
        $newDailyRate = ($newPlan->monthly ?? 0) / 30;

        $credit = $daysRemaining * $oldDailyRate;
        $charge = $daysRemaining * $newDailyRate;

        $prorationAmount = $charge - $credit;

        return [
            'credit_from_old' => round($credit, 2),
            'charge_for_new' => round($charge, 2),
            'proration_amount' => round($prorationAmount, 2),
            'days_remaining' => $daysRemaining,
            'total_days' => $totalDays,
            'direction' => $prorationAmount > 0 ? 'upgrade' : 'downgrade',
        ];
    }

    public function applyProration(int $serviceId, array $proration): int {
        $description = sprintf(
            'Proration: %d days remaining (Upgrade/Downgrade)',
            $proration['days_remaining']
        );

        if ($proration['proration_amount'] > 0) {
            return $this->createInvoiceItem($serviceId, $proration['proration_amount'], $description);
        } else {
            return $this->createCreditNote($serviceId, abs($proration['proration_amount']), $description);
        }
    }

    private function getDaysRemaining(string $from, string $to): int {
        $fromTs = strtotime($from);
        $toTs = strtotime($to);
        return max(0, ceil(($toTs - $fromTs) / 86400));
    }

    private function getTotalDays(string $from, string $to): int {
        $fromTs = strtotime($from);
        $toTs = strtotime($to);
        return ceil(($toTs - $fromTs) / 86400);
    }
}
```

---

**Related Skills:**
- whmcs-addon-builder
- whmcs-pricing-engine