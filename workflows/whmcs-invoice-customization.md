# WHMCS Invoice Customization Workflow
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Customize invoice templates and generation.

## Invoice Template Customization

### Hook-Based Customization
```php
add_hook('InvoiceCreation', 1, function($vars) {
    // Add custom line items
    Capsule::table('tblinvoiceitems')->insert([
        'invoiceid' => $vars['invoiceid'],
        'userid' => $vars['userid'],
        'description' => 'Setup Fee',
        'amount' => 0,
        'taxed' => 0,
    ]);
});
```

### Invoice Items Modification
```php
add_hook('AddInvoiceInvoiceTax', 1, function($vars) {
    // Add custom taxes
    $customTax = $vars['subtotal'] * 0.01; // 1% custom fee
    return $customTax;
});
```

## Custom Invoice PDF
```php
class CustomInvoicePDF {
    public function generate(int $invoiceId): string {
        $invoice = Capsule::table('tblinvoices')->where('id', $invoiceId)->first();
        $client = getClientsDetails($invoice->userid);

        $html = $this->getHeader($invoice, $client);
        $html .= $this->getItems($invoice->id);
        $html .= $this->getFooter($invoice);

        return $html;
    }

    private function getHeader($invoice, $client): string {
        return <<<HTML
<div class="invoice-header">
    <h1>Invoice #{$invoice->id}</h1>
    <p>Date: {$invoice->date}</p>
    <p>Due: {$invoice->duedate}</p>
</div>
<div class="client-info">
    <h3>{$client['fullname']}</h3>
    <p>{$client['address']}</p>
    <p>{$client['email']}</p>
</div>
HTML;
    }
}
```

## Invoice Notifications
```php
add_hook('InvoiceCreation', 1, function($vars) {
    // Custom notification
    $invoice = Capsule::table('tblinvoices')->where('id', $vars['invoiceid'])->first();

    if ($invoice->total > 10000000) { // > 10M VND
        // Send VIP notification
    }
});
```

---

**Related Skills:**
- whmcs-email-template-builder
- whmcs-hooks-development
