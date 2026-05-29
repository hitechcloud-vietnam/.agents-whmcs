# WHMCS Testing - Integration Tests

## Skill Description
Implement integration tests for WHMCS modules to test module interactions, database operations, and end-to-end workflows.

## Prerequisites
- WHMCS 7.0+ installation
- PHP 7.4+ with PHPUnit
- Separate test database
- Basic testing knowledge

## Step-by-Step Implementation

### 1. Integration Test Case
```php
<?php
// tests/IntegrationTestCase.php

namespace WHMCS\Module\YourModule\Tests\Integration;

use PHPUnit\Framework\TestCase;

abstract class IntegrationTestCase extends TestCase
{
    protected static PDO $db;
    protected static bool $dbSetUp = false;

    public static function setUpBeforeClass(): void
    {
        parent::setUpBeforeClass();

        // Set up test database connection
        self::$db = new PDO(
            'mysql:host=127.0.0.1;dbname=whmcs_test',
            'root',
            '',
            [PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION]
        );

        self::setUpDatabase();
    }

    protected static function setUpDatabase(): void
    {
        // Run migrations
        self::$db->exec("
            CREATE TABLE IF NOT EXISTS mod_yourmodule_integration_test (
                id INT AUTO_INCREMENT PRIMARY KEY,
                name VARCHAR(255) NOT NULL,
                email VARCHAR(255),
                status VARCHAR(50) DEFAULT 'active',
                created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
                updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
    }

    protected static function tearDownAfterClass(): void
    {
        // Clean up test tables
        self::$db->exec("DROP TABLE IF EXISTS mod_yourmodule_integration_test");
        self::$db = null;

        parent::tearDownAfterClass();
    }

    protected function setUp(): void
    {
        parent::setUp();

        // Clean table before each test
        self::$db->exec("TRUNCATE TABLE mod_yourmodule_integration_test");
    }

    protected function assertDatabaseHas(string $table, array $data): void
    {
        $where = [];
        $params = [];

        foreach ($data as $column => $value) {
            $where[] = "{$column} = ?";
            $params[] = $value;
        }

        $sql = "SELECT COUNT(*) as cnt FROM {$table} WHERE " . implode(' AND ', $where);
        $stmt = self::$db->prepare($sql);
        $stmt->execute($params);
        $result = $stmt->fetch(PDO::FETCH_ASSOC);

        $this->assertGreaterThan(0, $result['cnt'], "No record found in {$table} with data: " . json_encode($data));
    }

    protected function assertDatabaseMissing(string $table, array $data): void
    {
        $where = [];
        $params = [];

        foreach ($data as $column => $value) {
            $where[] = "{$column} = ?";
            $params[] = $value;
        }

        $sql = "SELECT COUNT(*) as cnt FROM {$table} WHERE " . implode(' AND ', $where);
        $stmt = self::$db->prepare($sql);
        $stmt->execute($params);
        $result = $stmt->fetch(PDO::FETCH_ASSOC);

        $this->assertEquals(0, $result['cnt'], "Record found in {$table} with data: " . json_encode($data));
    }

    protected function insertRecord(string $table, array $data): int
    {
        $columns = implode(', ', array_keys($data));
        $placeholders = implode(', ', array_fill(0, count($data), '?'));

        $sql = "INSERT INTO {$table} ({$columns}) VALUES ({$placeholders})";
        $stmt = self::$db->prepare($sql);
        $stmt->execute(array_values($data));

        return (int) self::$db->lastInsertId();
    }
}
```

### 2. Integration Test Examples
```php
<?php
// tests/Integration/DatabaseOperationsTest.php

namespace WHMCS\Module\YourModule\Tests\Integration;

class DatabaseOperationsTest extends IntegrationTestCase
{
    public function testCreateRecord(): void
    {
        $id = $this->insertRecord('mod_yourmodule_integration_test', [
            'name' => 'Test User',
            'email' => 'test@example.com',
            'status' => 'active'
        ]);

        $this->assertGreaterThan(0, $id);
        $this->assertDatabaseHas('mod_yourmodule_integration_test', [
            'id' => $id,
            'name' => 'Test User'
        ]);
    }

    public function testUpdateRecord(): void
    {
        $id = $this->insertRecord('mod_yourmodule_integration_test', [
            'name' => 'Original Name',
            'email' => 'original@example.com',
            'status' => 'active'
        ]);

        // Update the record
        $sql = "UPDATE mod_yourmodule_integration_test SET name = ? WHERE id = ?";
        $stmt = self::$db->prepare($sql);
        $stmt->execute(['Updated Name', $id]);

        $this->assertDatabaseHas('mod_yourmodule_integration_test', [
            'id' => $id,
            'name' => 'Updated Name'
        ]);

        $this->assertDatabaseMissing('mod_yourmodule_integration_test', [
            'id' => $id,
            'name' => 'Original Name'
        ]);
    }

    public function testDeleteRecord(): void
    {
        $id = $this->insertRecord('mod_yourmodule_integration_test', [
            'name' => 'To Be Deleted',
            'email' => 'delete@example.com',
            'status' => 'active'
        ]);

        // Delete the record
        $sql = "DELETE FROM mod_yourmodule_integration_test WHERE id = ?";
        $stmt = self::$db->prepare($sql);
        $stmt->execute([$id]);

        $this->assertDatabaseMissing('mod_yourmodule_integration_test', ['id' => $id]);
    }

    public function testTransactionRollback(): void
    {
        try {
            self::$db->beginTransaction();

            $this->insertRecord('mod_yourmodule_integration_test', [
                'name' => 'Transaction Test',
                'email' => 'transaction@example.com',
                'status' => 'active'
            ]);

            throw new \Exception('Simulated error');

        } catch (\Exception $e) {
            self::$db->rollBack();
        }

        $this->assertDatabaseMissing('mod_yourmodule_integration_test', [
            'email' => 'transaction@example.com'
        ]);
    }

    public function testTransactionCommit(): void
    {
        self::$db->beginTransaction();

        $this->insertRecord('mod_yourmodule_integration_test', [
            'name' => 'Commit Test',
            'email' => 'commit@example.com',
            'status' => 'active'
        ]);

        self::$db->commit();

        $this->assertDatabaseHas('mod_yourmodule_integration_test', [
            'email' => 'commit@example.com'
        ]);
    }

    public function testForeignKeyConstraint(): void
    {
        // This test verifies foreign key behavior
        $sql = "SET FOREIGN_KEY_CHECKS = 1";
        self::$db->exec($sql);

        // Create parent record
        $parentId = $this->insertRecord('mod_yourmodule_integration_test', [
            'name' => 'Parent',
            'email' => 'parent@example.com',
            'status' => 'active'
        ]);

        $this->assertEquals(1, $parentId);
    }
}
```

```php
<?php
// tests/Integration/WorkflowTest.php

namespace WHMCS\Module\YourModule\Tests\Integration;

class WorkflowTest extends IntegrationTestCase
{
    public function testCompleteOrderWorkflow(): void
    {
        // Step 1: Create client
        $clientId = $this->insertRecord('mod_yourmodule_integration_test', [
            'name' => 'New Client',
            'email' => 'client@example.com',
            'status' => 'active'
        ]);

        $this->assertGreaterThan(0, $clientId);

        // Step 2: Create order
        $orderId = $this->insertRecord('mod_yourmodule_integration_test', [
            'name' => 'Order for Client',
            'email' => 'order@example.com',
            'status' => 'pending'
        ]);

        // Step 3: Create invoice
        $invoiceId = $this->insertRecord('mod_yourmodule_integration_test', [
            'name' => 'Invoice',
            'email' => 'invoice@example.com',
            'status' => 'unpaid'
        ]);

        // Step 4: Process payment (update invoice status)
        $sql = "UPDATE mod_yourmodule_integration_test SET status = ? WHERE id = ?";
        $stmt = self::$db->prepare($sql);
        $stmt->execute(['paid', $invoiceId]);

        // Step 5: Activate service
        $sql = "UPDATE mod_yourmodule_integration_test SET status = ? WHERE id = ?";
        $stmt = self::$db->prepare($sql);
        $stmt->execute(['completed', $orderId]);

        // Verify final state
        $this->assertDatabaseHas('mod_yourmodule_integration_test', [
            'id' => $clientId,
            'status' => 'active'
        ]);

        $this->assertDatabaseHas('mod_yourmodule_integration_test', [
            'id' => $orderId,
            'status' => 'completed'
        ]);

        $this->assertDatabaseHas('mod_yourmodule_integration_test', [
            'id' => $invoiceId,
            'status' => 'paid'
        ]);
    }

    public function testServiceSuspensionWorkflow(): void
    {
        // Create active service
        $serviceId = $this->insertRecord('mod_yourmodule_integration_test', [
            'name' => 'Active Service',
            'email' => 'service@example.com',
            'status' => 'active'
        ]);

        // Suspend service
        $sql = "UPDATE mod_yourmodule_integration_test SET status = ? WHERE id = ?";
        $stmt = self::$db->prepare($sql);
        $stmt->execute(['suspended', $serviceId]);

        $this->assertDatabaseHas('mod_yourmodule_integration_test', [
            'id' => $serviceId,
            'status' => 'suspended'
        ]);

        // Reactivate service
        $sql = "UPDATE mod_yourmodule_integration_test SET status = ? WHERE id = ?";
        $stmt = self::$db->prepare($sql);
        $stmt->execute(['active', $serviceId]);

        $this->assertDatabaseHas('mod_yourmodule_integration_test', [
            'id' => $serviceId,
            'status' => 'active'
        ]);
    }

    public function testServiceTerminationWorkflow(): void
    {
        // Create service
        $serviceId = $this->insertRecord('mod_yourmodule_integration_test', [
            'name' => 'Service to Terminate',
            'email' => 'terminate@example.com',
            'status' => 'active'
        ]);

        // Terminate service
        $sql = "UPDATE mod_yourmodule_integration_test SET status = ? WHERE id = ?";
        $stmt = self::$db->prepare($sql);
        $stmt->execute(['terminated', $serviceId]);

        // Verify termination
        $this->assertDatabaseHas('mod_yourmodule_integration_test', [
            'id' => $serviceId,
            'status' => 'terminated'
        ]);

        // Verify no further updates to terminated service
        $sql = "UPDATE mod_yourmodule_integration_test SET status = ? WHERE id = ?";
        $stmt = self::$db->prepare($sql);
        $stmt->execute(['active', $serviceId]);

        // Status should still be terminated (business logic enforcement)
        $sql = "SELECT status FROM mod_yourmodule_integration_test WHERE id = ?";
        $stmt = self::$db->prepare($sql);
        $stmt->execute([$serviceId]);
        $result = $stmt->fetch(PDO::FETCH_ASSOC);

        $this->assertEquals('terminated', $result['status']);
    }
}
```

### 3. Database Factory for Tests
```php
<?php
// tests/Integration/Factory.php

namespace WHMCS\Module\YourModule\Tests\Integration;

class Factory
{
    public static function client(array $overrides = []): array
    {
        $faker = \Faker\Factory::create();

        return array_merge([
            'name' => $faker->name,
            'email' => $faker->unique()->safeEmail,
            'status' => 'active',
            'created_at' => date('Y-m-d H:i:s')
        ], $overrides);
    }

    public static function invoice(array $overrides = []): array
    {
        $faker = \Faker\Factory::create();

        return array_merge([
            'userid' => 1,
            'invoicenum' => 'INV-' . $faker->numerify('######'),
            'date' => date('Y-m-d'),
            'duedate' => date('Y-m-d', strtotime('+14 days')),
            'subtotal' => 100.00,
            'total' => 100.00,
            'status' => 'Unpaid',
            'created_at' => date('Y-m-d H:i:s')
        ], $overrides);
    }

    public static function service(array $overrides = []): array
    {
        $faker = \Faker\Factory::create();

        return array_merge([
            'userid' => 1,
            'packageid' => 1,
            'domain' => strtolower($faker->domainWord) . '.example.com',
            'registrationdate' => date('Y-m-d'),
            'nextduedate' => date('Y-m-d', strtotime('+30 days')),
            'billingcycle' => 'Monthly',
            'amount' => 9.99,
            'domainstatus' => 'Active',
            'created_at' => date('Y-m-d H:i:s')
        ], $overrides);
    }
}
```

## Common Pitfalls and Solutions

| Pitfall | Solution |
|---------|----------|
| Test database conflicts | Use transactions for isolation |
| Slow integration tests | Use in-memory database |
| Data pollution | Clean up before each test |
| Complex setup | Use factories and fixtures |
| Order-dependent tests | Make tests independent |

## Security Considerations

1. **Use separate test database** - Never test on production
2. **Clean up test data** - Remove all test records
3. **Secure test credentials** - Don't commit to version control
4. **Mock external services** - Don't rely on live APIs
5. **Validate test environment** - Check for proper setup

## Testing Checklist

- [ ] Test database operations
- [ ] Test transaction handling
- [ ] Test foreign key constraints
- [ ] Test workflows
- [ ] Test data cleanup
- [ ] Test database isolation
- [ ] Test with factories
- [ ] Run tests in CI/CD

## Reference Links

- [Laravel Integration Testing](https://laravel.com/docs/database-testing)
- [Database Testing in PHPUnit](https://phpunit.readthedocs.io/en/9.5/database.html)
