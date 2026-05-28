# WHMCS SSL Certificate Module Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Guide for building SSL certificate provisioning modules.

## When to Use

- Creating SSL certificate provisioning modules
- Implementing automatic SSL provisioning and renewal
- Building DV/OV/EV certificate ordering

## SSL Module Pattern

```php
<?php
// modules/servers/{sslmodule}/{sslmodule}.php
if (!defined("WHMCS")) { die("Direct access denied"); }

function {sslmodule}_MetaData(): array {
    return [
        'DisplayName' => 'SSL Provider',
        'APIVersion' => '1.1',
        'RequiresServer' => true,
        'DefaultNonSSLPort' => 443,
    ];
}

function {sslmodule}_ConfigOptions(array $params): array {
    return [
        'ProductType' => [
            'Type' => 'dropdown',
            'Options' => 'dv,ov,ev',
            'Default' => 'dv',
        ],
        'ValidationMethod' => [
            'Type' => 'dropdown',
            'Options' => 'email,dns,http',
            'Default' => 'email',
        ],
        'ServerType' => [
            'Type' => 'dropdown',
            'Options' => 'apache,nginx,iis,other',
            'Default' => 'apache',
        ],
    ];
}

function {sslmodule}_CreateAccount(array $params): string {
    $api = new \SSL\ApiClient($params);

    try {
        // Generate CSR
        $csr = generateCSR(
            $params['customfields']['csr'] ?? '',
            $params['customfields']['domain'] ?? $params['domain']
        );

        // Submit order
        $order = $api->submitOrder([
            'product' => $params['configoption1'],
            'csr' => $csr,
            'domain' => $params['customfields']['domain'] ?? $params['domain'],
            'validation_email' => $params['customfields']['validation_email'] ?? $params['email'],
        ]);

        // Store certificate info
        Capsule::table('mod_ssl_certificates')->insert([
            'service_id' => $params['serviceid'],
            'order_id' => $order['id'],
            'status' => 'pending_validation',
            'created_at' => date('Y-m-d H:i:s'),
        ]);

        // Save validation instructions
        saveCustomFieldValue($params['serviceid'], 'validation_instructions', $order['validation_instructions']);

        return 'success';
    } catch (\Exception $e) {
        return 'Error: ' . $e->getMessage();
    }
}

function {sslmodule}_RenewAccount(array $params): string {
    $api = new \SSL\ApiClient($params);
    $existingCert = Capsule::table('mod_ssl_certificates')
        ->where('service_id', $params['serviceid'])
        ->first();

    try {
        $renewal = $api->renewCertificate($existingCert->order_id);
        return 'success';
    } catch (\Exception $e) {
        return 'Error: ' . $e->getMessage();
    }
}

function {sslmodule}_TerminateAccount(array $params): string {
    $existingCert = Capsule::table('mod_ssl_certificates')
        ->where('service_id', $params['serviceid'])
        ->first();

    if ($existingCert) {
        Capsule::table('mod_ssl_certificates')
            ->where('id', $existingCert->id)
            ->update(['status' => 'terminated']);
    }

    return 'success';
}

function {sslmodule}_AdminServices(array $params): array {
    $cert = Capsule::table('mod_ssl_certificates')
        ->where('service_id', $params['serviceid'])
        ->first();

    return [
        'Certificate Status' => $cert->status ?? 'N/A',
        'Order ID' => $cert->order_id ?? 'N/A',
        'Expires At' => $cert->expires_at ?? 'N/A',
    ];
}

function {sslmodule}_TestConnection(array $params): array {
    try {
        $api = new \SSL\ApiClient($params);
        $result = $api->testConnection();

        return ['success' => true, 'error' => ''];
    } catch (\Exception $e) {
        return ['success' => false, 'error' => $e->getMessage()];
    }
}

function {sslmodule}_ClientAreaAllowedFunctions(): array {
    return [
        'DownloadCertificate' => 'Download Certificate',
        'ReIssueCertificate' => 'Re-Issue Certificate',
    ];
}

function DownloadCertificate(array $params): array {
    $cert = Capsule::table('mod_ssl_certificates')
        ->where('service_id', $params['serviceid'])
        ->first();

    if (!$cert || $cert->status !== 'issued') {
        return ['error' => 'Certificate not available'];
    }

    $api = new \SSL\ApiClient($params);
    $certs = $api->downloadCertificate($cert->order_id);

    return [
        'symlink' => 'download',
        'download' => [
            'certificate' => $certs['certificate'],
            'ca_bundle' => $certs['ca_bundle'],
            'private_key' => getCustomFieldValue($params['serviceid'], 'private_key'),
        ],
    ];
}
```

### CSR Generation (for DV certificates)

```php
function generateCSR(string $csrData, string $domain): string {
    if (!empty($csrData)) {
        return $csrData;
    }

    // Generate new CSR if not provided
    $config = [
        'private_key_bits' => 2048,
        'private_key_type' => OPENSSL_KEYTYPE_RSA,
    ];

    $key = openssl_pkey_new($config);
    openssl_pkey_export($key, $privateKey);

    $csrConfig = [
        'countryName' => 'US',
        'stateOrProvinceName' => 'State',
        'localityName' => 'City',
        'organizationName' => 'Organization',
        'commonName' => $domain,
    ];

    $csr = openssl_csr_new($csrConfig, $key, ['private_key_bits' => 2048]);
    openssl_csr_export($csr, $csrOutput);

    return $csrOutput;
}
```

### Certificate Validation Handler

```php
// Handle DCV (Domain Control Validation) emails/callbacks
function {sslmodule}_callback(array $params): void {
    $challenge = $_GET['challenge'] ?? '';
    $token = $_GET['token'] ?? '';

    // Validate challenge
    Capsule::table('mod_ssl_validations')
        ->where('token', $token)
        ->update([
            'challenge' => $challenge,
            'validated_at' => date('Y-m-d H:i:s'),
        ]);

    // Update certificate status
    Capsule::table('mod_ssl_certificates')
        ->where('order_id', $_GET['order_id'])
        ->update(['status' => 'issued']);
}
```

---

**Related Skills:**
- whmcs-server-builder
- whmcs-cron-automation
- whmcs-webhook-handler
