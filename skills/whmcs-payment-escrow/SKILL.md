# WHMCS Payment Escrow Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Build escrow payment modules for marketplace transactions.

## Escrow Module

```php
<?php
class EscrowManager {
    public function createEscrow(int $invoiceId, float $amount, string $releaseCondition): int {
        return Capsule::table('mod_escrow_payments')->insertGetId([
            'invoice_id' => $invoiceId,
            'amount' => $amount,
            'status' => 'held',
            'release_condition' => $releaseCondition,
            'held_at' => date('Y-m-d H:i:s'),
        ]);
    }

    public function releaseEscrow(int $escrowId): array {
        $escrow = Capsule::table('mod_escrow_payments')->where('id', $escrowId)->first();

        if (!$escrow || $escrow->status !== 'held') {
            return ['success' => false, 'error' => 'Escrow not available'];
        }

        Capsule::table('mod_escrow_payments')
            ->where('id', $escrowId)
            ->update([
                'status' => 'released',
                'released_at' => date('Y-m-d H:i:s'),
            ]);

        $this->disburseFunds($escrow);

        return ['success' => true];
    }

    public function disputeEscrow(int $escrowId, string $reason): array {
        Capsule::table('mod_escrow_payments')
            ->where('id', $escrowId)
            ->update([
                'status' => 'disputed',
                'dispute_reason' => $reason,
                'disputed_at' => date('Y-m-d H:i:s'),
            ]);

        return ['success' => true];
    }

    private function disburseFunds(object $escrow): void {
        $invoice = Capsule::table('tblinvoices')->where('id', $escrow->invoice_id)->first();

        $vendorShare = $escrow->amount * 0.9;
        $platformFee = $escrow->amount * 0.1;

        Capsule::table('mod_escrow_disbursements')->insert([
            'escrow_id' => $escrow->id,
            'vendor_id' => $invoice->userid,
            'amount' => $vendorShare,
            'status' => 'pending',
        ]);

        logActivity("Escrow #{$escrow->id} released: {$vendorShare} to vendor");
    }
}
```

---

**Related Skills:**
- whmcs-gateway-builder
- whmcs-payment-split