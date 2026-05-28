# WHMCS E-Invoice Integration Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Guide for integrating Vietnamese e-invoice systems (Misa, Viettel, VNPT, etc.) with WHMCS.

## Supported Providers

| Provider | Type | API |
|----------|------|-----|
| Misa Me Invoice | REST | api.misa.vn |
| Viettel S-Invoice | REST | m-invoice.vn |
| VNPT Invoice | REST | api.vnptinvoices.vn |
| FPT Invoice | REST | api.fpt.com |

## Misa Integration

```php
<?php
// modules/addons/misa_invoice/misa_invoice.php
if (!defined("WHMCS")) { die("Direct access denied"); }

use WHMCS\Database\Capsule;

function misa_invoice_config(): array {
    return [
        'name' => 'Misa Invoice',
        'description' => 'Misa e-Invoice integration for Vietnam',
        'version' => '1.0.0',
        'author' => 'HiTechCloud',
        'fields' => [
            'client_id' => ['FriendlyName' => 'Client ID', 'Type' => 'text'],
            'client_secret' => ['FriendlyName' => 'Client Secret', 'Type' => 'password'],
            'tax_code' => ['FriendlyName' => 'Tax Code', 'Type' => 'text'],
            'branch_code' => ['FriendlyName' => 'Branch Code', 'Type' => 'text'],
            'pattern_code' => ['FriendlyName' => 'Pattern Code', 'Type' => 'text'],
            'test_mode' => ['FriendlyName' => 'Test Mode', 'Type' => 'yesno'],
        ],
    ];
}

function misa_invoice_activate(): array {
    try {
        Capsule::schema()->create('mod_misa_invoice_settings', function($t) {
            $t->increments('id');
            $t->string('setting_key', 100)->unique();
            $t->text('setting_value')->nullable();
            $t->timestamps();
        });

        Capsule::schema()->create('mod_misa_invoice_invoices', function($t) {
            $t->increments('id');
            $t->integer('invoice_id')->unsigned();
            $t->string('misa_id', 100)->nullable();
            $t->string('invoice_number', 50)->nullable();
            $t->string('fkey', 100)->nullable();
            $t->string('status', 20)->default('pending');
            $t->json('response')->nullable();
            $t->timestamp('issued_at')->nullable();
            $t->timestamps();
            $t->index(['invoice_id']);
        });

        return ['status' => 'success', 'description' => 'Module activated'];
    } catch (\Exception $e) {
        return ['status' => 'error', 'description' => $e->getMessage()];
    }
}

// Invoice creation hook
add_hook('InvoiceCreation', 1, function($vars) {
    $invoiceId = $vars['invoiceid'];
    createMisaInvoice($invoiceId);
});

function createMisaInvoice(int $invoiceId): ?string {
    $invoice = localAPI('GetInvoice', ['invoiceid' => $invoiceId]);

    if ($invoice['result'] !== 'success') {
        return null;
    }

    $config = getMisaConfig();
    $accessToken = getMisaAccessToken($config);

    $payload = buildInvoicePayload($invoice);

    $response = callMisaApi('/invoices', $accessToken, $payload);

    if (isset($response['data']['invoice_id'])) {
        saveMisaInvoice($invoiceId, $response['data']);
        return $response['data']['invoice_id'];
    }

    return null;
}

function buildInvoicePayload(array $invoice): array {
    return [
        'invoice_series' => 'C',
        'invoice_pattern' => '01GTKT0/001',
        'buyer_name' => $invoice['clientname'],
        'buyer_tax_code' => $invoice['client_vat'] ?? '',
        'buyer_address' => $invoice['clientaddress'],
        'buyer_email' => $invoice['clientemail'],
        'items' => array_map(function($item) {
            return [
                'item_name' => $item['description'],
                'unit' => 'Suất',
                'quantity' => $item['qty'],
                'price' => $item['amount'],
                'amount' => $item['amount'],
                'tax_rate' => 10,
            ];
        }, $invoice['lineitems']),
    ];
}

function getMisaAccessToken(array $config): string {
    $ch = curl_init('https://api.misa.vn/token');
    curl_setopt_array($ch, [
        CURLOPT_POST => true,
        CURLOPT_POSTFIELDS => http_build_query([
            'grant_type' => 'client_credentials',
            'client_id' => $config['client_id'],
            'client_secret' => $config['client_secret'],
        ]),
        CURLOPT_RETURNTRANSFER => true,
    ]);
    $response = json_decode(curl_exec($ch), true);
    curl_close($ch);

    return $response['access_token'] ?? '';
}
```

## Checklist

- [ ] Module configuration
- [ ] Access token management
- [ ] Invoice payload builder
- [ ] API error handling
- [ ] Database storage
- [ ] Webhook handler

---

**Related Skills:**
- whmcs-addon-builder
- whmcs-vietnamese-billing
- whmcs-webhook-handler