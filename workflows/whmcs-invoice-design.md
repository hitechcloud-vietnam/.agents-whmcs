# WHMCS Invoice Design Customization Workflow

## Overview
Customize invoice templates, branding, and styling for professional billing documents.

## Prerequisites
- WHMCS v8.0+
- PDF generation tools

## Step-by-Step Guide

### Step 1: Invoice Template Location
```bash
# Template files location
/var/www/whmcs/templates/invoice_templates/
```

### Step 2: Invoice Hook for Custom Data
```php
<?php
add_hook('InvoiceCreation', 1, function($vars) {
    $invoiceId = $vars['invoiceid'];
    
    // Add custom fields to invoice
    return [
        'custom_footer' => 'Thank you for your business!',
        'bank_details' => 'Bank: XYZ Bank, Account: 1234567890',
    ];
});
```

### Step 3: PDF Invoice Customization
```php
<?php
add_hook('InvoicePDFOutput', 1, function($vars) {
    $pdf = $vars['pdf'];
    
    // Add company logo
    $pdf->Image('/path/to/logo.png', 10, 10, 50);
    
    // Add custom header
    $pdf->SetFont('Arial', 'B', 12);
    $pdf->Cell(0, 10, 'Custom Invoice Header', 0, 1, 'R');
    
    return $pdf;
});
```

## Checklist
- Invoice template modified
- Company branding applied
- Custom fields added
- PDF generation tested
