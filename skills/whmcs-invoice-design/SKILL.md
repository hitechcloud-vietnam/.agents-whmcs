# WHMCS Invoice Design Skill
# Version: 1.0 | Updated: 2026-05-29

## Purpose

Guide for customizing invoice design in WHMCS including templates, branding, and PDF generation.

## When to Use

- Customizing invoice templates
- Adding custom fields to invoices
- PDF invoice customization
- Branding invoice documents

## Invoice Template Structure

```
templates/orderforms/{template}/
├── invoice.phtml              ← Invoice template
└── invoice-pdf.phtml          ← PDF invoice template
```

### 1. Custom Invoice Template

```php
<?php
// templates/orderforms/custom-invoice/invoice.phtml
if (!defined("WHMCS")) { exit; }

/** @var \WHMCS\Invoice\Invoice $invoice */
$invoice = $invoice;

$companyLogo = $vars['company_logo'] ?? '';
$primaryColor = $vars['primary_color'] ?? '#1a1a1a';
$secondaryColor = $vars['secondary_color'] ?? '#666666';
?>
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>Invoice #<?= $invoice->getInvoiceNumber(); ?></title>
    <style>
        * { box-sizing: border-box; margin: 0; padding: 0; }
        body { font-family: 'Helvetica Neue', Arial, sans-serif; font-size: 14px; line-height: 1.5; color: #333; }

        .invoice-container { max-width: 800px; margin: 0 auto; padding: 40px; }

        .header { display: flex; justify-content: space-between; margin-bottom: 40px; }
        .logo img { max-height: 60px; }
        .invoice-title { text-align: right; }
        .invoice-title h1 { font-size: 28px; font-weight: 300; color: <?= $primaryColor; ?>; }
        .invoice-title .invoice-number { font-size: 16px; color: <?= $secondaryColor; ?>; margin-top: 5px; }

        .addresses { display: flex; gap: 40px; margin-bottom: 30px; }
        .address-block h3 { font-size: 12px; text-transform: uppercase; letter-spacing: 1px; color: <?= $secondaryColor; ?>; margin-bottom: 10px; }
        .address-block p { margin-bottom: 5px; }

        .invoice-details { display: flex; gap: 40px; margin-bottom: 30px; }
        .detail-item { display: flex; flex-direction: column; }
        .detail-label { font-size: 11px; text-transform: uppercase; color: <?= $secondaryColor; ?>; }
        .detail-value { font-weight: 600; font-size: 16px; }

        .items-table { width: 100%; border-collapse: collapse; margin-bottom: 30px; }
        .items-table th { padding: 12px 15px; text-align: left; font-size: 12px; text-transform: uppercase; letter-spacing: 1px; color: white; background: <?= $primaryColor; ?>; }
        .items-table td { padding: 15px; border-bottom: 1px solid #eee; }
        .items-table .amount { text-align: right; font-weight: 500; }
        .items-table tfoot td { font-weight: 600; background: #f9f9f9; }

        .notes-section { background: #f9f9f9; padding: 20px; border-radius: 4px; margin-bottom: 30px; }
        .notes-section h4 { font-size: 12px; text-transform: uppercase; letter-spacing: 1px; margin-bottom: 10px; }

        .footer { text-align: center; font-size: 12px; color: <?= $secondaryColor; ?>; padding-top: 20px; border-top: 1px solid #eee; }
    </style>
</head>
<body>
    <div class="invoice-container">
        <div class="header">
            <div class="logo">
                <?php if ($companyLogo): ?>
                    <img src="<?= $companyLogo; ?>" alt="Company Logo">
                <?php else: ?>
                    <h2><?= $companyName; ?></h2>
                <?php endif; ?>
            </div>
            <div class="invoice-title">
                <h1>Invoice</h1>
                <div class="invoice-number">#<?= $invoice->getInvoiceNumber(); ?></div>
            </div>
        </div>

        <div class="addresses">
            <div class="address-block">
                <h3>From</h3>
                <p><strong><?= $companyName; ?></strong></p>
                <p><?= nl2br($companyAddress); ?></p>
            </div>
            <div class="address-block">
                <h3>Bill To</h3>
                <p><strong><?= $invoice->getClient()->getFullName(); ?></strong></p>
                <p><?= nl2br($invoice->getClient()->getAddress()); ?></p>
                <p><?= $invoice->getClient()->getEmail(); ?></p>
            </div>
        </div>

        <div class="invoice-details">
            <div class="detail-item">
                <span class="detail-label">Invoice Date</span>
                <span class="detail-value"><?= $invoice->getDate()->format('M j, Y'); ?></span>
            </div>
            <div class="detail-item">
                <span class="detail-label">Due Date</span>
                <span class="detail-value"><?= $invoice->getDueDate()->format('M j, Y'); ?></span>
            </div>
            <div class="detail-item">
                <span class="detail-label">Amount Due</span>
                <span class="detail-value"><?= $invoice->getCurrency()->format($invoice->getTotal()); ?></span>
            </div>
        </div>

        <table class="items-table">
            <thead>
                <tr>
                    <th>Description</th>
                    <th style="width: 15%;">Qty</th>
                    <th style="width: 20%;" class="amount">Amount</th>
                </tr>
            </thead>
            <tbody>
                <?php foreach ($invoice->getLineItems() as $item): ?>
                    <tr>
                        <td>
                            <strong><?= $item->description; ?></strong>
                            <?php if ($item->unitDescription): ?>
                                <br><small style="color: <?= $secondaryColor; ?>;"><?= $item->unitDescription; ?></small>
                            <?php endif; ?>
                        </td>
                        <td><?= $item->quantity; ?></td>
                        <td class="amount"><?= $invoice->getCurrency()->format($item->amount); ?></td>
                    </tr>
                <?php endforeach; ?>
            </tbody>
            <tfoot>
                <tr>
                    <td colspan="2">Subtotal</td>
                    <td class="amount"><?= $invoice->getCurrency()->format($invoice->getSubtotal()); ?></td>
                </tr>
                <?php if ($invoice->getTax() > 0): ?>
                    <tr>
                        <td colspan="2">Tax (<?= $invoice->getTaxRate(); ?>%)</td>
                        <td class="amount"><?= $invoice->getCurrency()->format($invoice->getTax()); ?></td>
                    </tr>
                <?php endif; ?>
                <tr>
                    <td colspan="2"><strong>Total Due</strong></td>
                    <td class="amount"><strong><?= $invoice->getCurrency()->format($invoice->getTotal()); ?></strong></td>
                </tr>
            </tfoot>
        </table>

        <?php if ($invoice->getNotes()): ?>
            <div class="notes-section">
                <h4>Notes</h4>
                <p><?= nl2br($invoice->getNotes()); ?></p>
            </div>
        <?php endif; ?>

        <div class="footer">
            <p><?= $companyName; ?> | <?= $companyAddress; ?></p>
            <p>Thank you for your business!</p>
        </div>
    </div>
</body>
</html>
```

### 2. PDF Invoice Customization

```php
<?php
// templates/orderforms/custom-invoice/invoice-pdf.phtml
if (!defined("WHMCS")) { exit; }

// TCPDF configuration for better PDF output
$pdf->SetCreator('WHMCS');
$pdf->SetAuthor($companyName);
$pdf->SetTitle('Invoice #' . $invoice->getInvoiceNumber());
$pdf->SetSubject('Invoice');

// Custom page size (A4)
$pdf->setPageFormat('A4', 'P');

// Margins
$pdf->SetMargins(20, 20, 20);
$pdf->SetHeaderMargin(10);
$pdf->SetFooterMargin(10);

// Enable automatic page break
$pdf->SetAutoPageBreak(true, 25);

// Set default font
$pdf->SetFont('helvetica', '', 10);

// Colors
$primaryColor = [26, 26, 26];
$secondaryColor = [102, 102, 102];
$lightGray = [245, 245, 245];

// Header
$pdf->SetFont('helvetica', 'B', 20);
$pdf->SetTextColor($primaryColor[0], $primaryColor[1], $primaryColor[2]);
$pdf->Cell(0, 15, '', 0, 0, 'L');
$pdf->Ln();
$pdf->Cell(0, 10, 'INVOICE', 0, 0, 'R');
$pdf->Ln(15);

// Invoice info box
$pdf->SetFont('helvetica', '', 10);
$pdf->SetTextColor($secondaryColor[0], $secondaryColor[1], $secondaryColor[2]);

// From address
$pdf->SetFont('helvetica', 'B', 10);
$pdf->SetTextColor($primaryColor[0], $primaryColor[1], $primaryColor[2]);
$pdf->MultiCell(85, 5, $companyName, 0, 'L');
$pdf->SetFont('helvetica', '', 9);
$pdf->SetTextColor($secondaryColor[0], $secondaryColor[1], $secondaryColor[2]);
$pdf->MultiCell(85, 4, $companyAddress, 0, 'L');

// To address
$pdf->SetY(20);
$pdf->SetX(120);
$pdf->SetFont('helvetica', 'B', 10);
$pdf->SetTextColor($primaryColor[0], $primaryColor[1], $primaryColor[2]);
$pdf->MultiCell(70, 5, 'Bill To', 0, 'L');
$pdf->SetFont('helvetica', '', 9);
$pdf->SetTextColor($secondaryColor[0], $secondaryColor[1], $secondaryColor[2]);
$pdf->SetX(120);
$pdf->MultiCell(70, 4, $invoice->getClient()->getFullName(), 0, 'L');
$pdf->SetX(120);
$pdf->MultiCell(70, 4, $invoice->getClient()->getAddress(), 0, 'L');

// Invoice details
$pdf->SetY($pdf->GetY() + 10);
$pdf->setX(120);
$pdf->SetFont('helvetica', '', 9);
$pdf->Cell(40, 5, 'Invoice Date:', 0, 0, 'L');
$pdf->Cell(30, 5, $invoice->getDate()->format('M j, Y'), 0, 1, 'R');
$pdf->setX(120);
$pdf->Cell(40, 5, 'Due Date:', 0, 0, 'L');
$pdf->Cell(30, 5, $invoice->getDueDate()->format('M j, Y'), 0, 1, 'R');
$pdf->setX(120);
$pdf->SetFont('helvetica', 'B', 9);
$pdf->Cell(40, 5, 'Amount Due:', 0, 0, 'L');
$pdf->Cell(30, 5, $invoice->getCurrency()->format($invoice->getTotal()), 0, 1, 'R');

// Line items table
$pdf->Ln(10);
$pdf->SetFont('helvetica', 'B', 9);
$pdf->SetFillColor($primaryColor[0], $primaryColor[1], $primaryColor[2]);
$pdf->SetTextColor(255, 255, 255);
$pdf->Cell(95, 8, 'Description', 1, 0, 'L', true);
$pdf->Cell(20, 8, 'Qty', 1, 0, 'C', true);
$pdf->Cell(35, 8, 'Amount', 1, 0, 'R', true);

$pdf->Ln();
$pdf->SetTextColor($primaryColor[0], $primaryColor[1], $primaryColor[2]);
$pdf->SetFont('helvetica', '', 9);
$fill = false;

foreach ($invoice->getLineItems() as $item) {
    $fillColor = $fill ? $lightGray : [255, 255, 255];
    $pdf->SetFillColor($fillColor[0], $fillColor[1], $fillColor[2]);

    $y = $pdf->GetY();
    $x = $pdf->GetX();

    // Description cell
    $pdf->Cell(95, 8, $item->description, 1, 0, 'L', true);
    $pdf->Cell(20, 8, $item->quantity, 1, 0, 'C', true);
    $pdf->Cell(35, 8, $invoice->getCurrency()->format($item->amount), 1, 0, 'R', true);
    $pdf->Ln();

    $fill = !$fill;
}

// Totals
$pdf->Ln(5);
$pdf->setX(115);
$pdf->Cell(35, 6, 'Subtotal:', 0, 0, 'R');
$pdf->Cell(35, 6, $invoice->getCurrency()->format($invoice->getSubtotal()), 0, 1, 'R');

if ($invoice->getTax() > 0) {
    $pdf->setX(115);
    $pdf->Cell(35, 6, 'Tax (' . $invoice->getTaxRate() . '%):', 0, 0, 'R');
    $pdf->Cell(35, 6, $invoice->getCurrency()->format($invoice->getTax()), 0, 1, 'R');
}

$pdf->SetFont('helvetica', 'B', 10);
$pdf->setX(115);
$pdf->Cell(35, 8, 'Total Due:', 0, 0, 'R');
$pdf->Cell(35, 8, $invoice->getCurrency()->format($invoice->getTotal()), 0, 1, 'R');

// Notes
if ($invoice->getNotes()) {
    $pdf->Ln(10);
    $pdf->SetFont('helvetica', 'B', 9);
    $pdf->Cell(0, 6, 'Notes:', 0, 1, 'L');
    $pdf->SetFont('helvetica', '', 9);
    $pdf->MultiCell(170, 4, $invoice->getNotes(), 0, 'L');
}

// Footer
$pdf->SetY(-30);
$pdf->SetFont('helvetica', '', 8);
$pdf->SetTextColor($secondaryColor[0], $secondaryColor[1], $secondaryColor[2]);
$pdf->Cell(0, 10, $companyName . ' | ' . $companyAddress, 0, 1, 'C');
```

### 3. Invoice Hook for Dynamic Content

```php
<?php
// hooks.php - Add custom invoice functionality
add_hook('InvoiceCreationPreOutput', 1, function($vars) {
    // Add custom branding variables
    $vars['company_logo'] = \WHMCS\Config\Setting::getValue('CompanyLogo');
    $vars['primary_color'] = \WHMCS\Config\Setting::getValue('InvoicePrimaryColor') ?? '#1a1a1a';
    $vars['secondary_color'] = \WHMCS\Config\Setting::getValue('InvoiceSecondaryColor') ?? '#666666';

    return $vars;
});

add_hook('InvoiceEmailPreSend', 1, function($vars) {
    $invoice = $vars['invoice'];

    // Add payment instructions for bank transfers
    $bankDetails = \WHMCS\Config\Setting::getValue('BankTransferDetails');

    if ($bankDetails) {
        $notes = $invoice->getNotes();
        $invoice->setNotes($notes . "\n\nBank Transfer Details:\n" . $bankDetails);
    }

    return $vars;
});

add_hook('InvoicePaid', 1, function($vars) {
    // Mark invoice with custom receipt number
    Capsule::table('tblinvoices')
        ->where('id', $vars['invoice_id'])
        ->update([
            'custom_receipt_number' => generateReceiptNumber($vars['invoice_id']),
        ]);
});
```

### 4. Admin Configuration

```php
<?php
// modules/addons/{module}/admin/invoice_settings.php
if (!defined("WHMCS")) { die("Direct access denied"); }

use WHMCS\Database\Capsule;

// Save settings
if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    check_token('WHMCS.admin.default');

    $config = Capsule::table('tblconfiguration')->get();

    $settings = [
        'InvoicePrimaryColor' => $_POST['primary_color'] ?? '#1a1a1a',
        'InvoiceSecondaryColor' => $_POST['secondary_color'] ?? '#666666',
        'InvoiceCompanyLogo' => $_POST['company_logo'] ?? '',
        'InvoiceTemplate' => $_POST['template'] ?? 'custom-invoice',
        'BankTransferDetails' => $_POST['bank_details'] ?? '',
    ];

    foreach ($settings as $key => $value) {
        Capsule::table('tblconfiguration')
            ->where('setting', $key)
            ->update(['value' => $value]);
    }

   flash('success', 'Invoice settings updated');
    redirect('addonmodules.php?module={module}&action=invoice_settings');
}

// Load current settings
$currentSettings = [
    'primary_color' => Capsule::table('tblconfiguration')->where('setting', 'InvoicePrimaryColor')->first()->value ?? '#1a1a1a',
    'secondary_color' => Capsule::table('tblconfiguration')->where('setting', 'InvoiceSecondaryColor')->first()->value ?? '#666666',
    'company_logo' => Capsule::table('tblconfiguration')->where('setting', 'InvoiceCompanyLogo')->first()->value ?? '',
    'template' => Capsule::table('tblconfiguration')->where('setting', 'InvoiceTemplate')->first()->value ?? 'custom-invoice',
    'bank_details' => Capsule::table('tblconfiguration')->where('setting', 'BankTransferDetails')->first()->value ?? '',
];
```

## Checklist

- [ ] Invoice template structure (invoice.phtml)
- [ ] PDF invoice template (invoice-pdf.phtml)
- [ ] Custom styling and branding
- [ ] Custom fields support
- [ ] Dynamic content hooks
- [ ] Admin configuration page
- [ ] Email customization
- [ ] Payment method instructions

---

**Related Skills:**
- whmcs-template-styling
- whmcs-email-template-builder
- whmcs-service-billing
