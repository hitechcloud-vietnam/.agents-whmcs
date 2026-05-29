# WHMCS Migration Testing Workflow

## Overview
This workflow provides comprehensive guidance for testing data migrations in WHMCS.

## Prerequisites
- WHMCS installation (v8.0+)
- Migration scripts
- Test data

## Step-by-Step Guide

### Step 1: Create Migration Test Suite
```php
// tests/MigrationTest.php
<?php
namespace WHMCS\Tests;

use PHPUnit\Framework\TestCase;

class MigrationTest extends TestCase
{
    protected function setUp(): void
    {
        parent::setUp();
        $this->db = \WHMCS\Database\Capsule::connection();
        $this->db->beginTransaction();
    }

    protected function tearDown(): void
    {
        $this->db->rollBack();
        parent::tearDown();
    }

    public function testCreateTableMigration()
    {
        $tableName = 'test_migration_' . time();

        \WHMCS\Database\Capsule::schema()->create($tableName, function ($table) {
            $table->increments('id');
            $table->string('name');
            $table->timestamps();
        });

        $this->assertTrue(\WHMCS\Database\Capsule::schema()->hasTable($tableName));
    }

    public function testAddColumnMigration()
    {
        $tableName = 'test_add_column';
        \WHMCS\Database\Capsule::schema()->create($tableName, function ($table) {
            $table->increments('id');
            $table->string('name');
        });

        \WHMCS\Database\Capsule::schema()->table($tableName, function ($table) {
            $table->text('description')->nullable()->after('name');
        });

        $columns = \WHMCS\Database\Capsule::connection()
            ->getSchemaBuilder()
            ->getColumnListing($tableName);

        $this->assertContains('description', $columns);
    }

    public function testDataMigration()
    {
        // Create source table with old format
        \WHMCS\Database\Capsule::schema()->create('test_source', function ($table) {
            $table->increments('id');
            $table->string('old_field');
        });

        \WHMCS\Database\Capsule::table('test_source')->insert([
            ['old_field' => 'value1'],
            ['old_field' => 'value2'],
        ]);

        // Run data migration
        $migrator = new \WHMCS\Module\YourModule\DataMigrator();
        $count = $migrator->migrateOldToNew();

        $this->assertEquals(2, $count);
    }

    public function testRollbackMigration()
    {
        $tableName = 'test_rollback_migration';

        // Create
        \WHMCS\Database\Capsule::schema()->create($tableName, function ($table) {
            $table->increments('id');
        });

        // Drop (rollback)
        \WHMCS\Database\Capsule::schema()->dropIfExists($tableName);

        $this->assertFalse(\WHMCS\Database\Capsule::schema()->hasTable($tableName));
    }
}
```

### Step 2: Run Migration Tests
```bash
# Run migration tests
./vendor/bin/phpunit tests/MigrationTest.php

# Run with verbose output
./vendor/bin/phpunit tests/MigrationTest.php --testdox
```

## Migration Testing Checklist

### Schema Changes
- [ ] Tables created correctly
- [ ] Columns added properly
- [ ] Indexes created
- [ ] Foreign keys set

### Data
- [ ] Data transferred correctly
- [ ] No data loss
- [ ] Data types preserved

### Rollback
- [ ] Rollback works
- [ ] No partial state
- [ ] Logs maintained
