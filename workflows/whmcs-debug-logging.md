# WHMCS Debug Logging Setup Workflow

## Overview
This workflow guides you through setting up debug logging for WHMCS modules.

## Prerequisites
- WHMCS installation
- Module development environment

## Step-by-Step Guide

### Step 1: Configure Debug Mode
```php
// In configuration.php or module bootstrap
define('WHMCS_DEBUG', true);
ini_set('display_errors', 1);
error_reporting(E_ALL);
```

### Step 2: Create Logger Class
```php
// lib/Logger.php
<?php
namespace Vendor\Module;

class Logger
{
    private static string $logPath;
    private static bool $enabled = false;

    public static function init(string $logPath, bool $enabled = true): void
    {
        self::$logPath = $logPath;
        self::$enabled = $enabled;
    }

    public static function debug(string $message, array $context = []): void
    {
        self::log('DEBUG', $message, $context);
    }

    public static function info(string $message, array $context = []): void
    {
        self::log('INFO', $message, $context);
    }

    public static function error(string $message, array $context = []): void
    {
        self::log('ERROR', $message, $context);
    }

    private static function log(string $level, string $message, array $context): void
    {
        if (!self::$enabled) {
            return;
        }

        $timestamp = date('Y-m-d H:i:s');
        $contextJson = !empty($context) ? json_encode($context) : '';
        $logLine = "[$timestamp] [$level] $message $contextJson" . PHP_EOL;

        file_put_contents(self::$logPath, $logLine, FILE_APPEND);
    }
}
```

### Step 3: Use Logger in Module
```php
// In your module
use Vendor\Module\Logger;

Logger::init(__DIR__ . '/../logs/module.log', true);

function processClientSync(int $clientId): void
{
    Logger::debug("Starting client sync", ['client_id' => $clientId]);

    try {
        // Process logic
        Logger::info("Client sync completed", ['client_id' => $clientId]);
    } catch (Exception $e) {
        Logger::error("Client sync failed", [
            'client_id' => $clientId,
            'error' => $e->getMessage(),
        ]);
        throw $e;
    }
}
```

### Step 4: View Logs
```bash
# Tail log file
tail -f /var/www/whmcs/storage/logs/module.log

# Search for errors
grep "ERROR" /var/www/whmcs/storage/logs/module.log

# Filter by context
grep '"client_id": 123' /var/www/whmcs/storage/logs/module.log
```

## Debug Logging Checklist

### Configuration
- [ ] Debug mode enabled
- [ ] Log path configured
- [ ] Log rotation set up

### Implementation
- [ ] Logger class created
- [ ] Logging added to key functions
- [ ] Context data included
- [ ] Sensitive data excluded

### Monitoring
- [ ] Logs viewable
- [ ] Error alerts configured
- [ ] Log analysis automated
