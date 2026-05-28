# WHMCS Upgrade Handler Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Guide for implementing module upgrade handlers and migrations.

## When to Use

- Handling module version upgrades
- Migrating database schemas
- Managing breaking changes

## Upgrade Patterns

```php
function {module}_upgrade(array $vars): void {
    $fromVersion = $vars['version'];

    // Migration: 1.0 -> 1.1
    if (version_compare($fromVersion, '1.1', '<')) {
        migrateTo1_1();
    }

    // Migration: 1.1 -> 1.2
    if (version_compare($fromVersion, '1.2', '<')) {
        migrateTo1_2();
    }

    // Migration: 1.2 -> 2.0
    if (version_compare($fromVersion, '2.0', '<')) {
        migrateTo2_0();
    }
}

function migrateTo1_1(): void {
    // Add new column
    Capsule::schema()->table('mod_{module}_data', function($t) {
        if (!Capsule::schema()->hasColumn('mod_{module}_data', 'metadata')) {
            $t->text('metadata')->nullable();
        }
    });

    // Create new index
    Capsule::schema()->table('mod_{module}_logs', function($t) {
        $t->index(['level', 'created_at'], 'idx_level_date');
    });
}

function migrateTo1_2(): void {
    // Create new table
    if (!Capsule::schema()->hasTable('mod_{module}_cache')) {
        Capsule::schema()->create('mod_{module}_cache', function($t) {
            $t->increments('id');
            $t->string('key', 100)->unique();
            $t->text('value');
            $t->timestamp('expires_at')->nullable();
        });
    }
}

function migrateTo2_0(): void {
    // Major restructuring
    // Move data to new format
    $oldData = Capsule::table('mod_{module}_old_data')->get();

    foreach ($oldData as $row) {
        Capsule::table('mod_{module}_new_data')->insert([
            'new_format' => convertToNewFormat($row),
            'created_at' => date('Y-m-d H:i:s'),
        ]);
    }

    // Drop old table
    Capsule::schema()->dropIfExists('mod_{module}_old_data');
}
```

---

**Related Skills:**
- whmcs-database-design
- whmcs-deployment
- whmcs-migration-guide