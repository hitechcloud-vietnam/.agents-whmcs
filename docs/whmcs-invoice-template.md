# WHMCS Invoice Template Structure

## Overview

Invoice templates in WHMCS control the appearance of printable/downloadable invoices. They can be customized to match your brand identity.

## Template Structure

### Directory Location

```
templates/invoice-{template_name}/
    index.php
    stylesheet.css
    header.tpl
    footer.tpl
    ...
```

### Default Template

```
templates/invoice-default/
```

## Invoice Variables

### Available Variables

```smarty
{$invoice->id}           {* Invoice ID *}
{$invoice->invoicenum}   {* Invoice number *}
{$invoice->userid}       {* Client ID *}
{$invoice->date}        {* Invoice date *}
{$invoice->duedate}     {* Due date *}
{$invoice->datepaid}    {* Payment date *}
{$invoice->subtotal}     {* Subtotal *}
{$invoice->tax}         {* Tax amount *}
{$invoice->tax2}        {* Secondary tax *}
{$invoice->total}       {* Total *}
{$invoice->credit}      {* Credit applied *}
{$invoice->amountpaid}  {* Amount paid *}
{$invoice->balance}     {* Balance due *}
{$invoice->status}      {* Status *}
{$invoice->notes}       {* Invoice notes *}
{$invoice->paymentmethod} {* Payment method *}
{$invoice->items}       {* Line items *}
```

### Client Variables

```smarty
{$client->id}
{$client->firstname}
{$client->lastname}
{$client->companyname}
{$client->email}
{$client->address1}
{$client->address2}
{$client->city}
{$client->state}
{$client->postcode}
{$client->country}
```

### Line Item Variables

```smarty
{foreach $invoice->items as $item}
    {$item->id}           {* Line item ID *}
    {$item->type}        {* Item type *}
    {$item->relid}       {* Related ID *}
    {$item->description} {* Item description *}
    {$item->amount}      {* Amount *}
    {$item->taxed}       {* Tax status *}
{/foreach}
```

## Basic Template Structure

### index.php

```php
<?php
// templates/invoice-{name}/index.php

use WHMCS\Invoice\Invoice;

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

$invoice = Invoice::find($invoiceId);
$client = $invoice->client;

// Assign template variables
$templateVariables = [
    'invoice' => $invoice,
    'client' => $client,
    'company_name' => \WHMCS\Config\Setting::getValue('CompanyName'),
    'company_logo' => \WHMCS\Config\Setting::getValue('LogoURL'),
    'company_address' => \WHMCS\Config\Setting::getValue('InvoiceTerms'),
];

$pdf->setTemplateVariables($templateVariables);
```

### Basic Template (HTML)

```smarty
{*
    templates/invoice-{name}/template.tpl
*}
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>Invoice #{$invoice->invoicenum}</title>
    <link rel="stylesheet" href="{$template_path}/stylesheet.css">
</head>
<body>
    <div class="invoice">
        <div class="invoice-header">
            <div class="company-info">
                <img src="{$company_logo}" alt="{$company_name}" class="logo">
                <p>{$company_name}</p>
                <p>{$company_address}</p>
            </div>
            <div class="invoice-info">
                <h1>INVOICE</h1>
                <p><strong>Invoice #:</strong> {$invoice->invoicenum}</p>
                <p><strong>Date:</strong> {$invoice->date}</p>
                <p><strong>Due Date:</strong> {$invoice->duedate}</p>
                <p><strong>Status:</strong> {$invoice->status}</p>
            </div>
        </div>
        
        <div class="billing-info">
            <h3>Bill To:</h3>
            <p>
                {if $client->companyname}
                    {$client->companyname}<br>
                {/if}
                {$client->firstname} {$client->lastname}<br>
                {$client->address1}<br>
                {if $client->address2}{$client->address2}<br>{/if}
                {$client->city}, {$client->state} {$client->postcode}<br>
                {$client->country}
            </p>
            <p>{$client->email}</p>
        </div>
        
        <table class="line-items">
            <thead>
                <tr>
                    <th>Description</th>
                    <th>Amount</th>
                </tr>
            </thead>
            <tbody>
                {foreach $invoice->items as $item}
                    <tr>
                        <td>{$item->description|nl2br}</td>
                        <td class="text-right">{$item->amount|currency_format}</td>
                    </tr>
                {/foreach}
            </tbody>
        </table>
        
        <table class="totals">
            <tr>
                <td>Subtotal:</td>
                <td class="text-right">{$invoice->subtotal|currency_format}</td>
            </tr>
            {if $invoice->tax > 0}
                <tr>
                    <td>Tax:</td>
                    <td class="text-right">{$invoice->tax|currency_format}</td>
                </tr>
            {/if}
            {if $invoice->tax2 > 0}
                <tr>
                    <td>Tax 2:</td>
                    <td class="text-right">{$invoice->tax2|currency_format}</td>
                </tr>
            {/if}
            <tr class="total">
                <td><strong>Total:</strong></td>
                <td class="text-right"><strong>{$invoice->total|currency_format}</strong></td>
            </tr>
            {if $invoice->credit > 0}
                <tr>
                    <td>Credit Applied:</td>
                    <td class="text-right">-{$invoice->credit|currency_format}</td>
                </tr>
            {/if}
            <tr class="balance">
                <td><strong>Balance Due:</strong></td>
                <td class="text-right"><strong>{$invoice->balance|currency_format}</strong></td>
            </tr>
        </table>
        
        {if $invoice->notes}
            <div class="notes">
                <h4>Notes</h4>
                <p>{$invoice->notes|nl2br}</p>
            </div>
        {/if}
        
        <div class="invoice-footer">
            <p>Thank you for your business!</p>
            <p>{$company_name} | {$system_url}</p>
        </div>
    </div>
</body>
</html>
```

## CSS Stylesheet

### stylesheet.css

```css
/* Invoice Stylesheet */

body {
    font-family: Arial, Helvetica, sans-serif;
    font-size: 14px;
    line-height: 1.6;
    color: #333;
    margin: 0;
    padding: 40px;
}

.invoice {
    max-width: 800px;
    margin: 0 auto;
    background: #fff;
}

.invoice-header {
    display: flex;
    justify-content: space-between;
    border-bottom: 2px solid #333;
    padding-bottom: 20px;
    margin-bottom: 20px;
}

.company-info img.logo {
    max-height: 80px;
}

.invoice-info {
    text-align: right;
}

.invoice-info h1 {
    color: #333;
    margin: 0 0 10px 0;
    font-size: 28px;
}

.billing-info {
    margin-bottom: 30px;
}

.billing-info h3 {
    margin-bottom: 10px;
}

.line-items {
    width: 100%;
    border-collapse: collapse;
    margin-bottom: 30px;
}

.line-items th {
    background: #333;
    color: #fff;
    padding: 12px;
    text-align: left;
}

.line-items th:last-child {
    text-align: right;
}

.line-items td {
    padding: 12px;
    border-bottom: 1px solid #ddd;
}

.line-items td:last-child {
    text-align: right;
}

.totals {
    width: 300px;
    margin-left: auto;
    margin-bottom: 30px;
}

.totals td {
    padding: 8px 12px;
}

.totals tr.total td,
.totals tr.balance td {
    font-size: 16px;
    border-top: 2px solid #333;
}

.totals tr.balance td {
    font-size: 18px;
    background: #f9f9f9;
}

.text-right {
    text-align: right;
}

.notes {
    background: #f9f9f9;
    padding: 15px;
    margin-bottom: 30px;
}

.invoice-footer {
    text-align: center;
    padding-top: 20px;
    border-top: 1px solid #ddd;
    font-size: 12px;
    color: #666;
}

@media print {
    body {
        padding: 0;
    }
    
    .invoice {
        max-width: 100%;
    }
}
```

## Advanced Features

### Tax Calculation Display

```smarty
<div class="tax-breakdown">
    <table class="totals">
        <tr>
            <td>Subtotal:</td>
            <td class="text-right">{$invoice->subtotal|currency_format}</td>
        </tr>
        {if $invoice->taxrate}
            <tr>
                <td>Tax ({$invoice->taxrate}%):</td>
                <td class="text-right">{$invoice->tax|currency_format}</td>
            </tr>
        {/if}
        {if $invoice->taxrate2}
            <tr>
                <td>Tax 2 ({$invoice->taxrate2}%):</td>
                <td class="text-right">{$invoice->tax2|currency_format}</td>
            </tr>
        {/if}
    </table>
</div>
```

### Payment Status Badge

```smarty
{assign var="status_class" value=""}
{if $invoice->status eq "Paid"}
    {assign var="status_class" value="paid"}
{elseif $invoice->status eq "Unpaid"}
    {assign var="status_class" value="unpaid"}
{elseif $invoice->status eq "Overdue"}
    {assign var="status_class" value="overdue"}
{elseif $invoice->status eq "Cancelled"}
    {assign var="status_class" value="cancelled"}
{/if}

<span class="status-badge status-{$status_class}">
    {$invoice->status}
</span>
```

### QR Code for Payment

```php
<?php
// In index.php
$qrCodeData = "{$system_url}/viewinvoice.php?id={$invoice->id}";
$qrCodeImage = generateQRCode($qrCodeData);

$templateVariables['qr_code'] = $qrCodeImage;
```

```smarty
<div class="payment-qr">
    <h4>Scan to Pay</h4>
    <img src="{$qr_code}" alt="Payment QR Code">
</div>
```

## PDF Generation

### Invoice PDF Template

```php
<?php
// Custom PDF generation logic

add_hook('InvoiceCreationPreEmail', 1, function($params) {
    $invoiceId = $params['invoiceid'];
    $invoice = Invoice::find($invoiceId);
    
    // Generate custom PDF
    $pdfContent = generateCustomPDF($invoice);
    
    return ['pdf_content' => $pdfContent];
});
```

## Best Practices

1. **Use inline CSS** for better email/PDF compatibility
2. **Include all necessary information** clearly
3. **Make totals prominent** and easy to find
4. **Include payment instructions** if applicable
5. **Support printing** with @media print styles
6. **Consider different paper sizes** (A4, Letter)

## See Also

- [Email Templates](../whmcs-email-templates.md)
- [Client Area Templates](../whmcs-clientarea-templates.md)
- [Template Variables](../whmcs-template-variables.md)