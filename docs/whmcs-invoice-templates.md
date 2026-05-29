# WHMCS Invoice Templates

## Overview

WHMCS invoice templates control the visual appearance and content of generated invoices. Templates are PHP-based and can be customized for branding, layout, and content display.

## Template Location

```
/whmcs/templates/invoice/
├── classic/
├── modern/
├── boxified/
└── your-custom-template/
```

## Template Files Structure

```
invoice-pdf.html          # PDF invoice content
invoice-email.html        # Email invoice attachment
invoice-print.css         # Print styles
invoice-pdf.php           # PDF generation script (legacy)
```

## PDF Template Variables

### Client Information
```php
{$clientdetails.firstname}      // First name
{$clientdetails.lastname}       // Last name
{$clientdetails.companyname}     // Company name
{$clientdetails.email}          // Email address
{$clientdetails.address1}       // Address line 1
{$clientdetails.address2}       // Address line 2
{$clientdetails.city}          // City
{$clientdetails.state}         // State/Region
{$clientdetails.postcode}      // Postal code
{$clientdetails.country}       // Country
{$clientdetails.phonenumber}   // Phone number
```

### Invoice Details
```php
{$invoicenumber}         // Invoice number
{$invoicedate}           // Issue date
{$datedue}               // Due date
{$status}                // Status (Paid/Unpaid/Overdue)
{$payterm}               // Payment terms
```

### Line Items
```php
{foreach from=$lineitems item=item}
    {$item.description}       // Item description
    {$item.amount}            // Amount
    {$item.taxed}             // Tax applied (true/false)
    {$item.taxrate}           // Tax rate percentage
    {$item.taxamount}         // Tax amount
{/foreach}
```

### Totals
```php
{$subtotal}              // Subtotal before tax
{$tax}                   // Total tax amount
{$total}                 // Grand total
{$balance}               // Balance due
{$credit}                // Credit applied
```

### Company Information
```php
{$companyname}           // Your company name
{$companyaddress}        // Company address
{$companyphone}          // Company phone
{$companyemail}          // Company email
{$companylogo}           // Logo path/URL
{$tax_id}                // Tax ID/VAT number
```

## PDF Generation Method

WHMCS 8.0+ uses HTML-based PDF generation:

```php
// In Configuration > Invoices
PDF Engine: dompdf        // Default
```

### Alternative Engines
- dompdf (default)
- mpdf
- tcpdf (legacy)

## Custom Template Development

### Basic HTML Structure
```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="utf-8">
    <title>Invoice {$invoicenumber}</title>
    <style>
        /* CSS for PDF rendering */
        body { font-family: Arial, sans-serif; }
        .invoice-header { text-align: center; }
        .line-items { width: 100%; border-collapse: collapse; }
        .line-items th, .line-items td { padding: 8px; }
    </style>
</head>
<body>
    <!-- Invoice content -->
</body>
</html>
```

### Important Notes
- Use inline CSS for better PDF compatibility
- Avoid complex CSS (flexbox, grid) - use tables
- Use standard web fonts or embed fonts
- Test on actual PDF output

## Email Template

```html
<!-- invoice-email.html -->
<p>Dear {$client_name},</p>
<p>Thank you for your business. Please find attached invoice {$invoicenumber}.</p>
<p>
    <strong>Invoice Total:</strong> {$total}<br>
    <strong>Due Date:</strong> {$datedue}
</p>
<p>Pay now: <a href="{$invoice_url}">{$invoice_url}</a></p>
```

## Styling Best Practices

1. **Keep it simple**: Complex layouts may break in PDF
2. **Use tables**: Tabular data renders better
3. **Inline styles**: All styles should be inline
4. **Standard fonts**: Arial, Helvetica, Times New Roman
5. **Fallback fonts**: Define font stacks
6. **A4/Letter size**: Standard page dimensions

## Customization Examples

### Adding Company Logo
```php
<img src="{$companylogo}" alt="{$companyname}" style="max-width:200px;">
```

### Tax Breakdown
```php
{foreach from=$taxrates item=taxrate}
    <tr>
        <td>{$taxrate.name} ({$taxrate.rate}%)</td>
        <td>{$taxrate.amount}</td>
    </tr>
{/foreach}
```

### Payment Details
```php
{if $payment_method}
    <p>Payment Method: {$payment_method}</p>
{/if}
```

## Testing Templates

1. Create a test invoice in WHMCS
2. Navigate to invoice in Admin
3. Click "Email" or "Download PDF"
4. Review and adjust as needed

## Related Documentation

- [Invoice Generation](./whmcs-invoice-generation.md)
- [Invoice Reminders](./whmcs-invoice-reminders.md)
- [Email Templates](./whmcs-email-templates.md)
- [PDF Customization](./whmcs-pdf-customization.md)