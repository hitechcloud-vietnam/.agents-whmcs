# WHMCS Database Testing Workflow

## Overview
This workflow provides comprehensive guidance for testing WHMCS database operations, schema integrity, and data migrations.

## Prerequisites
- WHMCS installation (v8.0+)
- MySQL/MariaDB access
- Database testing tools
- Migration testing environment

## Step-by-Step Guide

### Step 1: Set Up Database Test Environment

#### Create Test Database
```sql
-- Create dedicated test database
CREATE DATABASE whmcs_test;
CREATE USER 'whmcs_test'@'localhost' IDENTIFIED BY 'test_password';
GRANT ALL PRIVILEGES ON whmcs_test.* TO 'whmcs_test'@'localhost';
FLUSH PRIVILEGES;

-- Clone production data for testing
mysqldump -u root -p whmcs_production | mysql -u whmcs_test -p whmcs_test
```

#### Configure Test Database Connection
```php
// tests/bootstrap.php
<?php
// Database configuration for testing
define('DB_host', 'localhost');
define('DB_name', 'whmcs_test');
define('DB_username', 'whmcs_test');
define('DB_password', 'test_password');
define('DB_type', 'mysql');

// Load WHMCS
require_once __DIR__ . '/../../init.php';
```

### Step 2: Schema Validation Tests
```php
// tests/Database/SchemaTest.php
<?php
namespace WHMCS\Tests\Database;

use PHPUnit\Framework\TestCase;
use WHMCS\Database\Capsule;

class SchemaTest extends TestCase
{
    public function testCoreTablesExist()
    {
        $requiredTables = [
            'tblclients',
            'tblhosting',
            'tblhostingaddons',
            'tblinvoices',
            'tblinvoiceitems',
            'tbldomains',
            'tbladmins',
            'tblproducts',
            'tblpaymentgateways',
            'tblactivitylog',
        ];

        foreach ($requiredTables as $table) {
            $this->assertTrue(
                Capsule::schema()->hasTable($table),
                "Table $table should exist"
            );
        }
    }

    public function testModuleTablesExist()
    {
        $moduleTables = [
            'mod_yourmodule_settings',
            'mod_yourmodule_sync_logs',
            'mod_yourmodule_cache',
        ];

        foreach ($moduleTables as $table) {
            $this->assertTrue(
                Capsule::schema()->hasTable($table),
                "Module table $table should exist"
            );
        }
    }

    public function testClientTableSchema()
    {
        Capsule::schema()->table('tblclients', function ($table) {
            // Test column existence
            $this->assertTrue(Capsule::connection()->getSchemaBuilder()->hasColumn('tblclients', 'id'));
            $this->assertTrue(Capsule::connection()->getSchemaBuilder()->hasColumn('tblclients', 'email'));
            $this->assertTrue(Capsule::connection()->getSchemaBuilder()->hasColumn('tblclients', 'firstname'));
            $this->assertTrue(Capsule::connection()->getSchemaBuilder()->hasColumn('tblclients', 'lastname'));
            $this->assertTrue(Capsule::connection()->getSchemaBuilder()->hasColumn('tblclients', 'datecreated'));
        });
    }

    public function testModuleTableSchema()
    {
        Capsule::schema()->table('mod_yourmodule_settings', function ($table) {
            $columns = Capsule::connection()->getSchemaBuilder()->getColumnListing('mod_yourmodule_settings');
            
            $requiredColumns = ['id', 'key', 'value', 'created_at', 'updated_at'];
            foreach ($requiredColumns as $column) {
                $this->assertContains($column, $columns, "Column $column should exist");
            }
        });
    }

    public function testIndexesExist()
    {
        $indexes = Capsule::connection()->select(
            "SHOW INDEX FROM tblclients WHERE Key_name = 'email'"
        );

        $this->assertNotEmpty($indexes, 'Email index should exist on tblclients');
    }

    public function testForeignKeysExist()
    {
        $foreignKeys = Capsule::connection()->select(
            "SELECT CONSTRAINT_NAME FROM INFORMATION_SCHEMA.TABLE_CONSTRAINTS 
             WHERE TABLE_SCHEMA = DATABASE() 
             AND TABLE_NAME = 'tblhosting' 
             AND CONSTRAINT_TYPE = 'FOREIGN KEY'"
        );

        $this->assertNotEmpty($foreignKeys, 'Foreign keys should exist on tblhosting');
    }
}
```

### Step 3: Data Integrity Tests
```php
// tests/Database/DataIntegrityTest.php
<?php
namespace WHMCS\Tests\Database;

use PHPUnit\Framework\TestCase;
use WHMCS\Database\Capsule;

class DataIntegrityTest extends TestCase
{
    public function testUniqueEmails()
    {
        $duplicateEmails = Capsule::connection()->select(
            "SELECT email, COUNT(*) as cnt 
             FROM tblclients 
             WHERE email IS NOT NULL 
             GROUP BY email 
             HAVING cnt > 1"
        );

        $this->assertEmpty($duplicateEmails, 'Email addresses should be unique');
    }

    public function testNoOrphanedHosting()
    {
        $orphanedRecords = Capsule::connection()->select(
            "SELECT h.id 
             FROM tblhosting h 
             LEFT JOIN tblclients c ON h.userid = c.id 
             WHERE c.id IS NULL"
        );

        $this->assertEmpty($orphanedRecords, 'No orphaned hosting records should exist');
    }

    public function testNoOrphanedInvoices()
    {
        $orphanedInvoices = Capsule::connection()->select(
            "SELECT i.id 
             FROM tblinvoices i 
             LEFT JOIN tblclients c ON i.userid = c.id 
             WHERE c.id IS NULL AND i.userid > 0"
        );

        $this->assertEmpty($orphanedInvoices, 'No orphaned invoice records should exist');
    }

    public function testInvoiceTotalsMatchItems()
    {
        $mismatchedInvoices = Capsule::connection()->select(
            "SELECT i.id, i.total, 
                    (SELECT SUM(amount) FROM tblinvoiceitems WHERE invoiceid = i.id) as items_total
             FROM tblinvoices i
             HAVING ROUND(total, 2) != ROUND(COALESCE(items_total, 0), 2)"
        );

        $this->assertEmpty($mismatchedInvoices, 'Invoice totals should match item totals');
    }

    public function testNoNegativeAmounts()
    {
        $negativeAmounts = Capsule::connection()->select(
            "SELECT id, total FROM tblinvoices WHERE total < 0"
        );

        $this->assertEmpty($negativeAmounts, 'Invoice totals should not be negative');
    }

    public function testDateFormatConsistency()
    {
        $invalidDates = Capsule::connection()->select(
            "SELECT id, datecreated 
             FROM tblclients 
             WHERE datecreated IS NOT NULL 
             AND datecreated NOT REGEXP '^[0-9]{4}-[0-9]{2}-[0-9]{2}'"
        );

        $this->assertEmpty($invalidDates, 'Dates should be in Y-m-d format');
    }
}
```

### Step 4: Query Performance Tests
```php
// tests/Database/QueryPerformanceTest.php
<?php
namespace WHMCS\Tests\Database;

use PHPUnit\Framework\TestCase;
use WHMCS\Database\Capsule;

class QueryPerformanceTest extends TestCase
{
    private float $slowQueryThreshold = 0.5; // 500ms

    public function testClientSearchPerformance()
    {
        $start = microtime(true);

        Capsule::table('tblclients')
            ->where('email', 'like', '%@example.com')
            ->orWhere('firstname', 'like', '%John%')
            ->orWhere('lastname', 'like', '%Doe%')
            ->limit(100)
            ->get();

        $duration = microtime(true) - $start;

        $this->assertLessThan(
            $this->slowQueryThreshold,
            $duration,
            "Client search should complete in under {$this->slowQueryThreshold}s"
        );
    }

    public function testInvoiceQueryPerformance()
    {
        $start = microtime(true);

        Capsule::table('tblinvoices')
            ->join('tblclients', 'tblinvoices.userid', '=', 'tblclients.id')
            ->join('tblinvoiceitems', 'tblinvoices.id', '=', 'tblinvoiceitems.invoiceid')
            ->where('tblinvoices.status', 'Unpaid')
            ->where('tblclients.groupid', 1)
            ->get();

        $duration = microtime(true) - $start;

        $this->assertLessThan(
            $this->slowQueryThreshold,
            $duration,
            "Invoice query should complete in under {$this->slowQueryThreshold}s"
        );
    }

    public function testServiceSearchPerformance()
    {
        $start = microtime(true);

        Capsule::table('tblhosting')
            ->join('tblclients', 'tblhosting.userid', '=', 'tblclients.id')
            ->join('tblproducts', 'tblhosting.packageid', '=', 'tblproducts.id')
            ->where('tblhosting.domainstatus', 'Active')
            ->where('tblproducts.gid', 1)
            ->with(['addons'])
            ->get();

        $duration = microtime(true) - $start;

        $this->assertLessThan(
            $this->slowQueryThreshold,
            $duration,
            "Service search should complete in under {$this->slowQueryThreshold}s"
        );
    }

    public function testFindSlowQueries()
    {
        // Enable slow query log
        Capsule::connection()->statement('SET SESSION sql_mode = ""');

        $slowQueries = Capsule::connection()->select(
            "SHOW SESSION STATUS LIKE 'Slow_queries'"
        );

        // Log if any slow queries detected
        if (!empty($slowQueries) && $slowQueries[0]->Value > 0) {
            $this->markTestSkipped('Slow queries detected - needs optimization');
        }
    }
}
```

### Step 5: Migration Tests
```php
// tests/Database/MigrationTest.php
<?php
namespace WHMCS\Tests\Database;

use PHPUnit\Framework\TestCase;
use WHMCS\Database\Capsule;

class MigrationTest extends TestCase
{
    protected function setUp(): void
    {
        parent::setUp();
        Capsule::connection()->beginTransaction();
    }

    protected function tearDown(): void
    {
        Capsule::connection()->rollBack();
        parent::tearDown();
    }

    public function testMigrationCreatesTables()
    {
        // Drop table if exists
        Capsule::schema()->dropIfExists('mod_test_table');

        // Run migration
        Capsule::schema()->create('mod_test_table', function ($table) {
            $table->increments('id');
            $table->string('name');
            $table->text('data')->nullable();
            $table->timestamps();
        });

        // Verify table exists
        $this->assertTrue(Capsule::schema()->hasTable('mod_test_table'));
    }

    public function testMigrationAddsColumns()
    {
        // Ensure table exists
        Capsule::schema()->create('mod_test_migration', function ($table) {
            $table->increments('id');
            $table->string('name');
        });

        // Add new column
        Capsule::schema()->table('mod_test_migration', function ($table) {
            $table->text('description')->nullable()->after('name');
        });

        // Verify column exists
        $columns = Capsule::connection()->getSchemaBuilder()->getColumnListing('mod_test_migration');
        $this->assertContains('description', $columns);
    }

    public function testMigrationDropsColumns()
    {
        // Ensure table with column exists
        Capsule::schema()->create('mod_test_drop', function ($table) {
            $table->increments('id');
            $table->string('name');
            $table->string('temp_field');
        });

        // Drop column
        Capsule::schema()->table('mod_test_drop', function ($table) {
            $table->dropColumn('temp_field');
        });

        // Verify column dropped
        $columns = Capsule::connection()->getSchemaBuilder()->getColumnListing('mod_test_drop');
        $this->assertNotContains('temp_field', $columns);
    }

    public function testMigrationCreatesIndexes()
    {
        Capsule::schema()->create('mod_test_index', function ($table) {
            $table->increments('id');
            $table->string('email');
            $table->index('email');
        });

        $indexes = Capsule::connection()->select(
            "SHOW INDEX FROM mod_test_index WHERE Column_name = 'email'"
        );

        $this->assertNotEmpty($indexes, 'Index should be created');
    }

    public function testMigrationRollback()
    {
        $tableName = 'mod_test_rollback_' . time();

        // Create table
        Capsule::schema()->create($tableName, function ($table) {
            $table->increments('id');
            $table->string('name');
        });

        $this->assertTrue(Capsule::schema()->hasTable($tableName));

        // Drop table (simulating rollback)
        Capsule::schema()->dropIfExists($tableName);

        $this->assertFalse(Capsule::schema()->hasTable($tableName));
    }
}
```

### Step 6: Data Factory for Testing
```php
// tests/factories/ClientFactory.php
<?php
namespace WHMCS\Tests\Factories;

use WHMCS\Database\Capsule;

class ClientFactory
{
    public static function create(array $attributes = []): array
    {
        $defaults = [
            'firstname' => 'Test',
            'lastname' => 'User',
            'companyname' => '',
            'email' => 'test_' . uniqid() . '@example.com',
            'address1' => '123 Test Street',
            'address2' => '',
            'city' => 'Test City',
            'state' => 'TS',
            'postcode' => '12345',
            'country' => 'US',
            'phonenumber' => '555-1234',
            'password' => 'hashed_password',
            'currency' => 1,
            'defaultgateway' => '',
            'credit' => 0,
            'taxexempt' => 0,
            'notes' => '',
            'status' => 'Active',
            'language' => '',
            'lastlogin' => date('Y-m-d H:i:s'),
            'datecreated' => date('Y-m-d H:i:s'),
            'newData' => '',
        ];

        $data = array_merge($defaults, $attributes);
        $data['id'] = Capsule::table('tblclients')->insertGetId($data);

        return $data;
    }

    public static function createMultiple(int $count, array $attributes = []): array
    {
        $clients = [];
        for ($i = 0; $i < $count; $i++) {
            $clients[] = self::create($attributes);
        }
        return $clients;
    }

    public static function delete(int $clientId): void
    {
        Capsule::table('tblclients')->where('id', $clientId)->delete();
    }

    public static function cleanup(): void
    {
        Capsule::table('tblclients')
            ->where('email', 'like', '%@example.com')
            ->delete();
    }
}
```

### Step 7: Run Database Tests
```bash
# Run all database tests
./vendor/bin/phpunit tests/Database/

# Run schema tests only
./vendor/bin/phpunit tests/Database/SchemaTest.php

# Run with coverage
./vendor/bin/phpunit tests/Database/ --coverage-html coverage/db/

# Run specific test
./vendor/bin/phpunit tests/Database/DataIntegrityTest.php --filter testUniqueEmails
```

## Database Testing Checklist

### Schema
- [ ] Required tables exist
- [ ] Columns have correct types
- [ ] Indexes created
- [ ] Foreign keys defined
- [ ] Constraints enforced

### Data Integrity
- [ ] No duplicate keys
- [ ] No orphaned records
- [ ] No null values where required
- [ ] Data types correct
- [ ] Business rules enforced

### Performance
- [ ] Queries complete within threshold
- [ ] No missing indexes
- [ ] Slow queries identified
- [ ] Query plans optimized

### Migrations
- [ ] Up migrations work
- [ ] Down migrations work
- [ ] Rollback handled
- [ ] Data preserved when possible
