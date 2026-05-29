---
name: whmcs-security-headers
description: Security header setup for WHMCS
category: SSL & Security
version: 1.0.0
---

# WHMCS Security Headers Setup Skill

## Overview
This skill provides patterns and implementations for configuring security headers in WHMCS, including CSP, X-Frame-Options, X-Content-Type-Options, and other protective headers.

## Implementation Patterns

### Security Headers Manager
```php
<?php
/**
 * WHMCS Security Headers Configuration
 * Implements web security headers
 */

namespace WHMCS\Module\Server\Security;

class SecurityHeadersManager {
    private $db;

    public function __construct() {
        $this->db = \WHMCS\Database\Capsule::connection();
    }

    /**
     * Configure security headers for service
     */
    public function configureHeaders(int $serviceId, array $config): array {
        $headers = [
            'x_frame_options' => $config['x_frame_options'] ?? 'SAMEORIGIN',
            'x_content_type_options' => $config['x_content_type_options'] ?? 'nosniff',
            'x_xss_protection' => $config['x_xss_protection'] ?? '1; mode=block',
            'referrer_policy' => $config['referrer_policy'] ?? 'strict-origin-when-cross-origin',
            'permissions_policy' => $config['permissions_policy'] ?? 'geolocation=(), microphone=(), camera=()',
            'csp_enabled' => $config['csp_enabled'] ?? true,
            'csp_config' => $config['csp_config'] ?? []
        ];

        // Store configuration
        $this->db->update('mod_service_security', [
            'security_headers' => json_encode($headers),
            'security_enabled' => true
        ], ['service_id' => $serviceId]);

        // Generate server configuration
        $serverConfig = $this->generateServerConfig($headers);
        $this->applyServerConfig($serviceId, $serverConfig);

        return [
            'success' => true,
            'service_id' => $serviceId,
            'headers_configured' => count($headers)
        ];
    }

    /**
     * Generate Nginx security config
     */
    public function generateNginxConfig(int $serviceId): string {
        $service = $this->db->select(
            "SELECT security_headers FROM mod_service_security WHERE service_id = ?",
            [$serviceId]
        )[0];

        if (!$service) {
            throw new \Exception("Service not found");
        }

        $headers = json_decode($service->security_headers, true);

        $config = "# Security Headers\n";

        // X-Frame-Options
        $config .= "add_header X-Frame-Options \"{$headers['x_frame_options']}\" always;\n";

        // X-Content-Type-Options
        $config .= "add_header X-Content-Type-Options \"{$headers['x_content_type_options']}\" always;\n";

        // X-XSS-Protection
        $config .= "add_header X-XSS-Protection \"{$headers['x_xss_protection']}\" always;\n";

        // Referrer-Policy
        $config .= "add_header Referrer-Policy \"{$headers['referrer_policy']}\" always;\n";

        // Permissions-Policy
        $config .= "add_header Permissions-Policy \"{$headers['permissions_policy']}\" always;\n";

        // HSTS
        $config .= "add_header Strict-Transport-Security \"max-age=31536000; includeSubDomains\" always;\n";

        // CSP
        if ($headers['csp_enabled'] && !empty($headers['csp_config'])) {
            $cspHeader = $this->buildCSPHeader($headers['csp_config']);
            $config .= "add_header Content-Security-Policy \"{$cspHeader}\" always;\n";
        }

        return $config;
    }

    /**
     * Build Content Security Policy header
     */
    public function buildCSPHeader(array $cspConfig): string {
        $directives = [];

        // Default source
        $directives[] = "default-src 'self'";

        // Script sources
        if (!empty($cspConfig['script_src'])) {
            $scripts = implode(' ', $cspConfig['script_src']);
            $directives[] = "script-src {$scripts}";
        } else {
            $directives[] = "script-src 'self' 'unsafe-inline'";
        }

        // Style sources
        if (!empty($cspConfig['style_src'])) {
            $styles = implode(' ', $cspConfig['style_src']);
            $directives[] = "style-src {$styles}";
        } else {
            $directives[] = "style-src 'self' 'unsafe-inline'";
        }

        // Image sources
        $directives[] = "img-src 'self' data: https:";

        // Font sources
        $directives[] = "font-src 'self'";

        // Connect sources
        if (!empty($cspConfig['connect_src'])) {
            $directives[] = "connect-src {$cspConfig['connect_src']}";
        } else {
            $directives[] = "connect-src 'self'";
        }

        // Frame sources
        $directives[] = "frame-ancestors 'self'";

        // Object sources
        $directives[] = "object-src 'none'";

        // Base URI
        $directives[] = "base-uri 'self'";

        // Form action
        $directives[] = "form-action 'self'";

        return implode('; ', $directives);
    }

    /**
     * Get security headers report
     */
    public function getSecurityReport(int $serviceId): array {
        $service = $this->db->select(
            "SELECT * FROM mod_service_security WHERE service_id = ?",
            [$serviceId]
        )[0];

        if (!$service) {
            return ['service_id' => $serviceId, 'status' => 'not_configured'];
        }

        $headers = json_decode($service->security_headers, true);

        return [
            'service_id' => $serviceId,
            'security_enabled' => (bool) $service->security_enabled,
            'headers' => [
                'strict_transport_security' => [
                    'enabled' => $service->hsts_enabled ?? false,
                    'value' => $service->hsts_config ? json_decode($service->hsts_config, true)['max_age'] ?? null : null
                ],
                'x_frame_options' => $headers['x_frame_options'] ?? null,
                'x_content_type_options' => $headers['x_content_type_options'] ?? null,
                'csp_enabled' => $headers['csp_enabled'] ?? false
            ],
            'pinning' => [
                'enabled' => (bool) $service->pinning_enabled
            ]
        ];
    }

    /**
     * Validate security headers configuration
     */
    public function validateConfiguration(int $serviceId): array {
        $issues = [];

        $service = $this->db->select(
            "SELECT * FROM mod_service_security WHERE service_id = ?",
            [$serviceId]
        )[0];

        if (!$service || !$service->security_enabled) {
            return ['valid' => false, 'issues' => ['Security headers not enabled']];
        }

        $headers = json_decode($service->security_headers, true);

        // Check X-Frame-Options
        if (empty($headers['x_frame_options'])) {
            $issues[] = 'X-Frame-Options not configured';
        }

        // Check CSP for unsafe-inline
        if ($headers['csp_enabled'] && !empty($headers['csp_config']['script_src'])) {
            if (in_array("'unsafe-inline'", $headers['csp_config']['script_src'])) {
                $issues[] = 'CSP contains unsafe-inline which reduces security';
            }
        }

        // Check HSTS
        if (!$service->hsts_enabled) {
            $issues[] = 'HSTS not enabled - recommended for SSL sites';
        }

        return [
            'valid' => empty($issues),
            'issues' => $issues
        ];
    }
}
```

## Security Headers Reference

| Header | Purpose | Recommended Value |
|--------|---------|-------------------|
| Strict-Transport-Security | Force HTTPS | max-age=31536000; includeSubDomains |
| X-Frame-Options | Prevent clickjacking | SAMEORIGIN |
| X-Content-Type-Options | Prevent MIME sniffing | nosniff |
| X-XSS-Protection | XSS filter (legacy) | 1; mode=block |
| Referrer-Policy | Control referrer info | strict-origin-when-cross-origin |
| Permissions-Policy | Control browser features | geolocation=(), camera=() |
| Content-Security-Policy | Prevent XSS/injection | custom rules |

## Nginx Configuration Template
```nginx
server {
    listen 443 ssl;
    server_name example.com;

    # Security Headers
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;
    add_header Permissions-Policy "geolocation=(), microphone=(), camera=()" always;
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;

    # CSP
    add_header Content-Security-Policy "default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline';" always;
}
```

## Best Practices

1. **Test in Development**: Validate headers before production
2. **Gradual CSP**: Start with report-only mode
3. **Monitor Violations**: Use CSP report-uri
4. **Keep Updated**: Review headers regularly
5. **Balance Security/Functionality**: Some directives may break features

## Related Skills

- whmcs-csp-configuration
- whmcs-certificate-pinning
- whmcs-hsts-preload
- whmcs-subresource-integrity