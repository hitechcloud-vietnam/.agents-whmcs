# WHMCS Rollback Testing Workflow

## Overview
This workflow provides comprehensive guidance for testing rollback scenarios in WHMCS modules and migrations.

## Prerequisites
- WHMCS installation (v8.0+)
- Database access
- Transaction handling

## Step-by-Step Guide

### Step 1: Create Rollback Test Suite
```php
// tests/RollbackTest.php
<?php
namespace WHMCS\Tests;

use PHPUnit\Framework\TestCase;

class RollbackTest extends TestCase
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

    public function testMigrationRollback()
    {
        $tableName = 'test_rollback_' . time();

        // Create table (up migration)
        \WHMCS\Database\Capsule::schema()->create($tableName, function ($table) {
            $table->increments('id');
            $table->string('name');
        });

        $this->assertTrue(\WHMCS\Database\Capsule::schema()->hasTable($tableName));

        // Rollback (down migration)
        \WHMCS\Database\Capsule::schema()->dropIfExists($tableName);

        $this->assertFalse(\WHMCS\Database\Capsule::schema()->hasTable($tableName));
    }

    public function testTransactionRollback()
    {
        $initialCount = \WHMCS\Database\Capsule::table('tblclients')->count();

        // Perform insert
        \WHMCS\Database\Capsule::table('test_rollback_temp')->insert([
            'name' => 'Rollback Test',
            'created_at' => date('Y-m-d H:i:s'),
        ]);

        // Transaction will be rolled back in tearDown
        $this->assertEquals(
            $initialCount,
            \WHMCS\Database\Capsule::table('tblclients')->count()
        );
    }

    public function testFailedOperationRollback()
    {
        $this->expectException(\Exception::class);

        try {
            \WHMCS\Database\Capsule::transaction(function () {
                // Insert record
                \WHMCS\Database\Capsule::table('test_rollback_temp')->insert([
                    'name' => 'Should Rollback',
                ]);

                // Force an error
                throw new \Exception('Simulated failure');
            });
        } catch (\Exception $e) {
            // Verify record was rolled back
            $count = \WHMCS\Database\Capsule::table('test_rollback_temp')
                ->where('name', 'Should Rollback')
                ->count();

            $this->assertEquals(0, $count);
            throw $e;
        }
    }

    public function testPartialRollback()
    {
        $operations = [];

        // Operation 1: Insert
        \WHMCS\Database\Capsule::table('test_rollback_temp')->insert([
            'name' => 'First',
            'created_at' => date('Y-m-d H:i:s'),
        ]);
        $operations[] = 'First';

        // Operation 2: Insert
        \WHMCS\Database\Capsule::table('test_rollback_temp')->insert([
            'name' => 'Second',
            'created_at' => date('Y-m-d H:i:s'),
        ]);
        $operations[] = 'Second';

        // Rollback both
        foreach ($operations as $op) {
            \WHMCS\Database\Capsule::table('test_rollback_temp')
                ->where('name', $op)
                ->delete();
        }

        $count = \WHMCS\Database\Capsule::table('test_rollback_temp')
            ->whereIn('name', $operations)
            ->count();

        $this->assertEquals(0, $count);
    }
}
```

### Step 2: Run Rollback Tests
```bash
# Run rollback tests
./vendor/bin/phpunit tests/RollbackTest.php

# Run with verbose output
./vendor/bin/phpunit tests/RollbackTest.php --testdox
```

## Rollback Testing Checklist

### Database
- [ ] Migrations have rollback scripts
- [ ] Transactions are atomic
- [ ] Failed operations rollback completely

### Files
- [ ] File uploads can be undone
- [ ] Directory creation is reversible
- [ ] Temporary files are cleaned up

### State
- [ ] State changes are tracked
- [ ] Previous state can be restored
- [ ] Cache is invalidated on rollback
