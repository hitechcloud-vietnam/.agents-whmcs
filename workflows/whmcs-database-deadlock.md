# WHMCS Database Deadlock Debug Workflow

## Overview
This workflow guides you through debugging database deadlocks.

## Prerequisites
- MySQL/MariaDB access
- Process list access

## Step-by-Step Guide

### Step 1: Identify Deadlocks
```sql
-- Check for locked tables
SHOW OPEN TABLES WHERE In_use > 0;

-- Check process list
SHOW FULL PROCESSLIST;

-- Get deadlock info
SHOW ENGINE INNODB STATUS;
```

### Step 2: Enable Deadlock Logging
```sql
-- Enable InnoDB monitor
SET GLOBAL innodb_status_output = ON;
SET GLOBAL innodb_status_output_locks = ON;
```

### Step 3: Analyze Deadlock
```php
// Catch deadlock exceptions
try {
    \WHMCS\Database\Capsule::transaction(function() {
        // Your transaction
        $this->updateRecord($id, $data);
    });
} catch (\Illuminate\Database\QueryException $e) {
    if (strpos($e->getMessage(), 'Deadlock') !== false) {
        $this->log->error("Deadlock occurred, retrying", [
            'error' => $e->getMessage(),
        ]);
        // Retry logic
        return $this->retryTransaction($id, $data);
    }
    throw $e;
}
```

### Step 4: Prevent Deadlocks
```php
// Always lock in same order
// BAD: Lock A then B, then B then A
// GOOD: Always lock A then B

// Use SELECT ... FOR UPDATE consistently
$record = Model::where('id', $id)->lockForUpdate()->first();

// Reduce lock duration
// BAD:
$record = Model::find($id);
process(); // Long operation
$record->save();

// GOOD:
Model::where('id', $id)->update(['field' => 'value']);
```

## Deadlock Debug Checklist

### Investigation
- [ ] Process list examined
- [ ] Deadlock information retrieved
- [ ] Tables involved identified
- [ ] Queries analyzed

### Resolution
- [ ] Lock order fixed
- [ ] Transaction size reduced
- [ ] Retry logic added
- [ ] Indexes optimized
