# WHMCS Database Seeding

## Skill Description
Implement database seeding patterns for WHMCS modules to create test data, populate initial configurations, and support development and testing workflows.

## Prerequisites
- WHMCS 7.0+ installation
- PHP 7.4+
- Understanding of database relationships
- Basic faker library knowledge

## Step-by-Step Implementation

### 1. Seed Runner
```php
<?php
// includes/seeding/SeedRunner.php

namespace WHMCS\Module\YourModule\Seeding;

class SeedRunner
{
    private string $seedsPath;

    public function __construct(string $seedsPath = null)
    {
        $this->seedsPath = $seedsPath ?? dirname(__DIR__) . '/seeds';
    }

    public function run(string $class = null): array
    {
        if ($class) {
            return [$this->runSeed($class)];
        }

        $seeds = $this->getSeedClasses();
        $results = [];

        foreach ($seeds as $seed) {
            $results[] = $this->runSeed($seed);
        }

        return $results;
    }

    private function getSeedClasses(): array
    {
        $files = glob($this->seedsPath . '/*Seed.php');
        $classes = [];

        foreach ($files as $file) {
            $className = basename($file, '.php');
            $classes[] = $className;
        }

        return $classes;
    }

    private function runSeed(string $class): array
    {
        $startTime = microtime(true);
        $seedFile = $this->seedsPath . '/' . $class . '.php';

        if (!file_exists($seedFile)) {
            return [
                'class' => $class,
                'status' => 'failed',
                'error' => 'Seed file not found'
            ];
        }

        require_once $seedFile;

        $seedClass = "WHMCS\\Module\\YourModule\\Seeds\\{$class}";

        if (!class_exists($seedClass)) {
            return [
                'class' => $class,
                'status' => 'failed',
                'error' => 'Seed class not found'
            ];
        }

        try {
            $seeder = new $seedClass();
            $count = $seeder->run();

            $duration = round(microtime(true) - $startTime, 4);

            return [
                'class' => $class,
                'status' => 'success',
                'count' => $count,
                'duration' => $duration
            ];

        } catch (\Exception $e) {
            return [
                'class' => $class,
                'status' => 'failed',
                'error' => $e->getMessage()
            ];
        }
    }

    public function fresh(): void
    {
        // Truncate all module tables
        global $db;

        $tables = ['mod_yourmodule_data', 'mod_yourmodule_settings'];

        foreach ($tables as $table) {
            $db->query("TRUNCATE TABLE {$table}");
        }

        // Run all seeds
        $this->run();
    }
}
```

### 2. Base Seeder
```php
<?php
// includes/seeding/Seeder.php

namespace WHMCS\Module\YourModule\Seeding;

abstract class Seeder
{
    protected int $count = 0;
    protected bool $quiet = false;

    public function run(): int
    {
        $this->count = 0;

        $this->seed();

        if (!$this->quiet) {
            echo "Seeded: " . static::class . " ({$this->count} records)\n";
        }

        return $this->count;
    }

    abstract protected function seed(): void;

    protected function create(string $table, array $data): int
    {
        global $db;

        $columns = implode(', ', array_keys($data));
        $placeholders = implode(', ', array_fill(0, count($data), '?'));

        $sql = "INSERT INTO {$table} ({$columns}) VALUES ({$placeholders})";

        $db->query($sql, array_values($data));

        $this->count++;

        return $db->getLastInsertID();
    }

    protected function createMany(string $table, array $records): array
    {
        $ids = [];

        foreach ($records as $record) {
            $ids[] = $this->create($table, $record);
        }

        return $ids;
    }

    protected function truncate(string $table): void
    {
        global $db;
        $db->query("TRUNCATE TABLE {$table}");
    }

    protected function randomId(string $table): int
    {
        global $db;

        $result = $db->select(
            "SELECT id FROM {$table} ORDER BY RAND() LIMIT 1"
        );

        return $result[0]['id'] ?? 0;
    }

    protected function faker(): \Faker\Generator
    {
        return \Faker\Factory::create();
    }
}
```

### 3. Example Seeders
```php
<?php
// includes/seeding/ApiKeysSeed.php

namespace WHMCS\Module\YourModule\Seeds;

use WHMCS\Module\YourModule\Seeding\Seeder;

class ApiKeysSeed extends Seeder
{
    protected function seed(): void
    {
        $this->create('mod_yourmodule_api_keys', [
            'user_id' => 1,
            'name' => 'Development Key',
            'api_key' => bin2hex(random_bytes(32)),
            'api_secret' => password_hash(bin2hex(random_bytes(16)), PASSWORD_ARGON2ID),
            'scopes' => 'read,write',
            'is_active' => 1,
            'created_at' => date('Y-m-d H:i:s')
        ]);

        $this->create('mod_yourmodule_api_keys', [
            'user_id' => 1,
            'name' => 'Production Key',
            'api_key' => bin2hex(random_bytes(32)),
            'api_secret' => password_hash(bin2hex(random_bytes(16)), PASSWORD_ARGON2ID),
            'scopes' => 'read',
            'is_active' => 1,
            'created_at' => date('Y-m-d H:i:s')
        ]);
    }
}
```

```php
<?php
// includes/seeding/TestDataSeed.php

namespace WHMCS\Module\YourModule\Seeds;

use WHMCS\Module\YourModule\Seeding\Seeder;

class TestDataSeed extends Seeder
{
    protected function seed(): void
    {
        $faker = $this->faker();

        // Create test clients
        for ($i = 0; $i < 10; $i++) {
            $this->create('tblclients', [
                'firstname' => $faker->firstName,
                'lastname' => $faker->lastName,
                'email' => $faker->email,
                'companyname' => $faker->company,
                'datecreated' => $faker->dateTimeBetween('-2 years', 'now')->format('Y-m-d'),
                'status' => $faker->randomElement(['Active', 'Inactive']),
                'ip' => $faker->ipv4
            ]);
        }
    }
}
```

### 4. Faker Data Generator
```php
<?php
// includes/seeding/FakerData.php

namespace WHMCS\Module\YourModule\Seeding;

class FakerData
{
    private \Faker\Generator $faker;

    public function __construct(?string $locale = null)
    {
        $this->faker = \Faker\Factory::create($locale ?? 'en_US');
    }

    public function generateClient(): array
    {
        $this->faker->seed(rand());

        return [
            'firstname' => $this->faker->firstName,
            'lastname' => $this->faker->lastName,
            'email' => $this->faker->unique()->safeEmail,
            'companyname' => $this->faker->company,
            'datecreated' => $this->faker->dateTimeBetween('-2 years', 'now')->format('Y-m-d'),
            'status' => $this->faker->randomElement(['Active', 'Active', 'Active', 'Inactive']),
            'ip' => $this->faker->ipv4
        ];
    }

    public function generateService(int $userId): array
    {
        $this->faker->seed(rand());

        $productIds = $this->getProductIds();
        $paymentMethods = ['paypal', 'banktransfer', 'creditcard', 'paywithcredit'];

        return [
            'userid' => $userId,
            'packageid' => $this->faker->randomElement($productIds),
            'domain' => strtolower($this->faker->domainWord) . '.' . $this->faker->domainName,
            'registrationdate' => $this->faker->dateTimeBetween('-1 year', 'now')->format('Y-m-d'),
            'nextduedate' => $this->faker->dateTimeBetween('now', '+1 year')->format('Y-m-d'),
            'billingcycle' => $this->faker->randomElement(['Monthly', 'Quarterly', 'Semi-Annual', 'Annual']),
            'amount' => $this->faker->randomFloat(2, 5, 200),
            'domainstatus' => $this->faker->randomElement(['Active', 'Active', 'Active', 'Suspended', 'Pending']),
            'paymentmethod' => $this->faker->randomElement($paymentMethods)
        ];
    }

    public function generateInvoice(int $userId, float $total): array
    {
        $this->faker->seed(rand());

        $statuses = ['Unpaid', 'Paid', 'Cancelled'];

        return [
            'userid' => $userId,
            'invoicenum' => $this->faker->numerify('INV-######'),
            'date' => $this->faker->dateTimeBetween('-30 days', 'now')->format('Y-m-d'),
            'duedate' => $this->faker->dateTimeBetween('now', '+30 days')->format('Y-m-d'),
            'subtotal' => $total,
            'total' => $total,
            'status' => $this->faker->randomElement($statuses)
        ];
    }

    private function getProductIds(): array
    {
        global $db;

        $result = $db->select("SELECT id FROM tblproducts WHERE status = 'Active'");

        return array_column($result, 'id') ?: [1, 2, 3];
    }
}
```

## Common Pitfalls and Solutions

| Pitfall | Solution |
|---------|----------|
| Duplicate data | Use truncate before seeding |
| Foreign key errors | Seed in correct order |
| Unpredictable data | Seed with fixed random seed |
| Large datasets | Implement batch processing |
| Performance issues | Use transactions for bulk inserts |

## Security Considerations

1. **Don't seed production** - Only seed in development/testing
2. **Remove test data** - Clean up after tests
3. **Use fake data** - Never use real user information
4. **Secure seed files** - Don't commit with real credentials
5. **Validate foreign keys** - Ensure related records exist

## Testing Checklist

- [ ] Test individual seeders
- [ ] Test seed ordering
- [ ] Test with fresh database
- [ ] Test with existing data
- [ ] Test rollback functionality
- [ ] Test data integrity
- [ ] Test foreign key constraints

## Reference Links

- [PHPFaker Library](https://fakerphp.github.io/)
- [Laravel Database Seeding](https://laravel.com/docs/database/seeding)
