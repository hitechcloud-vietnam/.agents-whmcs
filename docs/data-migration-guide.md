# WHMCS Data Migration Guide
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Data migration patterns for WHMCS module upgrades.

## Migration Structure

```php
// Migration class pattern
class Migration_1_0_to_1_1 {
    public function up(): void {
        // Add new columns
        Capsule::schema()->table('mod_{module}_data', function($t) {
            $t->string('new_field')->nullable();
        });

        // Create new tables
        Capsule::schema()->create('mod_{module}_logs', function($t) {
            $t->increments('id');
            $t->text('message');
        });

        // Migrate existing data
        $data = Capsule::table('mod_{module}_old')->get();
        foreach ($data as $row) {
            Capsule::table('mod_{module}_data')->insert([
                'item_id' => $row->id,
                'old_data' => $row->config,
            ]);
        }
    }

    public function down(): void {
        // Rollback changes
        Capsule::schema()->table('mod_{module}_data', function($t) {
            $t->dropColumn('new_field');
        });
    }
}
```

## Data Transformation

### Example: Transform Configuration
```php
public function migrateConfig(): void {
    $oldConfigs = Capsule::table('mod_{module}_settings')
        ->where('key', 'config_v1')
        ->get();

    foreach ($oldConfigs as $old) {
        $config = json_decode($old->value, true);

        Capsule::table('mod_{module}_settings_v2')->insert([
            'service_id' => $old->service_id,
            'api_version' => $config['api_version'] ?? 'v2',
            'endpoint' => $config['api_endpoint'] ?? '',
            'timeout' => $config['timeout'] ?? 30,
            'retry_count' => $config['retry'] ?? 3,
        ]);
    }
}
```

### Example: Data Cleanup
```php
public function cleanupData(): void {
    // Remove deprecated fields
    Capsule::table('mod_{module}_data')
        ->whereNull('deprecated_field')
        ->update(['deprecated_field' => '']);

    // Rename columns
    if (Capsule::schema()->hasColumn('mod_{module}_data', 'old_name')) {
        Capsule::statement('ALTER TABLE mod_{module}_data CHANGE old_name new_name VARCHAR(255)');
    }
}
```

## Testing Migrations

```php
function testMigration(): void {
    // Backup current data
    $backup = Capsule::table('mod_{module}_data')->get();

    // Run migration
    $migration = new Migration_1_0_to_1_1();
    $migration->up();

    // Verify new structure
    $columns = Capsule::schema()->getColumnListing('mod_{module}_data');
    $this->assertContains('new_field', $columns);

    // Rollback
    $migration->down();

    // Verify data integrity
    $current = Capsule::table('mod_{module}_data')->get();
    $this->assertEquals($backup, $current);
}
```

---

**Related Skills:**
- whmcs-migration-guide
- whmcs-database-design
