# WHMCS Quote System Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Build quote/estimate management modules.

## Quote Module

```php
<?php
class QuoteManager {
    public function createQuote(int $userId, array $items, array $options = []): int {
        $quoteNumber = 'QUO-' . date('Ymd') . '-' . strtoupper(substr(md5(uniqid()), 0, 6));

        $quoteId = Capsule::table('mod_quotes')->insertGetId([
            'quote_number' => $quoteNumber,
            'user_id' => $userId,
            'subject' => $options['subject'] ?? 'Quote',
            'valid_until' => $options['valid_until'] ?? date('Y-m-d', strtotime('+30 days')),
            'status' => 'draft',
            'notes' => $options['notes'] ?? '',
            'created_at' => date('Y-m-d H:i:s'),
        ]);

        foreach ($items as $item) {
            Capsule::table('mod_quote_items')->insert([
                'quote_id' => $quoteId,
                'description' => $item['description'],
                'quantity' => $item['quantity'] ?? 1,
                'unit_price' => $item['unit_price'],
                'discount' => $item['discount'] ?? 0,
            ]);
        }

        return $quoteId;
    }

    public function convertToInvoice(int $quoteId): int {
        $quote = Capsule::table('mod_quotes')->where('id', $quoteId)->first();

        if (!$quote || $quote->status !== 'accepted') {
            throw new \Exception('Quote must be accepted before conversion');
        }

        $items = Capsule::table('mod_quote_items')
            ->where('quote_id', $quoteId)
            ->get();

        $invoiceItems = [];
        foreach ($items as $item) {
            $amount = $item->unit_price * $item->quantity * (1 - $item->discount / 100);
            $invoiceItems[] = [
                'description' => $item->description,
                'amount' => $amount,
                'taxed' => false,
            ];
        }

        $invoiceId = localAPI('CreateInvoice', [
            'userid' => $quote->user_id,
            'date' => date('Y-m-d'),
            'itemdescription' => array_column($invoiceItems, 'description'),
            'itemamount' => array_column($invoiceItems, 'amount'),
        ]);

        Capsule::table('mod_quotes')
            ->where('id', $quoteId)
            ->update([
                'status' => 'converted',
                'converted_to_invoice' => $invoiceId,
            ]);

        return $invoiceId;
    }
}
```

---

**Related Skills:**
- whmcs-addon-builder
- whmcs-invoice-generator