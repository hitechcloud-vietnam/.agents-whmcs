# WHMCS Module Upgrade Workflow
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Systematic approach to upgrading WHMCS modules.

## Steps

### 1. Version Analysis
```php
function analyzeVersions(): array {
    $current = Capsule::table('mod_{module}_meta')->value('version');
    $latest = getLatestModuleVersion();

    return [
        'current' => $current,
        'latest' => $latest,
        'migrations_needed' => $this->getRequiredMigrations($current, $latest),
    ];
}
```

### 2. Migration Planning
```
Version Path: 1.0.0 → 1.2.0
├── Migration 1.0 → 1.1
│   ├── Add new column
│   └── Update indexes
├── Migration 1.1 → 1.2
│   ├── Create new table
│   ├── Migrate data
│   └── Add new feature
```

### 3. Implementation
```php
function {module}_upgrade(array $vars): void {
    $fromVersion = $vars['version'];

    $migrations = [
        '1.0' => 'migrate_1_0_to_1_1',
        '1.1' => 'migrate_1_1_to_1_2',
        '1.2' => 'migrate_1_2_to_1_3',
    ];

    foreach ($migrations as $version => $function) {
        if (version_compare($fromVersion, $version, '<')) {
            call_user_func($function);
        }
    }

    // Update meta version
    Capsule::table('mod_{module}_meta')->update(['version' => '{MODULE_VERSION}']);
}
```

### 4. Testing
```
□ Test in fresh install
□ Test upgrade from each previous version
□ Test data integrity
□ Test feature functionality
□ Performance regression test
```

### 5. Rollback Plan
```php
function rollback_1_2_to_1_1(): void {
    // Revert new table
    Capsule::schema()->dropIfExists('mod_{module}_new_table');

    // Restore data
    Capsule::table('mod_{module}_data')
        ->where('new_format', '!=', '')
        ->update(['old_format' => Capsule::raw('new_format')]);
}
```

## Output

Complete upgrade package:
- Migration scripts
- Data validation
- Version detection
- Rollback capability
