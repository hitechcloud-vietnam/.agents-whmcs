# WHMCS Database Design Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Guide for designing database schemas for WHMCS addon modules using Capsule (Eloquent).

## When to Use

- Creating database tables for addon modules
- Designing upgrade paths for schema migrations
- Optimizing queries for performance

## Basic Table Creation

```php
// In activate() function
use WHMCS\Database\Capsule;

// Settings table
Capsule::schema()->create('mod_{module}_settings', function($t) {
    $t->increments('id');
    $t->string('setting_key', 100)->unique();
    $t->text('setting_value')->nullable();
    $t->timestamps();
});

// Log table
Capsule::schema()->create('mod_{module}_logs', function($t) {
    $t->increments('id');
    $t->string('level', 20)->default('info');
    $t->string('action', 50);
    $t->text('message');
    $t->text('context')->nullable();
    $t->integer('user_id')->unsigned()->nullable();
    $t->integer('admin_id')->unsigned()->nullable();
    $t->timestamp('created_at')->useCurrent();
    $t->index(['level', 'created_at']);
    $t->index(['user_id']);
});

// Data table
Capsule::schema()->create('mod_{module}_data', function($t) {
    $t->increments('id');
    $t->integer('user_id')->unsigned();
    $t->integer('service_id')->unsigned()->nullable();
    $t->string('type', 50);
    $t->string('status', 20)->default('active');
    $t->string('external_id', 100)->nullable();
    $t->text('metadata')->nullable();
    $t->decimal('amount', 10, 2)->nullable();
    $t->timestamps();
    $t->index(['user_id', 'type']);
    $t->index(['service_id']);
    $t->index(['external_id']);
});
```

## Column Types Reference

```php
// Integer types
$t->increments('id');                    // INT AUTO_INCREMENT
$t->integer('count');                    // INT
$t->bigInteger('large_id');             // BIGINT
$t->tinyInteger('tiny_flag');           // TINYINT (0-255)
$t->unsignedInteger('positive_id');      // INT UNSIGNED

// String types
$t->string('code', 50);                  // VARCHAR(50)
$t->char('fixed', 10);                   // CHAR(10)
$t->text('description');                // TEXT
$t->mediumText('content');               // MEDIUMTEXT
$t->longText('long_content');            // LONGTEXT

// Numeric types
$t->decimal('price', 10, 2);            // DECIMAL(10,2)
$t->float('rate');                      // FLOAT
$t->double('precise', 8, 4);            // DOUBLE(8,4)
$t->boolean('active');                  // TINYINT(1)

// Date/Time types
$t->date('expire_date');               // DATE
$t->dateTime('created_at');            // DATETIME
$t->timestamp('updated_at');           // TIMESTAMP
$t->timestamp('created_at')->useCurrent(); // Default CURRENT_TIMESTAMP

// JSON and other
$t->json('options');                    // JSON
$t->binary('data');                     // BLOB
$t->enum('status', ['pending', 'active', 'suspended']); // ENUM
```

## Schema Modifications

```php
// Add column
Capsule::schema()->table('mod_module_data', function($t) {
    $t->string('new_field', 100)->after('existing_field');
});

// Rename column
Capsule::schema()->table('mod_module_data', function($t) {
    $t->renameColumn('old_name', 'new_name');
});

// Drop column
Capsule::schema()->table('mod_module_data', function($t) {
    $t->dropColumn('unused_field');
});

// Add index
Capsule::schema()->table('mod_module_data', function($t) {
    $t->index(['user_id', 'type'], 'idx_user_type');
});

// Add foreign key
Capsule::schema()->table('mod_module_data', function($t) {
    $t->foreign('user_id')->references('id')->on('tblusers')->onDelete('cascade');
});
```

## Migration Pattern

```php
function {module}_upgrade(array $vars): void {
    $fromVersion = $vars['version'];

    if (version_compare($fromVersion, '1.1', '<')) {
        // Migration 1.0 -> 1.1
        Capsule::schema()->table('mod_{module}_data', function($t) {
            if (!Capsule::schema()->hasColumn('mod_{module}_data', 'metadata')) {
                $t->text('metadata')->nullable();
            }
        });
    }

    if (version_compare($fromVersion, '1.2', '<')) {
        // Migration 1.1 -> 1.2
        Capsule::schema()->table('mod_{module}_logs', function($t) {
            if (!Capsule::schema()->hasColumn('mod_{module}_logs', 'extra_data')) {
                $t->text('extra_data')->nullable();
            }
        });

        // Create new table
        if (!Capsule::schema()->hasTable('mod_{module}_archive')) {
            Capsule::schema()->create('mod_{module}_archive', function($t) {
                $t->increments('id');
                $t->text('data');
                $t->timestamp('archived_at')->useCurrent();
            });
        }
    }
}
```

## Query Patterns

```php
// Insert
Capsule::table('mod_module_data')->insert([
    'user_id' => $userId,
    'type' => 'order',
    'metadata' => json_encode(['order_id' => 123]),
    'created_at' => date('Y-m-d H:i:s'),
]);

// Update
Capsule::table('mod_module_data')
    ->where('id', $id)
    ->update(['status' => 'completed']);

// Delete
Capsule::table('mod_module_data')
    ->where('id', $id)
    ->delete();

// Query with conditions
$results = Capsule::table('mod_module_data')
    ->where('user_id', $userId)
    ->whereIn('status', ['active', 'pending'])
    ->whereBetween('created_at', [$startDate, $endDate])
    ->orderBy('created_at', 'desc')
    ->limit(10)
    ->get();

// Aggregation
$count = Capsule::table('mod_module_data')
    ->where('user_id', $userId)
    ->count();

$total = Capsule::table('mod_module_data')
    ->where('user_id', $userId)
    ->sum('amount');
```

## Checklist

- [ ] Table names prefixed with mod_
- [ ] All tables have id as primary key
- [ ] Proper indexes on foreign keys
- [ ] Timestamps on all tables
- [ ] Migration path for upgrades
- [ ] Safe column checks in upgrades

---

**Related Skills:**
- whmcs-addon-builder
- whmcs-orm-patterns
- whmcs-performance