# WHMCS Invoice Customization Workflow
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Comprehensive guide to customizing WHMCS invoice templates, generation logic, PDF outputs, and automation rules for professional billing presentation.

## Prerequisites

- WHMCS installation with template access
- Admin access to template files
- Understanding of HTML/CSS for templating
- PHP knowledge for hook-based customization

## Workflow Steps

### Step 1: Locate and Understand Invoice Templates

Identify the invoice template files:

```php
// WHMCS invoice templates location
$templatePath = ROOTDIR . '/templates/{template_name}/views/invoice-';

// Key template files:
$templates = [
    'invoice-view'       => ROOTDIR . '/templates/{name}/views/invoice--viewinvoice.tpl',
    'invoice-print'      => ROOTDIR . '/templates/{name}/views/invoice-printable.tpl',
    'invoice-pdf-header' => ROOTDIR . '/templates/{name}/invoicepdf/header.tpl',
    'invoice-pdf-footer' => ROOTDIR . '/templates/{name}/invoicepdf/footer.tpl',
    'invoice-pdf-table'  => ROOTDIR . '/templates/{name}/invoicepdf/table.tpl',
];
```

Access invoice data in templates:

```smarty
{* Available invoice variables *}
{$invoice.id}
{$invoice.invoinenumber}
{$invoice.date}
{$invoice.duedate}
{$invoice.subtotal}
{$invoice.tax}
{$invoice.total}
{$invoice.balance}
{$invoice.status}
{$invoice.paymentmethod}

{* Client information *}
{$client.firstname}
{$client.lastname}
{$client.companyname}
{$client.email}
{$client.address1}
{$client.city}
{$client.state}
{$client.postcode}
{$client.country}

{* Line items *}
{foreach from=$invoiceitems item=row}
    {$row.description}
    {$row.amount}
    {$row.taxed}
{/foreach}
```

### Step 2: Customize Invoice Template Structure

Create custom invoice header:

```smarty
{* templates/{name}/invoicepdf/header.tpl *}

<div class="invoice-header">
    <div class="company-logo">
        <img src="{$baseurl}/templates/{$_template}/assets/img/logo.png"
             alt="{$companyname}" height="60">
    </div>

    <div class="invoice-meta">
        <h1 class="invoice-title">INVOICE</h1>
        <p class="invoice-number">{$invoice.invoicenumber}</p>
        <table class="invoice-details">
            <tr>
                <td><strong>Invoice Date:</strong></td>
                <td>{$invoice.date|date_format:"%B %d, %Y"}</td>
            </tr>
            <tr>
                <td><strong>Due Date:</strong></td>
                <td>{$invoice.duedate|date_format:"%B %d, %Y"}</td>
            </tr>
            <tr>
                <td><strong>Status:</strong></td>
                <td class="status-{$invoice.status|lower}">{$invoice.status}</td>
            </tr>
        </table>
    </div>

    <div class="payment-summary">
        <table class="amount-due">
            <tr>
                <td>Total Due:</td>
                <td class="amount">{$invoice.total|currency_format}</td>
            </tr>
            {if $invoice.balance > 0}
            <tr>
                <td>Balance Due:</td>
                <td class="balance">{$invoice.balance|currency_format}</td>
            </tr>
            {/if}
        </table>
    </div>
</div>

<div class="billing-address">
    <h3>Bill To:</h3>
    <p>
        {if $client.companyname}{$client.companyname}<br>{/if}
        {$client.firstname} {$client.lastname}<br>
        {$client.address1}{if $client.address2}<br>{$client.address2}{/if}<br>
        {$client.city}, {$client.state} {$client.postcode}<br>
        {$client.country}
    </p>
</div>
```

### Step 3: Implement Hook-Based Invoice Modifications

Add custom invoice line items via hooks:

```php
// includes/hooks/custom_invoice_items.php

use WHMCS\Database\Capsule;
use WHMCS\Billing\Invoice;

/**
 * Add setup fee to new service invoices
 */
add_hook('InvoiceCreation', 1, function(array $vars) {
    $invoiceId = $vars['invoiceid'];
    $serviceId = $vars['serviceid'] ?? null;

    if (!$serviceId) {
        return;
    }

    $service = Capsule::table('tblhosting')
        ->where('id', $serviceId)
        ->first();

    if (!$service || $service->domainstatus !== 'Active') {
        return;
    }

    // Check if setup fee already exists
    $existingFee = Capsule::table('tblinvoiceitems')
        ->where('invoiceid', $invoiceId)
        ->where('description', 'like', '%Setup Fee%')
        ->first();

    if ($existingFee) {
        return;
    }

    // Add one-time setup fee
    $setupFee = $this->calculateSetupFee($service);

    if ($setupFee > 0) {
        Capsule::table('tblinvoiceitems')->insert([
            'invoiceid'   => $invoiceId,
            'userid'      => $service->userid,
            'description' => 'Service Setup Fee - ' . $service->domain,
            'amount'      => $setupFee,
            'taxed'       => 1,
        ]);
    }
});

/**
 * Add late fee for overdue invoices
 */
add_hook('InvoiceOverdue', 1, function(array $vars) {
    $invoiceId = $vars['invoiceid'];
    $invoice = Capsule::table('tblinvoices')->where('id', $invoiceId)->first();

    // Only apply to invoices overdue by more than 15 days
    $dueDate = strtotime($invoice->duedate);
    $daysOverdue = (time() - $dueDate) / 86400;

    if ($daysOverdue >= 15) {
        $lateFee = $invoice->total * 0.02; // 2% late fee

        // Check if late fee already applied
        $existingFee = Capsule::table('tblinvoiceitems')
            ->where('invoiceid', $invoiceId)
            ->where('description', 'like', '%Late Fee%')
            ->first();

        if (!$existingFee) {
            Capsule::table('tblinvoiceitems')->insert([
                'invoiceid'   => $invoiceId,
                'userid'      => $invoice->userid,
                'description' => 'Late Payment Fee (2%)',
                'amount'      => $lateFee,
                'taxed'       => 1,
            ]);

            // Recalculate invoice totals
            recalcInvoiceTotal($invoiceId);
        }
    }
});

/**
 * Add volume discount based on subtotal
 */
add_hook('InvoiceCreation', 2, function(array $vars) {
    $invoiceId = $vars['invoiceid'];
    $invoice = Capsule::table('tblinvoices')->where('id', $invoiceId)->first();

    $discountTiers = [
        ['min' => 5000000, 'discount' => 0.03],  // 3% for orders >= 5M
        ['min' => 10000000, 'discount' => 0.05], // 5% for orders >= 10M
        ['min' => 25000000, 'discount' => 0.08], // 8% for orders >= 25M
    ];

    foreach ($discountTiers as $tier) {
        if ($invoice->subtotal >= $tier['min']) {
            // Check if discount already applied
            $existingDiscount = Capsule::table('tblinvoiceitems')
                ->where('invoiceid', $invoiceId)
                ->where('description', 'like', '%Volume Discount%')
                ->first();

            if (!$existingDiscount) {
                $discountAmount = $invoice->subtotal * $tier['discount'];
                $tierLabel = ($tier['discount'] * 100) . '%';

                Capsule::table('tblinvoiceitems')->insert([
                    'invoiceid'   => $invoiceId,
                    'userid'      => $invoice->userid,
                    'description' => "Volume Discount ({$tierLabel})",
                    'amount'      => -$discountAmount,
                    'taxed'       => 0,
                ]);

                recalcInvoiceTotal($invoiceId);
            }
            break;
        }
    }
});
```

### Step 4: Create Custom PDF Invoice Generator

Implement custom PDF generation:

```php
// includes/classes/CustomInvoicePDF.php

use Dompdf\Dompdf;
use Dompdf\Options;

class CustomInvoicePDF
{
    private Dompdf $pdf;
    private array $invoiceData;
    private string $templatePath;

    public function __construct()
    {
        $options = new Options();
        $options->set('isHtml5ParserEnabled', true);
        $options->set('isRemoteEnabled', true);

        $this->pdf = new Dompdf($options);
        $this->templatePath = ROOTDIR . '/templates/' . $GLOBALS['templates']->getActiveTemplate();
    }

    public function generate(int $invoiceId): string
    {
        $this->loadInvoiceData($invoiceId);
        $html = $this->buildHTML();
        $this->pdf->loadHtml($html);
        $this->pdf->setPaper('A4', 'portrait');
        $this->pdf->render();

        return $this->pdf->output();
    }

    private function loadInvoiceData(int $invoiceId): void
    {
        $invoice = Capsule::table('tblinvoices')->where('id', $invoiceId)->first();
        $client = Capsule::table('tblclients')->where('id', $invoice->userid)->first();
        $items = Capsule::table('tblinvoiceitems')->where('invoiceid', $invoiceId)->get();

        $this->invoiceData = [
            'invoice'   => $invoice,
            'client'    => $client,
            'items'     => $items,
            'company'   => $this->getCompanyDetails(),
        ];
    }

    private function buildHTML(): string
    {
        $inv = $this->invoiceData;
        $client = $inv['client'];

        $html = <<<HTML
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <style>
        body { font-family: 'DejaVu Sans', sans-serif; font-size: 12px; margin: 40px; }
        .header { display: flex; justify-content: space-between; margin-bottom: 30px; }
        .company-info h1 { margin: 0; color: #2c3e50; }
        .invoice-info { text-align: right; }
        .invoice-info h2 { margin: 0; color: #e74c3c; }
        .billing-section { margin-bottom: 30px; }
        .billing-section h3 { color: #7f8c8d; border-bottom: 1px solid #ddd; padding-bottom: 5px; }
        table { width: 100%; border-collapse: collapse; margin-bottom: 20px; }
        th { background: #3498db; color: white; padding: 10px; text-align: left; }
        td { padding: 10px; border-bottom: 1px solid #ecf0f1; }
        .text-right { text-align: right; }
        .text-right { text-align: right; }
        .text-center { text-align: center; }
        .total-row td { font-weight: bold; background: #ecf0f1; }
        .payment-section { background: #f8f9fa; padding: 20px; border-radius: 5px; margin-top: 30px; }
        .payment-methods { display: flex; gap: 20px; }
        .footer { margin-top: 50px; text-align: center; color: #7f8c8d; font-size: 10px; }
    </style>
</head>
<body>
    <div class="header">
        <div class="company-info">
            <h1>{$inv['company']['name']}</h1>
            <p>{$inv['company']['address']}</p>
            <p>{$inv['company']['phone']}</p>
            <p>{$inv['company']['email']}</p>
        </div>
        <div class="invoice-info">
            <h2>INVOICE</h2>
            <p><strong>#{$inv['invoice']->id}</strong></p>
            <p>Date: {$inv['invoice']->date}</p>
            <p>Due: {$inv['invoice']->duedate}</p>
            <p>Status: <span style="color: {$this->getStatusColor($inv['invoice']->status)}">{$inv['invoice']->status}</span></p>
        </div>
    </div>

    <div class="billing-section">
        <h3>Bill To:</h3>
        <p>
            {if \$client->companyname}{\$client->companyname}<br>{/if}
            {\$client->firstname} {\$client->lastname}<br>
            {\$client->address1}{if \$client->address2}<br>{\$client->address2}{/if}<br>
            {\$client->city}, {\$client->state} {\$client->postcode}<br>
            {\$client->country}
        </p>
    </div>

    <table class="items-table">
        <thead>
            <tr>
                <th style="width: 60%;">Description</th>
                <th style="width: 20%;" class="text-center">Qty</th>
                <th style="width: 20%;" class="text-right">Amount</th>
            </tr>
        </thead>
        <tbody>
HTML;

        foreach ($inv['items'] as $item) {
            $html .= <<<HTML
            <tr>
                <td>{$item->description}</td>
                <td class="text-center">{$item->qty ?? 1}</td>
                <td class="text-right">{$this->formatCurrency($item->amount)}</td>
            </tr>
HTML;
        }

        $html .= <<<HTML
        </tbody>
    </table>

    <table class="totals-table">
        <tr>
            <td class="text-right">Subtotal:</td>
            <td class="text-right" style="width: 120px;">{$this->formatCurrency($inv['invoice']->subtotal)}</td>
        </tr>
HTML;

        if ($inv['invoice']->tax > 0) {
            $html .= <<<HTML
        <tr>
            <td class="text-right">Tax:</td>
            <td class="text-right">{$this->formatCurrency($inv['invoice']->tax)}</td>
        </tr>
HTML;
        }

        $html .= <<<HTML
        <tr class="total-row">
            <td class="text-right">Total:</td>
            <td class="text-right">{$this->formatCurrency($inv['invoice']->total)}</td>
        </tr>
HTML;

        if ($inv['invoice']->balance > 0) {
            $html .= <<<HTML
        <tr>
            <td class="text-right">Balance Due:</td>
            <td class="text-right" style="color: #e74c3c; font-size: 14px;">{$this->formatCurrency($inv['invoice']->balance)}</td>
        </tr>
HTML;
        }

        $html .= <<<HTML
    </table>

    <div class="payment-section">
        <h3>Payment Information</h3>
        <p>Please pay before the due date to avoid service interruption.</p>
        <div class="payment-methods">
            <div>
                <strong>Bank Transfer:</strong><br>
                Bank: {$inv['company']['bank_name']}<br>
                Account: {$inv['company']['bank_account']}<br>
                Account Name: {$inv['company']['bank_account_name']}
            </div>
        </div>
    </div>

    <div class="footer">
        <p>Thank you for your business!</p>
        <p>{$inv['company']['name']} | {$inv['company']['website']}</p>
    </div>
</body>
</html>
HTML;

        return $html;
    }

    private function getCompanyDetails(): array
    {
        return [
            'name'            => Capsule::table('tblconfiguration')->where('setting', 'CompanyName')->first()->value,
            'address'         => Capsule::table('tblconfiguration')->where('setting', 'InvoiceTerms')->first()->value ?? '',
            'phone'           => Capsule::table('tblconfiguration')->where('setting', 'PhoneNumber')->first()->value ?? '',
            'email'           => Capsule::table('tblconfiguration')->where('setting', 'EmailAddress')->first()->value ?? '',
            'bank_name'       => Capsule::table('tblconfiguration')->where('setting', 'BankName')->first()->value ?? '',
            'bank_account'    => Capsule::table('tblconfiguration')->where('setting', 'BankAccount')->first()->value ?? '',
            'bank_account_name' => Capsule::table('tblconfiguration')->where('setting', 'BankAccountName')->first()->value ?? '',
            'website'         => Capsule::table('tblconfiguration')->where('setting', 'DomainURL')->first()->value ?? '',
        ];
    }

    private function formatCurrency(float $amount): string
    {
        return number_format($amount, 0, ',', '.') . ' VND';
    }

    private function getStatusColor(string $status): string
    {
        $colors = [
            'Paid'      => '#27ae60',
            'Unpaid'    => '#e74c3c',
            'Overdue'   => '#c0392b',
            'Cancelled' => '#95a5a6',
            'Draft'     => '#3498db',
        ];

        return $colors[$status] ?? '#333';
    }
}
```

### Step 5: Implement Invoice Automations

Set up automated invoice handling:

```php
// includes/hooks/invoice_automation.php

/**
 * Auto-charge credit on invoice due date
 */
add_hook('InvoiceDue', 1, function(array $vars) {
    $invoiceId = $vars['invoiceid'];
    $invoice = Capsule::table('tblinvoices')->where('id', $invoiceId)->first();

    // Check if client has credit on file
    $creditBalance = Capsule::table('tblcredit')
        ->where('clientid', $invoice->userid)
        ->sum('amount');

    if ($creditBalance > 0 && $creditBalance >= $invoice->total) {
        // Apply credit to invoice
        addInvoicePayment($invoiceId, 0, $creditBalance, 0, 'credit');

        Capsule::table('tblcredit')->insert([
            'clientid'    => $invoice->userid,
            'date'        => date('Y-m-d H:i:s'),
            'description' => "Auto-applied credit to invoice #{$invoiceId}",
            'amount'      => -$invoice->total,
        ]);

        logActivity("Auto-applied credit of {$creditBalance} to invoice #{$invoiceId}");
    }
});

/**
 * Send reminder based on days overdue
 */
add_hook('DailyCronJob', 1, function(array $vars) {
    $overdueInvoices = Capsule::table('tblinvoices')
        ->where('status', 'Overdue')
        ->where('duedate', '<', date('Y-m-d'))
        ->get();

    foreach ($overdueInvoices as $invoice) {
        $daysOverdue = (strtotime(date('Y-m-d')) - strtotime($invoice->duedate)) / 86400;

        // Send escalating reminders
        $reminderTemplates = [
            7  => 'invoice-reminder-1',
            14 => 'invoice-reminder-2',
            21 => 'invoice-reminder-3',
            30 => 'invoice-final-warning',
        ];

        if (isset($reminderTemplates[$daysOverdue])) {
            sendEmail($reminderTemplates[$daysOverdue], $invoice->id);
        }
    }
});

/**
 * Auto-void old unpaid invoices
 */
add_hook('DailyCronJob', 2, function(array $vars) {
    $oldUnpaidInvoices = Capsule::table('tblinvoices')
        ->where('status', 'Unpaid')
        ->where('date', '<', date('Y-m-d', strtotime('-60 days')))
        ->get();

    foreach ($oldUnpaidInvoices as $invoice) {
        // Don't void invoices with pending payments
        if ($invoice->paymentmethod && hasPendingPayment($invoice->id)) {
            continue;
        }

        // Void the invoice
        Capsule::table('tblinvoices')
            ->where('id', $invoice->id)
            ->update(['status' => 'Cancelled']);

        logActivity("Auto-voided invoice #{$invoice->id} (60+ days unpaid)");

        // Send notification
        sendEmail('invoice-void-notice', $invoice->id);
    }
});
```

---

## Best Practices

1. **Maintain consistent branding** - Use company colors and fonts across all invoices
2. **Include clear payment instructions** - Make it easy for clients to pay
3. **Use automation wisely** - Test hooks thoroughly before deployment
4. **Preserve audit trail** - Log all manual invoice modifications
5. **Support multiple currencies** - Display amounts in client's preferred currency
6. **Mobile-friendly design** - Ensure PDF is readable on all devices
7. **Personalize greetings** - Use client's preferred name/company
8. **Document template changes** - Track all modifications for compliance
9. **Test edge cases** - Verify with various invoice scenarios
10. **Backup original templates** - Keep originals before major changes

---

## Verification Checklist

- [ ] Invoice template displays company logo correctly
- [ ] All client information populated accurately
- [ ] Line items formatted consistently
- [ ] Tax calculations correct
- [ ] PDF generation produces valid document
- [ ] Payment methods clearly displayed
- [ ] Custom line items added via hooks working
- [ ] Late fees applied correctly
- [ ] Invoice automations functioning (reminders, auto-void)
- [ ] Invoice emails sent with correct formatting
- [ ] Mobile/PDF viewing tested on multiple devices
