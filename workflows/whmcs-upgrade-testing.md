# WHMCS Upgrade Testing Workflow

## Overview
This workflow provides comprehensive guidance for testing WHMCS module upgrades and version migrations.

## Prerequisites
- WHMCS installation (v8.0+)
- Previous module version
- Upgrade scripts

## Step-by-Step Guide

### Step 1: Create Upgrade Test Suite
```php
// tests/UpgradeTest.php
<?php
namespace WHMCS\Tests;

use PHPUnit\Framework\TestCase;

class UpgradeTest extends TestCase
{
    public function testUpgradeFromV1ToV2()
    {
        // Simulate old version data
        $oldData = [
            'setting1' => 'old_value',
            'setting2' => 'another_value',
        ];

        // Run upgrade
        $upgrader = new \WHMCS\Module\YourModule\Upgrader('v1.0.0', 'v2.0.0');
        $newData = $upgrader->upgrade($oldData);

        // Verify data migration
        $this->assertArrayHasKey('setting1', $newData);
        $this->assertEquals('old_value', $newData['setting1']);
    }

    public function testUpgradeMigrationsRun()
    {
        $upgrader = new \WHMCS\Module\YourModule\Upgrader();
        $migrations = $upgrader->getPendingMigrations();

        // Run all pending migrations
        foreach ($migrations as $migration) {
            $migration->up();
        }

        // Verify migrations completed
        $this->assertEmpty($upgrader->getPendingMigrations());
    }

    public function testUpgradeBackupCreated()
    {
        $upgrader = new \WHMCS\Module\YourModule\Upgrader();

        // Create backup before upgrade
        $backupPath = $upgrader->createBackup();

        $this->assertFileExists($backupPath);

        // Cleanup
        unlink($backupPath);
    }

    public function testUpgradeVersionStored()
    {
        $upgrader = new \WHMCS\Module\YourModule\Upgrader('v1.0.0', 'v2.0.0');
        $upgrader->upgrade([]);

        $currentVersion = \WHMCS\Module\YourModule\Settings::get('version');
        $this->assertEquals('v2.0.0', $currentVersion);
    }
}
```

### Step 2: Run Upgrade Tests
```bash
# Run upgrade tests
./vendor/bin/phpunit tests/UpgradeTest.php

# Run with verbose output
./vendor/bin/phpunit tests/UpgradeTest.php --testdox
```

## Upgrade Testing Checklist

### Version Migration
- [ ] Version stored correctly
- [ ] All migrations run
- [ ] No duplicate migrations

### Data Migration
- [ ] Old data preserved
- [ ] Data transformed correctly
- [ ] No data loss

### Backup
- [ ] Backup created before upgrade
- [ ] Backup restorable
- [ ] Backup cleanup scheduled
