# WHMCS Migration Debug Workflow

## Overview
This workflow guides you through debugging database migration issues.

## Prerequisites
- Migration files
- Database access

## Step-by-Step Guide

### Step 1: Check Migration Status
```bash
# List pending migrations
php artisan migrate:status

# Run with verbose output
php artisan migrate --pretend --verbose
```

### Step 2: Test Migration Manually
```php
// In migration file
public function up()
{
    \Schema::create('mod_yourmodule_table', function ($table) {
        $table->increments('id');
        $table->string('name');
        $table->timestamps();
    });

    // Log completion
    logActivity('Migration mod_yourmodule_table created');
}
```

### Step 3: Rollback Migration
```bash
# Rollback last migration
php artisan migrate:rollback

# Rollback specific migration
php artisan migrate:rollback --path=/modules/addons/yourmodule/migrations/
```

### Step 4: Common Migration Issues
```php
// Issue: Table already exists
// Fix: Use createIfNotExists() or check

// Issue: Column already exists
// Fix: Use hasColumn() check

// Issue: Foreign key constraint
// Fix: Drop child records first
```

## Migration Debug Checklist

### Investigation
- [ ] Migration status checked
- [ ] Errors logged
- [ ] Dependencies verified
- [ ] Data reviewed

### Resolution
- [ ] Migration fixed
- [ ] Rollback tested
- [ ] Data migrated
- [ ] Indexes created
