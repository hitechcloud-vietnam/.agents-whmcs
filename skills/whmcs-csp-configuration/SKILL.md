---
name: whmcs-csp-configuration
description: Content Security Policy for WHMCS
category: SSL & Security
version: 1.0.0
---

# WHMCS Content Security Policy Configuration Skill

## Overview
This skill provides patterns and implementations for configuring Content Security Policy (CSP) in WHMCS, including directive management, report handling, and policy testing.

## Implementation Patterns

### CSP Configuration Manager
```php
<?php
/**
 * WHMCS Content Security Policy Configuration
 * Manages CSP headers and enforcement
 */

namespace WHMCS\Module\Server\Security;

class CSPManager {
    private $db;

    public function __construct() {
        $this->db = \WHMCS\Database\Capsule::connection();
    }

    /**
     * Create CSP policy
     */
    public function createPolicy(array $params): array {
        $policyId = 'csp_' . bin2hex(random_bytes(12));

        $policy = [
            'id' => $policyId,
            'service_id' => $params['service_id'],
            'name' => $params['name'],
            'directives' => json_encode($params['directives']),
            'report_only' => $params['report_only'] ?? false,
            'report_uri' => $params['report_uri'] ?? null,
            'enabled' => true,
            'created_at' => date('Y-m-d H:i:s')
        ];

        $this->db->insert('mod_csp_policies', $policy);

        // Generate CSP header
        $cspHeader = $this->generateCSPHeader($params['directives']);

        return [
            'success' => true,
            'policy_id' => $policyId,
            'header' => $cspHeader
        ];
    }

    /**
     * Build CSP header from directives
     */
    public function generateCSPHeader(array $directives): string {
        $parts = [];

        // Default-src
        if (!empty($directives['default_src'])) {
            $parts[] = "default-src " . implode(' ', $directives['default_src']);
        } else {
            $parts[] = "default-src 'none'";
        }

        // Script-src
        if (!empty($directives['script_src'])) {
            $parts[] = "script-src " . implode(' ', $directives['script_src']);
        }

        // Style-src
        if (!empty($directives['style_src'])) {
            $parts[] = "style-src " . implode(' ', $directives['style_src']);
        }

        // Img-src
        if (!empty($directives['img_src'])) {
            $parts[] = "img-src " . implode(' ', $directives['img_src']);
        } else {
            $parts[] = "img-src 'self' data: https:";
        }

        // Font-src
        if (!empty($directives['font_src'])) {
            $parts[] = "font-src " . implode(' ', $directives['font_src']);
        }

        // Connect-src
        if (!empty($directives['connect_src'])) {
            $parts[] = "connect-src " . implode(' ', $directives['connect_src']);
        }

        // Media-src
        if (!empty($directives['media_src'])) {
            $parts[] = "media-src " . implode(' ', $directives['media_src']);
        }

        // Object-src
        $parts[] = "object-src " . ($directives['object_src'] ?? "'none'");

        // Frame-src
        if (!empty($directives['frame_src'])) {
            $parts[] = "frame-src " . implode(' ', $directives['frame_src']);
        }

        // Base-uri
        $parts[] = "base-uri " . ($directives['base_uri'] ?? "'self'");

        // Form-action
        $parts[] = "form-action " . ($directives['form_action'] ?? "'self'");

        // Frame-ancestors
        if (!empty($directives['frame_ancestors'])) {
            $parts[] = "frame-ancestors " . implode(' ', $directives['frame_ancestors']);
        }

        return implode('; ', $parts);
    }

    /**
     * Configure CSP reporting
     */
    public function configureReporting(int $serviceId, string $reportUri): array {
        $this->db->update('mod_service_security', [
            'csp_report_uri' => $reportUri,
            'csp_report_only' => true
        ], ['service_id' => $serviceId]);

        return [
            'success' => true,
            'report_uri' => $reportUri,
            'mode' => 'report-only'
        ];
    }

    /**
     * Analyze CSP violation reports
     */
    public function analyzeViolations(int $serviceId, \DateTime $from, \DateTime $to): array {
        $violations = $this->db->select(
            "SELECT csp_report FROM mod_csp_violations
             WHERE service_id = ? AND created_at BETWEEN ? AND ?",
            [$serviceId, $from->format('Y-m-d H:i:s'), $to->format('Y-m-d H:i:s')]
        );

        $analysis = [
            'total_violations' => count($violations),
            'by_directive' => [],
            'by_domain' => [],
            'recommendations' => []
        ];

        foreach ($violations as $violation) {
            $report = json_decode($violation->csp_report, true);
            $directive = $report['violated-directive'] ?? 'unknown';
            $blockedUri = $report['blocked-uri'] ?? '';

            // Count by directive
            if (!isset($analysis['by_directive'][$directive])) {
                $analysis['by_directive'][$directive] = 0;
            }
            $analysis['by_directive'][$directive]++;

            // Count by domain
            $domain = parse_url($blockedUri, PHP_URL_HOST) ?? 'inline';
            if (!isset($analysis['by_domain'][$domain])) {
                $analysis['by_domain'][$domain] = 0;
            }
            $analysis['by_domain'][$domain]++;
        }

        // Generate recommendations
        arsort($analysis['by_directive']);
        foreach (array_slice($analysis['by_directive'], 0, 3) as $directive => $count) {
            $analysis['recommendations'][] = "Consider adding '{$directive}' sources in CSP";
        }

        return $analysis;
    }

    /**
     * Generate strict CSP for WHMCS
     */
    public function generateWHMCSStrictPolicy(): array {
        $directives = [
            'default_src' => ["'self'"],
            'script_src' => ["'self'", "'unsafe-inline'"],
            'style_src' => ["'self'", "'unsafe-inline'"],
            'img_src' => ["'self'", "data:", "https:"],
            'font_src' => ["'self'", "https://fonts.gstatic.com"],
            'connect_src' => ["'self'", "https://api.whmcs.com"],
            'object_src' => ["'none'"],
            'base_uri' => ["'self'"],
            'form_action' => ["'self'"],
            'frame_ancestors' => ["'self'"],
            'frame_src' => ["'none'"]
        ];

        return [
            'directives' => $directives,
            'header' => $this->generateCSPHeader($directives)
        ];
    }
}
```

## CSP Directives Reference

| Directive | Purpose | Example |
|-----------|---------|---------|
| default-src | Fallback for other directives | 'self' |
| script-src | JavaScript sources | 'self' 'unsafe-inline' |
| style-src | CSS sources | 'self' 'unsafe-inline' |
| img-src | Image sources | 'self' https: data: |
| font-src | Font sources | 'self' https://fonts.gstatic.com |
| connect-src | XHR, WebSocket, etc | 'self' |
| object-src | Plugin sources | 'none' |
| frame-src | iframe sources | 'none' |
| base-uri | Base URL restriction | 'self' |
| form-action | Form submission targets | 'self' |
| frame-ancestors | Who can embed this page | 'self' |
| report-uri | CSP violation reports | /csp-report |

## Report-Only Header for Testing
```
Content-Security-Policy-Report-Only: default-src 'self'; script-src 'self' 'unsafe-inline'; report-uri /csp-reports
```

## Best Practices

1. **Start with Report-Only**: Test policies before enforcement
2. **Whitelist Known Good**: Start strict, add as needed
3. **Use Nonces**: Prefer nonces over unsafe-inline
4. **Monitor Reports**: Review violation reports regularly
5. **Plan for Third-party**: Account for CDN and external services

## Related Skills

- whmcs-security-headers
- whmcs-subresource-integrity
- whmcs-cors-configuration
- whmcs-xss-protection