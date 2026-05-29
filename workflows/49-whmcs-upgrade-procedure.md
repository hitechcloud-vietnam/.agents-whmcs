# WHMCS Upgrade Procedure Workflow

## Overview
This workflow provides a systematic approach to upgrading WHMCS installations.

## Step 1: Pre-Upgrade Checklist

```bash
#!/bin/bash
# pre-upgrade-check.sh

echo "=== WHMCS Pre-Upgrade Checklist ==="

# 1. Check current version
echo "Current WHMCS version:"
grep "version" /var/www/whmcs/init.php | head -1

# 2. Check PHP version
echo "PHP version:"
php -v | head -1

# 3. Check disk space
echo "Disk space:"
df -h /var/www/whmcs | tail -1

# 4. Check MySQL version
echo "MySQL version:"
mysql --version

# 5. Check for customizations
echo "Custom files:"
find /var/www/whmcs/custom -type f 2>/dev/null | wc -l

# 6. Check backup status
echo "Last backup:"
ls -la /var/www/whmcs/backups/*.tar.gz 2>/dev/null | tail -1

# 7. Test database connection
echo "Database connection:"
mysql -u whmcs -pwhmcs -e "SELECT 1" 2>/dev/null && echo "OK" || echo "FAILED"

# 8. Check module compatibility
echo "Checking module compatibility..."
for module in /var/www/whmcs/modules/addons/*; do
    if [ -f "$module/version.php" ]; then
        echo "  $(basename $module):"
        grep "version" "$module/version.php" 2>/dev/null | head -1
    fi
done
```

## Step 2: Upgrade Service

```php
<?php
// src/Service/UpgradeService.php

namespace WHMCS\Module\Addon\YourModule\Service;

use WHMCS\Database\Capsule;

class UpgradeService
{
    private $backupService;
    private $steps = [];

    public function __construct()
    {
        $this->backupService = new BackupService();
    }

    public function prepareUpgrade(string $targetVersion): array
    {
        $checks = [];

        // Check current version
        $checks['current_version'] = $this->checkCurrentVersion();
        $checks['target_version'] = $targetVersion;

        // Check system requirements
        $checks['php_version'] = $this->checkPhpVersion($targetVersion);
        $checks['mysql_version'] = $this->checkMysqlVersion();
        $checks['disk_space'] = $this->checkDiskSpace();

        // Check for pending migrations
        $checks['pending_migrations'] = $this->checkPendingMigrations();

        // Check module compatibility
        $checks['module_compatibility'] = $this->checkModuleCompatibility($targetVersion);

        // Check customizations
        $checks['custom_files'] = $this->checkCustomFiles();

        $checks['ready'] = $this->isReadyForUpgrade($checks);

        return $checks;
    }

    private function checkCurrentVersion(): string
    {
        return Capsule::config('version') ?? 'Unknown';
    }

    private function checkPhpVersion(string $targetVersion): array
    {
        $required = $this->getRequiredPhpVersion($targetVersion);
        $current = PHP_VERSION_ID;

        return [
            'required' => $required,
            'current' => PHP_VERSION,
            'compatible' => $current >= $required
        ];
    }

    private function checkMysqlVersion(): array
    {
        $result = Capsule::connection()->select('SELECT VERSION() as version');
        $version = $result[0]->version ?? 'Unknown';

        $minVersion = '5.7.0';
        return [
            'required' => $minVersion,
            'current' => $version,
            'compatible' => version_compare($version, $minVersion, '>=')
        ];
    }

    private function checkDiskSpace(): array
    {
        $free = disk_free_space('/');
        $required = 500 * 1024 * 1024; // 500MB

        return [
            'free_mb' => round($free / 1024 / 1024, 2),
            'required_mb' => round($required / 1024 / 1024, 2),
            'sufficient' => $free > $required
        ];
    }

    private function checkPendingMigrations(): array
    {
        $appliedMigrations = Capsule::table('mod_migrations')
            ->pluck('migration_name')
            ->toArray();

        $availableMigrations = $this->getAvailableMigrations();

        $pending = array_diff($availableMigrations, $appliedMigrations);

        return [
            'applied' => count($appliedMigrations),
            'pending' => count($pending),
            'pending_list' => array_values($pending)
        ];
    }

    private function checkModuleCompatibility(string $version): array
    {
        $modules = glob(dirname(__DIR__, 3) . '/modules/addons/*');
        $compatibility = [];

        foreach ($modules as $module) {
            $moduleName = basename($module);
            $compatibility[$moduleName] = [
                'compatible' => true, // Would check actual compatibility
                'version' => file_exists("$module/version.php")
                    ? trim(file_get_contents("$module/version.php"))
                    : 'Unknown'
            ];
        }

        return $compatibility;
    }

    private function checkCustomFiles(): array
    {
        $customDir = dirname(__DIR__, 3) . '/custom';
        $customFiles = [];

        if (is_dir($customDir)) {
            $iterator = new \RecursiveIteratorIterator(
                new \RecursiveDirectoryIterator($customDir)
            );

            foreach ($iterator as $file) {
                if ($file->isFile()) {
                    $customFiles[] = $file->getPathname();
                }
            }
        }

        return [
            'count' => count($customFiles),
            'files' => $customFiles
        ];
    }

    private function isReadyForUpgrade(array $checks): bool
    {
        if (!$checks['php_version']['compatible']) return false;
        if (!$checks['mysql_version']['compatible']) return false;
        if (!$checks['disk_space']['sufficient']) return false;

        return true;
    }

    private function getRequiredPhpVersion(string $targetVersion): int
    {
        // Map WHMCS versions to required PHP versions
        $phpVersions = [
            '8.0' => 80100,
            '8.1' => 80100,
            '8.2' => 80200,
            '8.3' => 80300
        ];

        return $phpVersions[$targetVersion] ?? 80100;
    }

    private function getAvailableMigrations(): array
    {
        $migrationsDir = dirname(__DIR__) . '/migrations';
        $migrations = [];

        if (is_dir($migrationsDir)) {
            $files = glob("$migrationsDir/*.php");
            foreach ($files as $file) {
                $migrations[] = basename($file, '.php');
            }
        }

        return $migrations;
    }

    public function executeUpgrade(string $targetVersion): array
    {
        $steps = [];

        // Step 1: Create backup
        $steps[] = 'Creating full backup';
        $backup = $this->backupService->createFullBackup(true);

        // Step 2: Enable maintenance mode
        $steps[] = 'Enabling maintenance mode';
        $this->enableMaintenanceMode();

        // Step 3: Download new version
        $steps[] = 'Downloading WHMCS ' . $targetVersion;
        $downloadPath = $this->downloadNewVersion($targetVersion);

        // Step 4: Extract files
        $steps[] = 'Extracting new files';
        $this->extractFiles($downloadPath);

        // Step 5: Run database migrations
        $steps[] = 'Running database migrations';
        $this->runMigrations();

        // Step 6: Clear cache
        $steps[] = 'Clearing cache';
        $this->clearCache();

        // Step 7: Disable maintenance mode
        $steps[] = 'Disabling maintenance mode';
        $this->disableMaintenanceMode();

        // Step 8: Verify upgrade
        $steps[] = 'Verifying upgrade';
        $verified = $this->verifyUpgrade($targetVersion);

        return [
            'success' => $verified,
            'backup' => $backup,
            'steps' => $steps,
            'new_version' => $targetVersion
        ];
    }

    private function enableMaintenanceMode(): void
    {
        file_put_contents(
            dirname(__DIR__, 3) . '/maintenance.html',
            '<html><body><h1>Maintenance in Progress</h1></body></html>'
        );
    }

    private function disableMaintenanceMode(): void
    {
        @unlink(dirname(__DIR__, 3) . '/maintenance.html');
    }

    private function downloadNewVersion(string $version): string
    {
        // Would download from WHMCS
        return '/tmp/whmcs_' . $version . '.zip';
    }

    private function extractFiles(string $zipPath): void
    {
        // Would extract to WHMCS directory
    }

    private function runMigrations(): void
    {
        // Would run WHMCS migrations
    }

    private function clearCache(): void
    {
        $cacheService = new CacheManagementService();
        $cacheService->clearAllCache();
    }

    private function verifyUpgrade(string $expectedVersion): bool
    {
        $actualVersion = $this->checkCurrentVersion();
        return version_compare($actualVersion, $expectedVersion, '>=');
    }
}
```

## Step 3: Rollback Plan

```php
<?php
// Rollback procedure

function rollbackToPreviousVersion(): array
{
    // Find previous backup
    $backups = glob('/var/www/whmcs/backups/whmcs_full_*.tar.gz');
    $latestBackup = end($backups);

    if (!$latestBackup) {
        return ['success' => false, 'error' => 'No backup found'];
    }

    // Restore from backup
    $backupService = new BackupService();
    $result = $backupService->restoreBackup($latestBackup);

    return $result;
}
```

## Verification Checklist

- [ ] Pre-upgrade checks implemented
- [ ] Version compatibility checking working
- [ ] System requirements validated
- [ ] Module compatibility checked
- [ ] Backup creation working
- [ ] Maintenance mode implemented
- [ ] Upgrade execution working
- [ ] Rollback procedure tested
- [ ] Post-upgrade verification working
