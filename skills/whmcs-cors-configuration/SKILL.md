---
name: whmcs-cors-configuration
description: CORS setup for WHMCS
category: SSL & Security
version: 1.0.0
---

# WHMCS CORS Configuration Skill

## Overview
This skill provides patterns and implementations for configuring Cross-Origin Resource Sharing (CORS) in WHMCS for secure API and resource sharing.

## Implementation Patterns

### CORS Configuration Manager
```php
<?php
/**
 * WHMCS CORS Configuration
 * Manages CORS headers and policies
 */

namespace WHMCS\Module\Server\Security;

class CORSManager {
    private $db;

    public function __construct() {
        $this->db = \WHMCS\Database\Capsule::connection();
    }

    /**
     * Create CORS policy
     */
    public function createPolicy(array $params): array {
        $policyId = 'cors_' . bin2hex(random_bytes(8));

        $policy = [
            'id' => $policyId,
            'service_id' => $params['service_id'],
            'name' => $params['name'],
            'allowed_origins' => json_encode($params['allowed_origins']),
            'allowed_methods' => json_encode($params['allowed_methods'] ?? ['GET', 'POST', 'PUT', 'DELETE']),
            'allowed_headers' => json_encode($params['allowed_headers'] ?? ['Content-Type', 'Authorization']),
            'exposed_headers' => json_encode($params['exposed_headers'] ?? []),
            'max_age' => $params['max_age'] ?? 86400,
            'allow_credentials' => $params['allow_credentials'] ?? true,
            'enabled' => true,
            'created_at' => date('Y-m-d H:i:s')
        ];

        $this->db->insert('mod_cors_policies', $policy);

        return [
            'success' => true,
            'policy_id' => $policyId
        ];
    }

    /**
     * Generate CORS headers
     */
    public function generateHeaders(string $origin, array $policy): array {
        $allowedOrigins = json_decode($policy['allowed_origins'], true);

        // Check if origin is allowed
        if (!$this->isOriginAllowed($origin, $allowedOrigins)) {
            return ['allowed' => false];
        }

        $headers = [
            'Access-Control-Allow-Origin' => $origin,
            'Access-Control-Allow-Methods' => implode(', ', json_decode($policy['allowed_methods'], true)),
            'Access-Control-Allow-Headers' => implode(', ', json_decode($policy['allowed_headers'], true)),
            'Access-Control-Max-Age' => $policy['max_age']
        ];

        if ($policy['allow_credentials']) {
            $headers['Access-Control-Allow-Credentials'] = 'true';
        }

        $exposedHeaders = json_decode($policy['exposed_headers'], true);
        if (!empty($exposedHeaders)) {
            $headers['Access-Control-Expose-Headers'] = implode(', ', $exposedHeaders);
        }

        return [
            'allowed' => true,
            'headers' => $headers
        ];
    }

    /**
     * Handle preflight request
     */
    public function handlePreflight(array $policy, string $origin, string $method): array {
        $allowedMethods = json_decode($policy['allowed_methods'], true);

        if (!in_array($method, $allowedMethods)) {
            return ['allowed' => false, 'status' => 405];
        }

        $headers = $this->generateHeaders($origin, $policy);

        return array_merge($headers, [
            'status' => 204,
            'Access-Control-Allow-Credentials' => $policy['allow_credentials'] ? 'true' : 'false'
        ]);
    }

    /**
     * Validate CORS configuration
     */
    public function validateConfiguration(array $policy): array {
        $issues = [];

        $allowedOrigins = json_decode($policy['allowed_origins'], true);
        $allowedMethods = json_decode($policy['allowed_methods'], true);

        // Check for wildcard in production
        if (in_array('*', $allowedOrigins) && $policy['allow_credentials']) {
            $issues[] = "Cannot use wildcard origin with credentials";
        }

        // Check for overly broad configuration
        if (count($allowedOrigins) > 50) {
            $issues[] = "Too many allowed origins - consider using dynamic validation";
        }

        // Check methods
        $validMethods = ['GET', 'POST', 'PUT', 'DELETE', 'PATCH', 'OPTIONS', 'HEAD'];
        foreach ($allowedMethods as $method) {
            if (!in_array(strtoupper($method), $validMethods)) {
                $issues[] = "Invalid HTTP method: {$method}";
            }
        }

        return [
            'valid' => empty($issues),
            'issues' => $issues
        ];
    }

    // Private helper methods

    private function isOriginAllowed(string $origin, array $allowedOrigins): bool {
        // Handle wildcard
        if (in_array('*', $allowedOrigins)) {
            return true;
        }

        // Exact match
        if (in_array($origin, $allowedOrigins)) {
            return true;
        }

        // Wildcard subdomain matching
        foreach ($allowedOrigins as $allowed) {
            if (strpos($allowed, '*.') === 0) {
                $domain = substr($allowed, 2);
                $originHost = parse_url($origin, PHP_URL_HOST);
                if ($originHost && preg_match("/\.?" . preg_quote($domain, '/') . "$/", $originHost)) {
                    return true;
                }
            }
        }

        return false;
    }
}
```

## Database Schema
```sql
CREATE TABLE `mod_cors_policies` (
  `id` VARCHAR(50) PRIMARY KEY,
  `service_id` INT,
  `name` VARCHAR(255) NOT NULL,
  `allowed_origins` TEXT NOT NULL,
  `allowed_methods` TEXT,
  `allowed_headers` TEXT,
  `exposed_headers` TEXT,
  `max_age` INT DEFAULT 86400,
  `allow_credentials` TINYINT(1) DEFAULT 1,
  `enabled` TINYINT(1) DEFAULT 1,
  `created_at' DATETIME NOT NULL
);
```

## CORS Header Reference

| Header | Description | Example |
|--------|-------------|---------|
| Access-Control-Allow-Origin | Allowed origin(s) | https://app.example.com |
| Access-Control-Allow-Methods | Allowed HTTP methods | GET, POST, PUT |
| Access-Control-Allow-Headers | Allowed request headers | Content-Type, Authorization |
| Access-Control-Expose-Headers | Headers exposed to JS | X-Custom-Header |
| Access-Control-Max-Age | Preflight cache duration | 86400 |
| Access-Control-Allow-Credentials | Allow cookies | true |

## Nginx CORS Configuration
```nginx
location /api/ {
    # Preflight
    if ($request_method = 'OPTIONS') {
        add_header 'Access-Control-Allow-Origin' '$http_origin';
        add_header 'Access-Control-Allow-Methods' 'GET, POST, PUT, DELETE';
        add_header 'Access-Control-Allow-Headers' 'Content-Type, Authorization';
        add_header 'Access-Control-Max-Age' 86400;
        add_header 'Access-Control-Allow-Credentials' 'true';
        add_header 'Content-Length' 0;
        add_header 'Content-Type' 'text/plain; charset=UTF-8';
        return 204;
    }

    # Actual request
    add_header 'Access-Control-Allow-Origin' '$http_origin';
    add_header 'Access-Control-Allow-Credentials' 'true';
}
```

## Best Practices

1. **Specific Origins**: Avoid wildcard in production
2. **Minimal Methods**: Only allow necessary HTTP methods
3. **Secure Credentials**: Don't use wildcard with credentials
4. **Cache Preflight**: Set reasonable max-age for preflight
5. **Monitor Requests**: Track CORS violations

## Related Skills

- whmcs-security-headers
- whmcs-api-authentication
- whmcs-api-middleware
- whmcs-webhook-handler