# WHMCS Module Migration Framework Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Guide for building migration frameworks between systems.

## When to Use

- Migrating data from legacy systems
- Importing from third-party platforms
- Batch data imports

## Migration Patterns

```php
<?php
class MigrationFramework {
    private string $source;
    private int $batchSize = 100;

    public function migrate(): array {
        $results = [
            'total' => 0,
            'success' => 0,
            'failed' => 0,
            'errors' => [],
        ];

        $items = $this->fetchSourceData();

        foreach ($items as $item) {
            $results['total']++;

            try {
                $this->migrateItem($item);
                $results['success']++;
            } catch (\Exception $e) {
                $results['failed']++;
                $results['errors'][] = [
                    'source_id' => $item['id'],
                    'error' => $e->getMessage(),
                ];
            }

            // Batch progress
            if ($results['total'] % $this->batchSize === 0) {
                $this->logProgress($results);
            }
        }

        return $results;
    }

    private function migrateItem(array $item): void {
        // Transform data
        $transformed = $this->transform($item);

        // Check for duplicates
        if ($this->exists($transformed)) {
            $this->update($transformed);
        } else {
            $this->insert($transformed);
        }

        // Log migration
        Capsule::table('mod_migration_log')->insert([
            'source_id' => $item['id'],
            'target_id' => $transformed['id'],
            'migrated_at' => date('Y-m-d H:i:s'),
        ]);
    }

    private function transform(array $item): array {
        // Map source fields to target fields
        return [
            'name' => $item['title'],
            'email' => $item['contact_email'],
            'status' => $this->mapStatus($item['status']),
        ];
    }

    private function mapStatus(string $status): string {
        $mapping = [
            'active' => 'Active',
            'inactive' => 'Suspended',
            'deleted' => 'Terminated',
        ];
        return $mapping[$status] ?? 'Pending';
    }
}
```

---

**Related Skills:**
- whmcs-migration-guide
- whmcs-database-design
- whmcs-api-integration
