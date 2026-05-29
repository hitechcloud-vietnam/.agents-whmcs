---
name: whmcs-subresource-integrity
description: SRI implementation for WHMCS
category: SSL & Security
version: 1.0.0
---

# WHMCS Subresource Integrity (SRI) Skill

## Overview
This skill provides patterns and implementations for implementing Subresource Integrity (SRI) in WHMCS to verify the integrity of external resources like CDN scripts.

## Implementation Patterns

### SRI Manager
```php
<?php
/**
 * WHMCS Subresource Integrity Implementation
 * Manages SRI hashes for external resources
 */

namespace WHMCS\Module\Server\Security;

class SRIManager {
    private $db;

    public function __construct() {
        $this->db = \WHMCS\Database\Capsule::connection();
    }

    /**
     * Add resource with SRI
     */
    public function addResource(array $params): array {
        $resourceId = 'sri_' . bin2hex(random_bytes(8));

        // Calculate integrity hashes
        $hashes = $this->calculateHashes($params['url'], $params['algorithm'] ?? 'sha384');

        $resource = [
            'id' => $resourceId,
            'service_id' => $params['service_id'],
            'url' => $params['url'],
            'integrity' => $hashes['integrity'],
            'algorithm' => $params['algorithm'] ?? 'sha384',
            'crossorigin' => $params['crossorigin'] ?? 'anonymous',
            'created_at' => date('Y-m-d H:i:s')
        ];

        $this->db->insert('mod_sri_resources', $resource);

        return [
            'success' => true,
            'resource_id' => $resourceId,
            'integrity' => $hashes['integrity'],
            'hashes' => $hashes
        ];
    }

    /**
     * Calculate SRI hashes for URL
     */
    public function calculateHashes(string $url, string $algorithm = 'sha384'): array {
        // Fetch content
        $content = $this->fetchResource($url);

        if (!$content) {
            throw new \Exception("Failed to fetch resource: {$url}");
        }

        // Calculate hash based on algorithm
        $hash = match($algorithm) {
            'sha256' => hash('sha256', $content, true),
            'sha384' => hash('sha384', $content, true),
            'sha512' => hash('sha512', $content, true),
            default => hash('sha384', $content, true)
        };

        $base64Hash = base64_encode($hash);

        return [
            'integrity' => "{$algorithm}-{$base64Hash}",
            'hash' => $base64Hash,
            'algorithm' => $algorithm,
            'url' => $url
        ];
    }

    /**
     * Generate HTML tag with SRI
     */
    public function generateTag(array $resource): string {
        $crossorigin = !empty($resource['crossorigin'])
            ? " crossorigin=\"{$resource['crossorigin']}\""
            : "";

        return "<script src=\"{$resource['url']}\" integrity=\"{$resource['integrity']}\"{$crossorigin}></script>";
    }

    /**
     * Batch update SRI hashes
     */
    public function updateHashes(int $serviceId): array {
        $resources = $this->db->select(
            "SELECT * FROM mod_sri_resources WHERE service_id = ?",
            [$serviceId]
        );

        $updated = [];

        foreach ($resources as $resource) {
            try {
                $hashes = $this->calculateHashes($resource->url, $resource->algorithm);

                $this->db->update('mod_sri_resources', [
                    'integrity' => $hashes['integrity'],
                    'last_verified_at' => date('Y-m-d H:i:s')
                ], ['id' => $resource->id]);

                $updated[] = [
                    'url' => $resource->url,
                    'status' => 'updated',
                    'integrity' => $hashes['integrity']
                ];
            } catch (\Exception $e) {
                $updated[] = [
                    'url' => $resource->url,
                    'status' => 'failed',
                    'error' => $e->getMessage()
                ];
            }
        }

        return [
            'total' => count($resources),
            'updated' => count(array_filter($updated, fn($u) => $u['status'] === 'updated')),
            'failed' => count(array_filter($updated, fn($u) => $u['status'] === 'failed')),
            'details' => $updated
        ];
    }

    /**
     * Verify resource integrity
     */
    public function verifyIntegrity(string $resourceId): array {
        $resource = $this->getResource($resourceId);

        if (!$resource) {
            throw new \Exception("Resource not found");
        }

        try {
            $hashes = $this->calculateHashes($resource->url, $resource->algorithm);

            $matches = $hashes['integrity'] === $resource->integrity;

            return [
                'resource_id' => $resourceId,
                'url' => $resource->url,
                'stored_integrity' => $resource->integrity,
                'current_integrity' => $hashes['integrity'],
                'matches' => $matches,
                'verified_at' => date('Y-m-d H:i:s')
            ];
        } catch (\Exception $e) {
            return [
                'resource_id' => $resourceId,
                'url' => $resource->url,
                'matches' => false,
                'error' => $e->getMessage()
            ];
        }
    }

    // Private helper methods

    private function fetchResource(string $url): ?string {
        $ch = curl_init($url);
        curl_setopt_array($ch, [
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_SSL_VERIFYPEER => true,
            CURLOPT_FOLLOWLOCATION => true
        ]);

        $content = curl_exec($ch);
        curl_close($ch);

        return $content ?: null;
    }
}
```

## Database Schema
```sql
CREATE TABLE `mod_sri_resources` (
  `id` VARCHAR(50) PRIMARY KEY,
  `service_id` INT NOT NULL,
  `url` VARCHAR(500) NOT NULL,
  `integrity` VARCHAR(200) NOT NULL,
  `algorithm` VARCHAR(20) DEFAULT 'sha384',
  `crossorigin` VARCHAR(20) DEFAULT 'anonymous',
  `last_verified_at` DATETIME,
  `created_at' DATETIME NOT NULL,
  INDEX `idx_service_id` (`service_id`)
);
```

## SRI HTML Examples

```html
<!-- JavaScript with SRI -->
<script src="https://cdn.example.com/jquery.min.js"
        integrity="sha384-oqVuAfXRKap7fdgcCY5uyukM35KFajEcBfF85nQ=="
        crossorigin="anonymous"></script>

<!-- Stylesheet with SRI -->
<link rel="stylesheet"
      href="https://cdn.example.com/bootstrap.min.css"
      integrity="sha384-9gVQ4dYFwwWSjIDZnLEWnxC2WWF70N3+GA="
      crossorigin="anonymous">
```

## Common CDN SRI Hashes

| Resource | SHA384 Hash |
|----------|-------------|
| jQuery 3.6.0 min | sha384-oqVuAfXRKap7fdgcCY5uyukM35KFajEcBfF85nQ== |
| Bootstrap 5.1.3 min | sha384-9sPfVcVjLAB07gDeai1z35gnRNVof8Cmv0R4L7mL3/3yT7N2J |

## Best Practices

1. **Use SHA-384+**: SHA-256 minimum, SHA-384 recommended
2. **Update Regularly**: Re-calculate hashes when resources change
3. **Cross-Origin**: Set crossorigin for CORS-enabled CDNs
4. **Fallback**: Have fallback options if integrity check fails
5. **Monitor**: Track verification failures

## Related Skills

- whmcs-csp-configuration
- whmcs-security-headers
- whmcs-cors-configuration
- whmcs-resource-optimization