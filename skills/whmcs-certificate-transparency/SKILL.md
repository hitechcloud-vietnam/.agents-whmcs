---
name: whmcs-certificate-transparency
description: CT log monitoring for WHMCS
category: SSL & Security
version: 1.0.0
---

# WHMCS Certificate Transparency Monitoring Skill

## Overview
This skill provides patterns for monitoring certificate transparency logs in WHMCS.

## Implementation Patterns

### CT Log Monitor
```php
<?php
/**
 * WHMCS Certificate Transparency
 * Monitors CT logs for certificates
 */

namespace WHMCS\Module\Server\SSL;

class CertificateTransparencyMonitor {
    /**
     * Search CT logs for domain
     */
    public function searchCTLogs(string $domain): array {
        $api = 'https://crt.sh/?q=' . urlencode($domain) . '&output=json';

        $response = file_get_contents($api);
        $certificates = json_decode($response, true);

        return array_map(function($cert) {
            return [
                'issuer' => $cert['issuer_name'] ?? '',
                'name' => $cert['name_value'] ?? '',
                'timestamp' => $cert['entry_timestamp'] ?? ''
            ];
        }, $certificates ?? []);
    }

    /**
     * Detect unauthorized certificates
     */
    public function detectUnauthorized(string $domain, array $authorizedIssuers): array {
        $certs = $this->searchCTLogs($domain);

        $unauthorized = [];
        foreach ($certs as $cert) {
            $isAuthorized = false;
            foreach ($authorizedIssuers as $issuer) {
                if (strpos($cert['issuer'], $issuer) !== false) {
                    $isAuthorized = true;
                    break;
                }
            }

            if (!$isAuthorized) {
                $unauthorized[] = $cert;
            }
        }

        return $unauthorized;
    }
}
```

## Best Practices

1. **Monitor Subdomains**: Watch all subdomains
2. **Alert Unauthorized**: Alert on unknown certs
3. **Regular Checks**: Check CT logs periodically
4. **Authorized List**: Maintain authorized issuers list
5. **Automation**: Automate detection and alerts

## Related Skills

- whmcs-ssl-monitor
- whmcs-ssl-renewal
- whmcs-certificate-import
- whmcs-ssl-scan-vulnerability