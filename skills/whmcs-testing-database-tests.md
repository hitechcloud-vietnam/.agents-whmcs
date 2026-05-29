# WHMCS Testing - Database Tests

## Skill Description
Implement comprehensive database testing for WHMCS modules to verify data integrity, relationships, and query functionality.

## Prerequisites
- WHMCS 7.0+ installation
- PHP 7.4+ with PHPUnit
- Separate test database
- Database testing knowledge

## Step-by-Step Implementation

### 1. Database Test Traits
```php
<?php
// tests/traits/DatabaseMigrations.php

namespace WHMCS\Module\YourModule\Tests\Traits;

trait DatabaseMigrations
{
    protected static function runMigrations(): void
    {
        $pdo = self::getPdo();

        // Create test tables
        $pdo->exec("
            CREATE TABLE IF NOT EXISTS test_clients (
                id INT AUTO_INCREMENT PRIMARY KEY,
                firstname VARCHAR(100),
                lastname VARCHAR(100),
                email VARCHAR(255) UNIQUE,
                created_at DATETIME
            )
        ");

        $pdo->exec("
            CREATE TABLE IF NOT EXISTS test_services (
                id INT AUTO_INCREMENT PRIMARY KEY,
                client_id INT,
                domain VARCHAR(255),
                status VARCHAR(50),
                created_at DATETIME,
                FOREIGN KEY (client_id) REFERENCES test_clients(id)
            )
        ");
    }

    protected static function rollbackMigrations(): void
    {
        $pdo = self::getPdo();

        $pdo->exec("DROP TABLE IF EXISTS test_services");
        $pdo->exec("DROP TABLE IF EXISTS test_clients");
    }
}
```

### 2. Database Test Examples
```php
<?php
// tests/Database/DatabaseIntegrityTest.php

namespace WHMCS\Module\YourModule\Tests\Database;

use PHPUnit\Framework\TestCase;
use WHMCS\Module\YourModule\Tests\Traits\DatabaseMigrations;

class DatabaseIntegrityTest extends TestCase
{
    use DatabaseMigrations;

    protected static $pdo;

    public static function setUpBeforeClass(): void
    {
        self::$pdo = new PDO('mysql:host=127.0.0.1;dbname=whmcs_test', 'root', '');
        self::runMigrations();
    }

    public static function tearDownAfterClass(): void
    {
        self::rollbackMigrations();
    }

    protected function setUp(): void
    {
        self::$pdo->exec("TRUNCATE test_clients");
        self::$pdo->exec("TRUNCATE test_services");
    }

    public function testForeignKeyConstraint(): void
    {
        // Insert valid client
        $stmt = self::$pdo->prepare("INSERT INTO test_clients (firstname, email) VALUES (?, ?)");
        $stmt->execute(['John', 'john@example.com']);
        $clientId = self::$pdo->lastInsertId();

        // Insert service with valid client_id
        $stmt = self::$pdo->prepare("INSERT INTO test_services (client_id, domain) VALUES (?, ?)");
        $stmt->execute([$clientId, 'example.com']);

        $this->assertEquals(1, self::$pdo->lastInsertId());
    }

    public function testUniqueConstraint(): void
    {
        $stmt = self::$pdo->prepare("INSERT INTO test_clients (firstname, email) VALUES (?, ?)");
        $stmt->execute(['John', 'unique@example.com']);

        // Attempt duplicate email
        $stmt->execute(['Jane', 'unique@example.com']);

        $this->assertEquals(1, $stmt->rowCount());
    }

    public function testCascadeDelete(): void
    {
        // Insert client
        $stmt = self::$pdo->prepare("INSERT INTO test_clients (firstname, email) VALUES (?, ?)");
        $stmt->execute(['Test', 'cascade@example.com']);
        $clientId = self::$pdo->lastInsertId();

        // Insert service
        $stmt = self::$pdo->prepare("INSERT INTO test_services (client_id, domain) VALUES (?, ?)");
        $stmt->execute([$clientId, 'cascade.example.com']);

        // Delete client (without cascade)
        $stmt = self::$pdo->prepare("DELETE FROM test_clients WHERE id = ?");
        $stmt->execute([$clientId]);

        // Verify service still exists but client_id is orphaned
        $stmt = self::$pdo->query("SELECT * FROM test_services WHERE client_id = " . $clientId);
        $this->assertEmpty($stmt->fetchAll());
    }
}
```

## Common Pitfalls and Solutions

| Pitfall | Solution |
|---------|----------|
| Test database pollution | Always truncate before tests |
| Foreign key errors | Insert in correct order |
| Slow tests | Use transactions for isolation |
| Data dependencies | Use factories |

## Testing Checklist

- [ ] Test foreign keys
- [ ] Test unique constraints
- [ ] Test cascade behaviors
- [ ] Test indexes
- [ ] Test data types
- [ ] Test NULL handling

## Reference Links

- [MySQL Testing](https://dev.mysql.com/doc/mysql-for-exercise/)
- [Database Testing Guide](https://phpunit.readthedocs.io/en/9.5/database.html)
