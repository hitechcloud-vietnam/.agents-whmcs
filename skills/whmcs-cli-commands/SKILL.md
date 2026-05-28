# WHMCS Module CLI Commands Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Guide for building CLI commands for WHMCS modules.

## When to Use

- Creating command-line tools
- Batch processing scripts
- Automated maintenance tasks

## CLI Patterns

```php
<?php
// modules/addons/{module}/cli.php
if (php_sapi_name() !== 'cli') {
    die("CLI only");
}

// Load WHMCS
require_once '/path/to/whmcs/init.php';

$command = $argv[1] ?? 'help';

switch ($command) {
    case 'sync':
        syncAllServices();
        break;
    case 'cleanup':
        cleanupOldData($argv[2] ?? 30);
        break;
    case 'report':
        generateReport($argv[2] ?? 'daily');
        break;
    default:
        showHelp();
}

function syncAllServices(): void {
    $services = Capsule::table('tblhosting')
        ->where('domainstatus', 'Active')
        ->where('module', '{module}')
        ->get();

    foreach ($services as $service) {
        echo "Syncing service {$service->id}... ";
        $result = localAPI('ModuleSuspend', ['serviceid' => $service->id]);
        echo ($result['success'] ?? false) ? "OK\n" : "FAILED\n";
    }
}

function cleanupOldData(int $days): void {
    $cutoff = date('Y-m-d H:i:s', strtotime("-$days days"));

    $deleted = Capsule::table('mod_{module}_logs')
        ->where('created_at', '<', $cutoff)
        ->where('processed', 1)
        ->delete();

    echo "Deleted $deleted old records\n";
}

function generateReport(string $type): void {
    // Generate reports based on type
}

function showHelp(): void {
    echo "Usage: php cli.php <command>\n";
    echo "Commands:\n";
    echo "  sync          - Sync all services\n";
    echo "  cleanup <days> - Clean up old data\n";
    echo "  report <type> - Generate report\n";
}
```

### Cron Entry
```bash
# /etc/cron.d/whmcs-module
0 2 * * * cd /var/www/html && php modules/addons/{module}/cli.php cleanup 30 >> /var/log/whmcs-module-cleanup.log 2>&1
```

---

**Related Skills:**
- whmcs-cron-automation
- whmcs-reporting
- whmcs-deployment
