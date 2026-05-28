# WHMCS Invoice Customizer Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Customize invoice templates with dynamic content and branding.

## Invoice Customizer

```php
<?php
class InvoiceCustomizer {
    private array $invoiceConfig;

    public function __construct() {
        $this->invoiceConfig = $this->loadConfig();
    }

    public function customizeTemplate(int $invoiceId, string $template): string {
        $invoice = $this->getInvoiceData($invoiceId);
        $client = $this->getClientData($invoice->userid);
        $items = $this->getInvoiceItems($invoiceId);

        $template = $this->replaceCompanyInfo($template);
        $template = $this->replaceClientInfo($template, $client);
        $template = $this->replaceInvoiceDetails($template, $invoice);
        $template = $this->replaceLineItems($template, $items);
        $template = $this->replaceTotals($template, $invoice);
        $template = $this->applyBranding($template, $client);

        return $template;
    }

    private function replaceCompanyInfo(string $template): string {
        $replacements = [
            '{company_name}' => $this->getConfig('CompanyName'),
            '{company_address}' => $this->getConfig('InvoiceAddress'),
            '{company_logo}' => $this->getConfig('LogoURL'),
            '{company_email}' => $this->getConfig('Email'),
            '{company_phone}' => $this->getConfig('Phone'),
            '{company_vat}' => $this->getConfig('TaxIdLabel'),
        ];

        return str_replace(array_keys($replacements), array_values($replacements), $template);
    }

    private function replaceLineItems(string $template, array $items): string {
        $rows = '';
        foreach ($items as $item) {
            $rows .= '<tr>';
            $rows .= '<td>' . htmlspecialchars($item->description) . '</td>';
            $rows .= '<td class="text-right">' . formatCurrency($item->amount, $item->currency) . '</td>';
            $rows .= '</tr>';
        }

        return str_replace('{line_items}', $rows, $template);
    }

    private function applyBranding(string $template, array $client): string {
        $style = Capsule::table('mod_invoice_branding')
            ->where('client_id', $client['id'])
            ->first();

        if ($style) {
            $customCSS = "
                <style>
                    .invoice-header { background: {$style->header_color}; }
                    .invoice-total { color: {$style->accent_color}; }
                    .btn-pay { background: {$style->accent_color}; }
                </style>
            ";
            $template = str_replace('</head>', $customCSS . '</head>', $template);
        }

        return $template;
    }
}
```

## Hook Integration

```php
add_hook('InvoiceCreation', 1, function($vars) {
    $customizer = new InvoiceCustomizer();

    Capsule::table('mod_invoice_customizations')->insert([
        'invoice_id' => $vars['invoiceid'],
        'custom_template' => 'custom_' . $vars['invoiceid'],
        'created_at' => date('Y-m-d H:i:s'),
    ]);
});

add_hook('InvoicePreOutput', 1, function($vars) {
    $customization = Capsule::table('mod_invoice_customizations')
        ->where('invoice_id', $vars['invoiceid'])
        ->first();

    if ($customization) {
        $customizer = new InvoiceCustomizer();
        return [
            'template_override' => $customizer->customizeTemplate(
                $vars['invoiceid'],
                $vars['template']
            ),
        ];
    }
});
```

---

**Related Skills:**
- whmcs-addon-builder
- whmcs-template-styling
- whmcs-invoice-customization