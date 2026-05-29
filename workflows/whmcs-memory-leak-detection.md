# WHMCS Memory Leak Detection Workflow

## Overview
This workflow guides you through detecting and fixing memory leaks in WHMCS modules.

## Prerequisites
- Memory profiling tools
- CLI access

## Step-by-Step Guide

### Step 1: Check Memory Usage
```php
// Add to module
function logMemoryUsage(string $context): void
{
    $memory = memory_get_usage(true) / 1024 / 1024;
    error_log("[$context] Memory: {$memory} MB");
}

// In your code
logMemoryUsage("Start of process");

foreach ($clients as $client) {
    // Process client
    logMemoryUsage("After client {$client->id}");
}

logMemoryUsage("End of process");
```

### Step 2: Use Memory Profiler
```bash
# Install Xdebug
pecl install xdebug

# Configure php.ini
memory_limit = 512M
xdebug.mode = profile
xdebug.output_dir = /tmp/profiles
```

### Step 3: Analyze with Webgrind
```bash
# View profiles
php webgrind/public/index.php

# Or use cachegrind tools
wget -qO- /tmp/profiles/cachegrind.out.* | head -100
```

### Step 4: Common Memory Leaks
```php
// BAD: Storing large objects in static
static $cache = [];
$cache['large_data'] = $bigArray; // Leak!

// GOOD: Unset when done
$bigArray = null;

// BAD: Event listeners not removed
$events->listen('Client.*', function($e) use ($bigObject) {
    // Keeps reference to $bigObject
});

// GOOD: Remove listeners or use weak references
$events->listen('Client.*', function($e) {
    // Don't capture external objects
});
```

## Memory Leak Detection Checklist

### Detection
- [ ] Memory monitoring added
- [ ] Peak memory tracked
- [ ] Leak patterns identified
- [ ] Profiling done

### Fixes
- [ ] References cleared
- [ ] Objects unset
- [ ] Static caches limited
- [ ] Event listeners removed

### Prevention
- [ ] Coding guidelines followed
- [ ] Memory checks in tests
- [ ] Regular profiling scheduled
