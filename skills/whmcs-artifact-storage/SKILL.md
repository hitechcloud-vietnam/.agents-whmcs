---
name: whmcs-artifact-storage
description: Artifact management for WHMCS
category: Automation & DevOps
version: 1.0.0
---

# WHMCS Artifact Storage Skill

## Overview
This skill provides patterns for managing build artifacts in WHMCS.

## Implementation Patterns

### Artifact Manager
```php
<?php
/**
 * WHMCS Artifact Storage
 * Manages build artifacts
 */

namespace WHMCS\Module\DevOps\Artifacts;

class ArtifactManager {
    private $db;

    public function __construct() {
        $this->db = \WHMCS\Database\Capsule::connection();
    }

    /**
     * Store artifact
     */
    public function storeArtifact(array $params): array {
        $artifactId = 'art_' . bin2hex(random_bytes(8));

        $this->db->insert('mod_artifacts', [
            'id' => $artifactId,
            'build_id' => $params['build_id'],
            'name' => $params['name'],
            'path' => $params['path'],
            'size' => filesize($params['path']),
            'checksum' => hash_file('sha256', $params['path']),
            'created_at' => date('Y-m-d H:i:s')
        ]);

        return [
            'success' => true,
            'artifact_id' => $artifactId,
            'checksum' => hash_file('sha256', $params['path'])
        ];
    }

    /**
     * List build artifacts
     */
    public function listBuildArtifacts(string $buildId): array {
        return $this->db->select(
            "SELECT * FROM mod_artifacts WHERE build_id = ?",
            [$buildId]
        );
    }

    /**
     * Cleanup old artifacts
     */
    public function cleanup(int $retentionDays = 30): int {
        $cutoff = date('Y-m-d H:i:s', strtotime("-{$retentionDays} days"));

        $deleted = $this->db->delete('mod_artifacts', [
            'created_at < ?' => $cutoff
        ]);

        return $deleted;
    }
}
```

## Database Schema
```sql
CREATE TABLE `mod_artifacts` (
  `id` VARCHAR(50) PRIMARY KEY,
  `build_id` VARCHAR(100) NOT NULL,
  `name` VARCHAR(255) NOT NULL,
  `path` VARCHAR(500) NOT NULL,
  `size` BIGINT NOT NULL,
  `checksum` VARCHAR(64) NOT NULL,
  `created_at' DATETIME NOT NULL,
  INDEX `idx_build_id` (`build_id`)
);
```

## Best Practices

1. **Naming Convention**: Consistent artifact naming
2. **Compression**: Compress large artifacts
3. **Retention Policies**: Define retention periods
4. **Metadata**: Store artifact metadata
5. **Integrity**: Verify artifact checksums

## Related Skills

- whmcs-cicd-integration
- whmcs-github-actions
- whmcs-gitlab-ci
- whmcs-jenkins-pipeline