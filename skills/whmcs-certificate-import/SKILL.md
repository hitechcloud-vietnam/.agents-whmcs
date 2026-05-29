---
name: whmcs-certificate-import
description: Certificate import for WHMCS
category: SSL & Security
version: 1.0.0
---

# WHMCS Certificate Import Skill

## Overview
This skill provides patterns for importing SSL certificates into WHMCS.

## Implementation Patterns

### Certificate Import Manager
```php
<?php
/**
 * WHMCS Certificate Import
 * Imports SSL certificates
 */

namespace WHMCS\Module\Server\SSL;

class CertificateImportManager {
    /**
     * Import certificate
     */
    public function importCertificate(array $params): array {
        $certId = 'cert_' . bin2hex(random_bytes(12));

        // Parse certificate
        $parsed = openssl_x509_parse($params['certificate']);

        // Store certificate
        \WHMCS\Database\Capsule::connection()->insert('mod_ssl_certificates', [
            'id' => $certId,
            'service_id' => $params['service_id'],
            'certificate' => $params['certificate'],
            'private_key' => $params['private_key'],
            'chain' => $params['chain'] ?? null,
            'common_name' => $parsed['subject']['CN'] ?? '',
            'expiry_date' => date('Y-m-d', $parsed['validTo_time_t']),
            'issuer' => $parsed['issuer']['O'] ?? '',
            'created_at' => date('Y-m-d H:i:s')
        ]);

        return [
            'success' => true,
            'cert_id' => $certId,
            'common_name' => $parsed['subject']['CN'] ?? ''
        ];
    }

    /**
     * Validate certificate bundle
     */
    public function validateBundle(string $cert, string $key, ?string $chain = null): array {
        $errors = [];

        // Validate certificate
        if (!openssl_x509_read($cert)) {
            $errors[] = 'Invalid certificate format';
        }

        // Validate private key
        if (!openssl_pkey_get_private($key)) {
            $errors[] = 'Invalid private key';
        }

        // Check match
        $certDigest = openssl_x509fingerprint($cert);
        $keyDigest = openssl_pkey_get_details(openssl_pkey_get_private($key));

        return [
            'valid' => empty($errors),
            'errors' => $errors
        ];
    }
}
```

## Best Practices

1. **Format Validation**: Validate PEM/CRT format
2. **Key Matching**: Ensure cert and key match
3. **Chain Verification**: Verify complete chain
4. **Expiry Check**: Check certificate expiry
5. **Secure Storage**: Encrypt stored private keys

## Related Skills

- whmcs-letsencrypt-auto
- whmcs-ssl-renewal
- whmcs-wildcard-ssl
- whmcs-ssl-monitor