# WHMCS Capsule Schema

## Overview

Capsule's schema builder provides methods for creating and modifying database tables.

## Table Creation

### Basic Table

```php
<?php
use WHMCS\Database\Capsule;

Capsule::schema()->create('mod_yourmodule_table', function ($t) {
    $t->increments('id');
    $t->string('name');
    $t->text('description')->nullable();
    $t->timestamps();
});
```

### Available Column Types

```php
<?php
Capsule::schema()->create('mod_example', function ($t) {
    // Auto-incrementing ID
    $t->increments('id');
    
    // Integer types
    $t->integer('qty');
    $t->bigInteger('views');
    $t->mediumInteger('medium');
    $t->smallInteger('small');
    $t->tinyInteger('tiny');
    
    // Decimal types
    $t->decimal('amount', 10, 2);
    $t->float('price');
    $t->double('weight', 8, 2);
    
    // String types
    $t->string('name', 100);
    $t->char('code', 10);
    $t->text('content');
    $t->mediumText('long_content');
    $t->longText('very_long_content');
    
    // Date/Time types
    $t->date('birth_date');
    $t->dateTime('created_at');
    $t->time('start_time');
    $t->timestamp('updated_at');
    $t->timestamps(); // created_at and updated_at
    
    // Boolean
    $t->boolean('is_active');
    
    // Enum
    $t->enum('status', ['pending', 'active', 'completed']);
    
    // JSON
    $t->json('metadata');
    $t->jsonb('data');
    
    // Binary
    $t->binary('image_data');
    
    // UUID
    $t->uuid('uuid');
});
```

### Column Modifiers

```php
<?php
Capsule::schema()->create('mod_example', function ($t) {
    // Nullable
    $t->string('optional')->nullable();
    
    // Default value
    $t->boolean('active')->default(true);
    $t->string('status')->default('pending');
    $t->integer('count')->default(0);
    
    // After column
    $t->string('secondary')->after('primary');
    
    // First column
    $t->string('first')->first();
    
    // Unsigned (for integers)
    $t->increments('id')->unsigned();
    $t->integer('qty')->unsigned();
    
    // Auto-increment starting value
    $t->increments('id')->startingAt(1000);
    
    // Comment
    $t->string('code')->comment('Unique identifier code');
});
```

## Table Modifications

### Adding Columns

```php
<?php
Capsule::schema()->table('mod_yourmodule_table', function ($t) {
    // Add new column
    if (!$t->hasColumn('new_column')) {
        $t->string('new_column')->after('existing_column');
    }
    
    // Add multiple columns
    $t->string('col1')->nullable();
    $t->string('col2')->nullable();
    $t->text('col3')->nullable();
});
```

### Modifying Columns

```php
<?php
Capsule::schema()->table('mod_yourmodule_table', function ($t) {
    // Change column type
    $t->string('name', 255)->change();
    
    // Rename column
    $t->renameColumn('old_name', 'new_name');
    
    // Drop column
    $t->dropColumn('unwanted_column');
    
    // Drop multiple columns
    $t->dropColumn(['col1', 'col2', 'col3']);
});
```

### Indexes

```php
<?php
Capsule::schema()->create('mod_example', function ($t) {
    // Primary key
    $t->increments('id');
    
    // Unique index
    $t->string('email')->unique();
    
    // Index with custom name
    $t->string('code');
    $t->index('code', 'idx_custom_code');
    
    // Composite index
    $t->integer('user_id');
    $t->integer('product_id');
    $t->index(['user_id', 'product_id'], 'idx_user_product');
    
    // Foreign key
    $t->integer('user_id')->unsigned();
    $t->foreign('user_id')
        ->references('id')
        ->on('tblclients')
        ->onDelete('cascade');
});
```

### Adding Indexes to Existing Tables

```php
<?php
Capsule::schema()->table('mod_yourmodule_table', function ($t) {
    // Add unique index
    $t->unique('email', 'unique_email');
    
    // Add index
    $t->index('status', 'idx_status');
    
    // Add composite index
    $t->index(['user_id', 'created_at'], 'idx_user_created');
    
    // Add foreign key
    $t->foreign('user_id')
        ->references('id')
        ->on('tblclients')
        ->onDelete('cascade');
});
```

## Table Operations

### Renaming Tables

```php
<?php
Capsule::schema()->rename('old_table_name', 'new_table_name');
```

### Dropping Tables

```php
<?php
// Drop if exists
Capsule::schema()->dropIfExists('mod_yourmodule_table');

// Conditional drop
if (Capsule::schema()->hasTable('mod_yourmodule_table')) {
    Capsule::schema()->drop('mod_yourmodule_table');
}
```

### Checking Existence

```php
<?php
// Check if table exists
if (Capsule::schema()->hasTable('mod_yourmodule_table')) {
    // Table exists
}

// Check if column exists
if (Capsule::schema()->hasColumn('mod_yourmodule_table', 'email')) {
    // Column exists
}
```

## Advanced Schema Operations

### Conditional Schema

```php
<?php
// Only run if MySQL
Capsule::schema()->getConnection()->getDriverName() === 'mysql';

// Only run if PostgreSQL
if (Capsule::schema()->getConnection()->getDriverName() === 'pgsql') {
    // PostgreSQL specific schema
}

// Platform-specific columns
Capsule::schema()->create('mod_example', function ($t) {
    $t->increments('id');
    $t->string('name');
    
    // Only add JSONB for PostgreSQL
    if ($t->getConnection()->getDriverName() === 'pgsql') {
        $t->jsonb('metadata');
    } else {
        $t->json('metadata');
    }
});
```

### Rename Indexes

```php
<?php
Capsule::schema()->table('mod_yourmodule_table', function ($t) {
    $t->renameIndex('old_index_name', 'new_index_name');
});
```

### Drop Indexes

```php
<?php
Capsule::schema()->table('mod_yourmodule_table', function ($t) {
    $t->dropIndex('idx_status');
    $t->dropUnique('unique_email');
    $t->dropForeign('user_id');
});
```

## Complete Migration Example

```php
<?php
function yourmodule_activate(): array
{
    try {
        if (!Capsule::schema()->hasTable('mod_yourmodule_data')) {
            Capsule::schema()->create('mod_yourmodule_data', function ($t) {
                $t->increments('id');
                $t->integer('user_id')->unsigned();
                $t->string('title', 200);
                $t->text('description')->nullable();
                $t->decimal('amount', 10, 2)->default(0);
                $t->enum('status', ['pending', 'active', 'completed'])->default('pending');
                $t->json('metadata')->nullable();
                $t->timestamps();
                
                // Indexes
                $t->index('user_id', 'idx_user');
                $t->index('status', 'idx_status');
                $t->index(['user_id', 'status'], 'idx_user_status');
                
                // Foreign key
                $t->foreign('user_id')
                    ->references('id')
                    ->on('tblclients')
                    ->onDelete('cascade');
            });
        }
        
        // Create secondary table
        if (!Capsule::schema()->hasTable('mod_yourmodule_log')) {
            Capsule::schema()->create('mod_yourmodule_log', function ($t) {
                $t->increments('id');
                $t->integer('data_id')->unsigned();
                $t->string('action', 50);
                $t->text('details')->nullable();
                $t->timestamp('created_at');
                
                $t->index('data_id', 'idx_data');
            });
        }
        
        return [
            'status' => 'success',
            'description' => 'Tables created successfully',
        ];
        
    } catch (Exception $e) {
        return [
            'status' => 'error',
            'description' => 'Failed: ' . $e->getMessage(),
        ];
    }
}

function yourmodule_deactivate(): array
{
    Capsule::schema()->dropIfExists('mod_yourmodule_log');
    Capsule::schema()->dropIfExists('mod_yourmodule_data');
    
    return [
        'status' => 'success',
        'description' => 'Tables removed',
    ];
}
```

## Best Practices

1. **Use mod_ prefix** - All addon module tables must use mod_ prefix
2. **Always use indexes** - Add indexes for foreign keys and frequent queries
3. **Use appropriate types** - Match column types to data
4. **Handle nullable** - Mark optional columns as nullable
5. **Use transactions** - Wrap schema changes in transactions when possible

## Related Documentation

- [WHMCS Capsule Queries](/docs/whmcs-capsule-queries.md)
- [WHMCS Database Migrations](/docs/whmcs-database-migrations.md)