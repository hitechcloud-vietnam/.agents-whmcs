# WHMCS SSL Module Builder Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Guide for building SSL certificate provisioning modules.

## When to Use

- Creating SSL certificate provisioning modules
- Integrating with certificate authorities
- Automating SSL issuance and validation

## SSL Module Structure

```
modules/servers/{sslmodule}/
├── {sslmodule}.php
├── lib/
│   ├── ApiClient.php
│   └── DcvValidator.php
└── templates/
    ├── overview.tpl
    └── certificate.tpl
```

## SSL Module Pattern

```php
<?php
// modules/servers/{sslmodule}/{sslmodule}.php
if (!defined("WHMCS")) { die("Direct access denied"); }

function {module}_MetaData(): array {
    return [
        'DisplayName' => 'SSL Provider',
        'APIVersion' => '1.1',
        'RequiresServer' => false,
    ];
}

function {module}_ConfigOptions(array $params): array {
    return [
        'Product' => [
            'Type' => 'dropdown',
            'Options' => 'DV SSL,OV SSL,EV SSL',
            'Default' => 'DV SSL',
        ],
        'DV Method' => [
            'Type' => 'dropdown',
            'Options' => 'email,http,dns',
        ],
    ];
}

function {module}_CreateAccount(array $params): string {
    try {
        $api = new \SSL\ApiClient($params);

        // Create order
        $order = $api->createOrder([
            'domain' => $params['domain'],
            'product' => $params['configoption1'],
            'csr' => generateCSR($params),
        ]);

        // Get DCV details
        $dcv = $api->getDCVTokens($order['id']);

        // Store for later use
        saveSSLOrder($params['serviceid'], [
            'order_id' => $order['id'],
            'dcv_token' => $dcv['http_token'],
            'dcv_method' => $params['configoption2'],
        ]);

        return 'pending';
    } catch (\Exception $e) {
        return 'Error: ' . $e->getMessage();
    }
}

function {module}_Pending(array $params): array {
    $order = getSSLOrder($params['serviceid']);

    $api = new \SSL\ApiClient($params);
    $status = $api->checkOrderStatus($order['order_id']);

    if ($status['status'] === 'issued') {
        // Certificate issued
        return [
            'completed' => true,
            'certificate' => $status['certificate'],
            'key' => $status['private_key'],
        ];
    }

    return ['completed' => false];
}

function {module}_AdminServices(array $params): array {
    return [
        'Order ID' => $params['configoptions']['order_id'] ?? 'N/A',
        'Validation' => $params['configoptions']['dcv_method'] ?? 'N/A',
    ];
}
```

## DCV Validation

```php
class DcvValidator {
    public function validateHTTP(string $domain, string $token): bool {
        $url = 'http://' . $domain . '/.well-known/pki-validation/' . $token . '.txt';
        $content = @file_get_contents($url);

        return strpos($content, 'ssl_provider_verification') !== false;
    }

    public function validateDNS(string $domain, string $token): array {
        return [
            'host' => '_acme-challenge.' . $domain,
            'type' => 'TXT',
            'value' => $token,
        ];
    }

    public function validateEmail(array $emails): array {
        // Return available emails for DCV
        return [
            'admin@' . $domain,
            'webmaster@' . $domain,
            'postmaster@' . $domain,
        ];
    }
}
```

## Checklist

- [ ] ConfigOptions with product tiers
- [ ] CSR generation
- [ ] DCV token retrieval
- [ ] Order status checking
- [ ] Certificate storage
- [ ] Renewal handling

---

**Related Skills:**
- whmcs-server-builder
- whmcs-certificate-handling
- whmcs-dcv-validation