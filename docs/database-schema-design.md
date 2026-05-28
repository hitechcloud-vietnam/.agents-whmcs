# WHMCS Database Schema Design Guide
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Database schema design best practices for WHMCS modules.

## Naming Conventions

```
Tables:   mod_{module}_{name}
Columns:  snake_case
Indexes:  idx_{table}_{columns}
```

## Schema Examples

### 1. Main Data Table

```php
Capsule::schema()->create('mod_{module}_items', function($t) {
    // Primary key
    $t->increments('id');

    // Foreign key
    $t->unsignedInteger('user_id');
    $t->foreign('user_id')->references('id')->on('tblclients');

    // Data columns
    $t->string('name', 255);
    $t->text('description')->nullable();
    $t->string('status', 50)->default('pending');

    // Pricing
    $t->decimal('price', 10, 2)->default(0);
    $t->decimal('setup_fee', 10, 2)->default(0);

    // JSON data
    $t->json('metadata')->nullable();

    // Timestamps
    $t->timestamps();

    // Indexes
    $t->index(['user_id', 'status']);
    $t->unique(['user_id', 'name']);
});
```

### 2. Settings Table

```php
Capsule::schema()->create('mod_{module}_settings', function($t) {
    $t->increments('id');
    $t->string('setting_key', 100)->unique();
    $t->text('setting_value')->nullable();
    $t->timestamps();
});
```

### 3. Log Table

```php
Capsule::schema()->create('mod_{module}_logs', function($t) {
    $t->increments('id');
    $t->unsignedInteger('user_id')->nullable();
    $t->unsignedInteger('admin_id')->nullable();
    $t->string('level', 20)->default('info');
    $t->text('message');
    $t->text('context')->nullable();
    $t->timestamp('created_at');

    $t->index(['level', 'created_at']);
    $t->index(['user_id', 'created_at']);
});
```

### 4. Many-to-Many Relationship

```php
Capsule::schema()->create('mod_{module}_services', function($t) {
    $t->increments('id');
    $t->unsignedInteger('service_id');
    $t->unsignedInteger('addon_id');
    $t->timestamps();

    $t->foreign('service_id')->references('id')->on('tblhosting');
    $t->foreign('addon_id')->references('id')->on('tblproducts');

    $t->unique(['service_id', 'addon_id']);
});
```

## Index Strategy

### When to Add Indexes
- Columns used in WHERE clauses
- Foreign key columns
- Columns in ORDER BY
- Columns in GROUP BY

### Index Types
```php
// Single column
$t->index('user_id');

// Composite
$t->index(['user_id', 'status']);

// Unique
$t->unique('email');

// Full-text
$t->text('content');
```

## Migrations

```php
function upgrade_1_1(): void {
    // Add index
    Capsule::schema()->table('mod_{module}_items', function($t) {
        if (!$t->hasIndex('idx_status')) {
            $t->index('status', 'idx_status');
        }
    });

    // Add column
    Capsule::schema()->table('mod_{module}_items', function($t) {
        if (!$t->hasColumn('category')) {
            $t->string('category', 100)->nullable();
        }
    });
}
```

---

**Related Skills:**
- whmcs-database-design
- whmcs-migration-guide
