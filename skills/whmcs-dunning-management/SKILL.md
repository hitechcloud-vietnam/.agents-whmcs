# WHMCS Dunning Management Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Build dunning modules for payment recovery.

## Dunning Module

```php
<?php
class DunningManager {
    private array $dunningPlan = [
        'day_1' => ['action' => 'email', 'template' => 'InvoiceReminder1', 'fee' => 0],
        'day_3' => ['action' => 'email', 'template' => 'InvoiceReminder2', 'fee' => 0],
        'day_7' => ['action' => 'email', 'template' => 'InvoiceReminder3', 'fee' => 5],
        'day_14' => ['action' => 'suspend', 'template' => 'ServiceSuspended', 'fee' => 10],
        'day_30' => ['action' => 'terminate', 'template' => 'ServiceTerminated', 'fee' => 25],
    ];

    public function processDunning(): array {
        $overdueInvoices = $this->getOverdueInvoices();
        $results = ['emails' => 0, 'suspensions' => 0, 'terminations' => 0, 'fees' => 0];

        foreach ($overdueInvoices as $invoice) {
            $stage = $this->determineDunningStage($invoice);
            $result = $this->executeStage($invoice, $stage);
            $results[$result['type']]++;
            $results['fees'] += $result['fee_added'] ?? 0;
        }

        return $results;
    }

    private function executeStage($invoice, int $daysOverdue): array {
        $stageKey = $this->getStageKey($daysOverdue);

        if (!$stageKey || !isset($this->dunningPlan[$stageKey])) {
            return ['type' => 'none', 'fee_added' => 0];
        }

        $stage = $this->dunningPlan[$stageKey];

        switch ($stage['action']) {
            case 'email':
                $this->sendReminder($invoice, $stage['template']);
                $this->logDunningAction($invoice->id, $stageKey, 'email_sent');
                break;

            case 'suspend':
                $this->addLateFee($invoice, $stage['fee']);
                $this->suspendServices($invoice->userid);
                $this->logDunningAction($invoice->id, $stageKey, 'suspended');
                break;

            case 'terminate':
                $this->addLateFee($invoice, $stage['fee']);
                $this->terminateServices($invoice->userid);
                $this->logDunningAction($invoice->id, $stageKey, 'terminated');
                break;
        }

        return ['type' => $stage['action'] . 's', 'fee_added' => $stage['fee']];
    }

    private function addLateFee($invoice, float $fee): void {
        if ($fee > 0) {
            Capsule::table('tblinvoiceitems')->insert([
                'invoiceid' => $invoice->id,
                'userid' => $invoice->userid,
                'description' => 'Late Payment Fee',
                'amount' => $fee,
                'taxed' => 0,
            ]);

            $this->recalculateInvoiceTotal($invoice->id);
        }
    }
}
```

---

**Related Skills:**
- whmcs-addon-builder
- whmcs-overdue-handler