---
name: whmcs-crl-distribution
description: Certificate revocation for WHMCS
category: SSL & Security
version: 1.0.0
---

# WHMCS CRL Distribution Skill

## Overview
This skill provides patterns for managing certificate revocation lists in WHMCS.

## Implementation Patterns

### CRL Manager
```php
<?php
/**
 * WHMCS Certificate Revocation
 * Manages CRL/OCSP checking
 */

namespace WHMCS\Module\Server\SSL;

class CRLManager {
    /**
     * Check if certificate is revoked
     */
    public function isRevoked(string $serialNumber): bool {
        $revoked = \WHMCS\Database\Capsule::connection()->select(
            "SELECT id FROM mod_revoked_certs WHERE serial_number = ?",
            [$serialNumber]
        );

        return !empty($revoked);
    }

    /**
     * Update CRL
     */
    public function updateCRL(string $crlUrl): array {
        $crlContent = file_get_contents($crlUrl);
        $certificates = $this->parseCRL($crlContent);

        foreach ($certificates as $serial) {
            $this->addToRevokedList($serial);
        }

        return ['updated' => count($certificates)];
    }

    private function parseCRL(string $content): array {
        // Parse CRL content
        return [];
    }
}
```

## Database Schema
```sql
CREATE TABLE `mod_revoked_certs` (
  `id` INT AUTO_INCREMENT PRIMARY KEY,
  `serial_number` VARCHAR(100) NOT NULL UNIQUE,
  `revoked_at` DATETIME NOT NULL,
  `reason` VARCHAR(50)
);
```

## Best Practices

1. **CRL Updates**: Regular CRL updates
2. **OCSP Stapling**: Use OCSP stapling
3. **Timeout Handling**: Handle CRL/OCSP timeouts
4. **Fail-Safe**: Default to secure on errors
5. **Monitoring**: Monitor revocation checks

## Related Skills

- whmcs-ocsp-stapling
- whmcs-ssl-monitor
- whmcs-ssl-renewal
- whmcs-certificate-transparency