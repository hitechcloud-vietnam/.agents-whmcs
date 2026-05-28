# WHMCS Revenue Recognition Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Build revenue recognition modules for accounting compliance.

## Revenue Recognition

```php
<?php
class RevenueRecognition {
    public function recognizeRevenue(int $invoiceId, string $method = 'immediate'): array {
        $invoice = Capsule::table('tblinvoices')->where('id', $invoiceId)->first();

        if ($invoice->status !== 'Paid') {
            return ['success' => false, 'error' => 'Invoice not paid'];
        }

        $items = Capsule::table('tblinvoiceitems')
            ->where('invoiceid', $invoiceId)
            ->get();

        foreach ($items as $item) {
            $this->recognizeItemRevenue($invoiceId, $item, $invoice->datepaid, $method);
        }

        return ['success' => true, 'recognized_at' => date('Y-m-d H:i:s')];
    }

    private function recognizeItemRevenue($invoiceId, $item, string $paidDate, string $method): void {
        $amount = $item->amount;

        switch ($method) {
            case 'immediate':
                $this->recordRevenue($invoiceId, $item->id, $amount, $paidDate);
                break;

            case 'ratable':
                $this->createRatableRevenue($invoiceId, $item, $amount, $paidDate);
                break;

            case 'milestone':
                $this->createMilestoneRevenue($invoiceId, $item, $amount, $paidDate);
                break;
        }
    }

    private function createRatableRevenue($invoiceId, $item, float $amount, string $startDate): void {
        $periodMonths = 12;
        $monthlyAmount = $amount / $periodMonths;

        for ($i = 0; $i < $periodMonths; $i++) {
            $recognizeDate = date('Y-m-d', strtotime($startDate . " +{$i} months"));

            Capsule::table('mod_revenue_recognition')->insert([
                'invoice_id' => $invoiceId,
                'item_id' => $item->id,
                'amount' => $monthlyAmount,
                'recognize_date' => $recognizeDate,
                'status' => 'pending',
            ]);
        }
    }
}
```

---

**Related Skills:**
- whmcs-addon-builder
- whmcs-reporting