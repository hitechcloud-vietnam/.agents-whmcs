---
name: whmcs-tls-configuration
description: TLS version management for WHMCS
category: SSL & Security
version: 1.0.0
---

# WHMCS TLS Configuration Skill

## Overview
This skill provides patterns for TLS version management in WHMCS.

## Implementation Patterns

### TLS Configuration Manager
```php
<?php
/**
 * WHMCS TLS Configuration
 * Manages TLS settings
 */

namespace WHMCS\Module\Server\SSL;

class TLSConfigManager {
    /**
     * Generate Nginx TLS config
     */
    public function generateNginxConfig(): string {
        return <<<CONFIG
# TLS Configuration
ssl_protocols TLSv1.2 TLSv1.3;
ssl_ciphers 'ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256';
ssl_prefer_server_ciphers on;
ssl_dhparam /etc/nginx/ssl/dhparam.pem;

# HSTS
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

# OCSP Stapling
ssl_stapling on;
ssl_stapling_verify on;
CONFIG;
    }

    /**
     * Check TLS version compliance
     */
    public function checkCompliance(string $domain): array {
        $result = shell_exec("testssl --protocols {$domain} 2>&1");

        return [
            'tls_1_0' => strpos($result, 'TLSv1.0') !== false,
            'tls_1_1' => strpos($result, 'TLSv1.1') !== false,
            'tls_1_2' => strpos($result, 'TLSv1.2') !== false,
            'tls_1_3' => strpos($result, 'TLSv1.3') !== false
        ];
    }
}
```

## TLS Version Requirements

| Version | Status | Notes |
|---------|--------|-------|
| TLS 1.0 | Disabled | Deprecated, security risks |
| TLS 1.1 | Disabled | Deprecated |
| TLS 1.2 | Required | Minimum secure version |
| TLS 1.3 | Recommended | Latest, fastest, most secure |

## Best Practices

1. **Disable Old Versions**: Disable TLS 1.0/1.1
2. **Enable TLS 1.3**: Support latest protocol
3. **Strong Ciphers**: Use secure cipher suites
4. **Perfect Forward Secrecy**: Enable PFS
5. **Regular Audits**: Audit TLS configuration

## Related Skills

- whmcs-cipher-suites
- whmcs-certificate-pinning
- whmcs-hsts-preload
- whmcs-ssl-monitor