# WHMCS Invoice Generator Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Build custom invoice generation with dynamic templates.

## Invoice Generator

```php
<?php
class InvoiceGenerator {
    public function createCustomInvoice(int $userId, array $items, array $options = []): int {
        $invoiceId = localAPI('CreateInvoice', [
            'userid' => $userId,
            'date' => date('Y-m-d'),
            'duedate' => $options['due_date'] ?? date('Y-m-d', strtotime('+14 days')),
            'itemdescription' => array_column($items, 'description'),
            'itemamount' => array_column($items, 'amount'),
            'itemtaxed' => array_column($items, 'taxed', false),
        ]);

        if (isset($options['notes'])) {
            Capsule::table('tblinvoices')
                ->where('id', $invoiceId)
                ->update(['notes' => $options['notes']]);
        }

        return $invoiceId;
    }

    public function generateProforma(int $userId, array $items): int {
        $invoiceId = $this->createCustomInvoice($userId, $items, [
            'due_date' => date('Y-m-d', strtotime('+30 days')),
            'notes' => 'This is a proforma invoice. Payment is not required at this time.',
        ]);

        Capsule::table('tblinvoices')
            ->where('id', $invoiceId)
            ->update(['status' => 'Draft', 'type' => 'proforma']);

        return $invoiceId;
    }

    public function createCreditNote(int $userId, float $amount, string $reason): int {
        return localAPI('CreateInvoice', [
            'userid' => $userId,
            'date' => date('Y-m-d'),
            'itemdescription' => ["Credit Note: {$reason}"],
            'itemamount' => [-$amount],
            'itemtaxed' => [false],
        ]);
    }
}
```

## PDF Customization

```php
<?php
class CustomInvoicePDF {
    public function generate(int $invoiceId): string {
        $invoice = Capsule::table('tblinvoices')->where('id', $invoiceId)->first();
        $client = getClientsDetails($invoice->userid);
        $items = $this->getInvoiceItems($invoiceId);

        $html = $this->getHeader($invoice, $client);
        $html .= $this->getItemsTable($items);
        $html .= $this->getTotals($invoice);
        $html .= $this->getFooter($invoice);

        return $this->convertToPDF($html);
    }

    private function getHeader($invoice, $client): string {
        $company = Capsule::table('tblconfiguration')
            ->where('setting', 'CompanyName')
            ->value('value');

        return <<<HTML
<div class="invoice-header">
    <div class="company-info">
        <h1>{$company}</h1>
    </div>
    <div class="invoice-info">
        <h2>Invoice #{$invoice->id}</h2>
        <p>Date: {$invoice->date}</p>
        <p>Due: {$invoice->duedate}</p>
    </div>
</div>
<div class="client-info">
    <h3>Bill To:</h3>
    <p>{$client['fullname']}</p>
    <p>{$client['address1']}</p>
    <p>{$client['city']}, {$client['state']} {$client['postcode']}</p>
</div>
HTML;
    }
}
```

---

**Related Skills:**
- whmcs-addon-builder
- whmcs-invoice-customization