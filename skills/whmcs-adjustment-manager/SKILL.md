# WHMCS Adjustment Manager Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Build invoice adjustment modules for credits and modifications.

## Adjustment Module

```php
<?php
class AdjustmentManager {
    public function addLineItem(int $invoiceId, array $item): array {
        $invoice = Capsule::table('tblinvoices')->where('id', $invoiceId)->first();

        if ($invoice->status !== 'Draft' && $invoice->status !== 'Unpaid') {
            return ['success' => false, 'error' => 'Cannot modify this invoice'];
        }

        Capsule::table('tblinvoiceitems')->insert([
            'invoiceid' => $invoiceId,
            'userid' => $invoice->userid,
            'description' => $item['description'],
            'amount' => $item['amount'],
            'taxed' => $item['taxed'] ?? false,
        ]);

        $this->recalculateTotal($invoiceId);

        return ['success' => true];
    }

    public function removeLineItem(int $invoiceId, int $itemId): array {
        $item = Capsule::table('tblinvoiceitems')
            ->where('id', $itemId)
            ->where('invoiceid', $invoiceId)
            ->first();

        if (!$item) {
            return ['success' => false, 'error' => 'Item not found'];
        }

        Capsule::table('tblinvoiceitems')->where('id', $itemId)->delete();

        $this->recalculateTotal($invoiceId);

        return ['success' => true];
    }

    public function applyCredit(int $invoiceId, float $amount, string $description = 'Credit Applied'): array {
        return $this->addLineItem($invoiceId, [
            'description' => $description,
            'amount' => -abs($amount),
            'taxed' => false,
        ]);
    }

    public function applyDiscount(int $invoiceId, float $percentage, string $description = 'Discount'): array {
        $subtotal = Capsule::table('tblinvoices')->where('id', $invoiceId)->value('subtotal');
        $discountAmount = $subtotal * ($percentage / 100);

        return $this->addLineItem($invoiceId, [
            'description' => "{$description} ({$percentage}%)",
            'amount' => -abs($discountAmount),
            'taxed' => false,
        ]);
    }

    private function recalculateTotal(int $invoiceId): void {
        $items = Capsule::table('tblinvoiceitems')
            ->where('invoiceid', $invoiceId)
            ->get();

        $subtotal = 0;
        $tax = 0;

        foreach ($items as $item) {
            if ($item->taxed) {
                $taxRate = Capsule::table('tbltax')->where('country', '')->value('taxrate') ?? 0;
                $tax += $item->amount * ($taxRate / 100);
            }
            $subtotal += $item->amount;
        }

        $total = $subtotal + $tax;

        Capsule::table('tblinvoices')->where('id', $invoiceId)->update([
            'subtotal' => $subtotal,
            'tax' => $tax,
            'total' => $total,
        ]);
    }
}
```

---

**Related Skills:**
- whmcs-addon-builder
- whmcs-invoice-customization