# WHMCS Upgrade Path Workflow

## Purpose

Comprehensive guide to upgrading WHMCS safely, including version compatibility checks, pre-upgrade planning, database migrations, module compatibility verification, and rollback procedures.

## Prerequisites

- Current WHMCS version information
- SSH/FTP access to WHMCS installation
- Full backup capability
- Module compatibility list
- Staging environment

## Workflow Steps

### Step 1: Pre-Upgrade Assessment

Gather current system information:

```php
// Upgrade assessment script

class WHMCSUpgradeAssessment
{
    private $currentVersion;
    private $targetVersion;
    
    public function __construct(string $targetVersion)
    {
        $this->currentVersion = $this->getCurrentVersion();
        $this->targetVersion = $targetVersion;
    }
    
    /**
     * Get current WHMCS version
     */
    public function getCurrentVersion(): string
    {
        $versionFile = ROOTDIR . '/version.php';
        
        if (file_exists($versionFile)) {
            require_once $versionFile;
            return $vers ?? 'unknown';
        }
        
        // Fallback: check database
        $version = Capsule::table('tblconfiguration')
            ->where('setting', 'Version')
            ->value('value');
        
        return $version ?? 'unknown';
    }
    
    /**
     * Perform full upgrade assessment
     */
    public function assess(): array
    {
        return [
            'current_version' => $this->currentVersion,
            'target_version' => $this->targetVersion,
            'compatibility' => $this->checkCompatibility(),
            'breaking_changes' => $this->getBreakingChanges(),
            'module_compatibility' => $this->checkModuleCompatibility(),
            'database_status' => $this->checkDatabaseStatus(),
            'disk_space' => $this->checkDiskSpace(),
            'recommendations' => $this->generateRecommendations(),
        ];
    }
    
    /**
     * Check version compatibility
     */
    private function checkCompatibility(): string
    {
        $current = version_compare($this->currentVersion, '8.0.0') >= 0 ? 'v8' : 'v7';
        $target = version_compare($this->targetVersion, '8.0.0') >= 0 ? 'v8' : 'v7';
        
        if ($current === $target) {
            return 'compatible'; // Minor/patch upgrade
        }
        
        // Major version upgrade
        return $this->checkMajorVersionCompatibility($current, $target);
    }
    
    /**
     * Check module compatibility
     */
    private function checkModuleCompatibility(): array
    {
        $modules = [];
        
        // Check provisioning modules
        $serverModules = glob(ROOTDIR . '/modules/servers/*');
        foreach ($serverModules as $modulePath) {
            $moduleName = basename($modulePath);
            $modules[$moduleName] = $this->checkModuleVersion($moduleName, 'servers');
        }
        
        // Check payment gateways
        $gateways = glob(ROOTDIR . '/modules/gateways/*.php');
        foreach ($gateways as $gatewayPath) {
            $gatewayName = basename($gatewayPath, '.php');
            $modules[$gatewayName] = $this->checkModuleVersion($gatewayName, 'gateways');
        }
        
        // Check addons
        $addons = glob(ROOTDIR . '/modules/addons/*');
        foreach ($addons as $addonPath) {
            $addonName = basename($addonPath);
            $modules[$addonName] = $this->checkModuleVersion($addonName, 'addons');
        }
        
        return $modules;
    }
    
    /**
     * Check individual module compatibility
     */
    private function checkModuleVersion(string $moduleName, string $type): array
    {
        $moduleVersion = null;
        $isCompatible = true;
        $issues = [];
        
        switch ($type) {
            case 'servers':
                $versionFile = ROOTDIR . "/modules/servers/{$moduleName}/version.php";
                break;
            case 'gateways':
                $versionFile = ROOTDIR . "/modules/gateways/{$moduleName}.php";
                break;
            case 'addons':
                $versionFile = ROOTDIR . "/modules/addons/{$moduleName}/version.php";
                break;
        }
        
        if (file_exists($versionFile)) {
            include $versionFile;
            $moduleVersion = $version ?? null;
        }
        
        // Check for known incompatibilities
        $knownIssues = $this->getKnownIncompatibilities();
        
        if (isset($knownIssues[$this->targetVersion][$moduleName])) {
            $isCompatible = false;
            $issues[] = $knownIssues[$this->targetVersion][$moduleName];
        }
        
        return [
            'version' => $moduleVersion,
            'compatible' => $isCompatible,
            'issues' => $issues,
        ];
    }
    
    /**
     * Get breaking changes between versions
     */
    private function getBreakingChanges(): array
    {
        $breakingChanges = [
            '8.0' => [
                'PHP 7.4 no longer supported (requires 8.0+)',
                'Legacy template system removed',
                'Some hook priorities changed',
                'MySQL 5.7 minimum requirement',
                'Removed deprecated API functions',
            ],
            '8.1' => [
                'Additional PHP 8.0 requirements',
                'Changed session handling',
                'Updated jQuery version',
            ],
            '8.2' => [
                'PHP 8.1 required',
                'New authentication requirements',
            ],
        ];
        
        $changes = [];
        $currentMajor = (int) substr($this->currentVersion, 0, 1);
        $targetMajor = (int) substr($this->targetVersion, 0, 1);
        
        for ($v = $currentMajor + 1; $v <= $targetMajor; $v++) {
            $vKey = (string) $v . '.0';
            if (isset($breakingChanges[$vKey])) {
                $changes[$vKey] = $breakingChanges[$vKey];
            }
        }
        
        return $changes;
    }
    
    /**
     * Check database status
     */
    private function checkDatabaseStatus(): array
    {
        $issues = [];
        
        // Check for pending migrations
        $pendingTables = $this->getPendingTables();
        if (!empty($pendingTables)) {
            $issues[] = 'Database schema not fully updated';
        }
        
        // Check for orphaned records
        $orphanedServices = Capsule::table('tblhosting')
            ->leftJoin('tblclients', 'tblclients.id', '=', 'tblhosting.userid')
            ->whereNull('tblclients.id')
            ->count();
        
        if ($orphanedServices > 0) {
            $issues[] = "Found {$orphanedServices} orphaned service records";
        }
        
        return [
            'status' => empty($issues) ? 'healthy' : 'needs_attention',
            'issues' => $issues,
        ];
    }
    
    /**
     * Check available disk space
     */
    private function checkDiskSpace(): array
    {
        $bytesAvailable = disk_free_space(ROOTDIR);
        $bytesTotal = disk_total_space(ROOTDIR);
        $bytesUsed = $bytesTotal - $bytesAvailable;
        $percentUsed = ($bytesUsed / $bytesTotal) * 100;
        
        $requiredMB = 500; // Minimum recommended space
        $availableMB = $bytesAvailable / (1024 * 1024);
        
        return [
            'available_mb' => round($availableMB, 2),
            'percent_used' => round($percentUsed, 2),
            'sufficient' => $availableMB >= $requiredMB,
        ];
    }
}
```

### Step 2: Backup Before Upgrade

Create comprehensive backup:

```bash
#!/bin/bash
# scripts/pre-upgrade-backup.sh

WHMCS_ROOT="/var/www/whmcs"
BACKUP_ROOT="/backups/whmcs"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="${BACKUP_ROOT}/pre_upgrade_${TIMESTAMP}"

mkdir -p "$BACKUP_DIR"

echo "Starting pre-upgrade backup..."
echo "Backup location: $BACKUP_DIR"

# 1. Backup database
echo "Backing up database..."
DB_NAME=$(grep "db_name" "$WHMCS_ROOT/includes/config.php" | cut -d"'" -f4)
DB_USER=$(grep "db_username" "$WHMCS_ROOT/includes/config.php" | cut -d"'" -f4)
DB_PASS=$(grep "db_password" "$WHMCS_ROOT/includes/config.php" | cut -d"'" -f4)

mysqldump -u "$DB_USER" -p"$DB_PASS" "$DB_NAME" | gzip > "$BACKUP_DIR/database.sql.gz"

if [ $? -eq 0 ]; then
    echo "Database backup completed"
else
    echo "ERROR: Database backup failed"
    exit 1
fi

# 2. Backup files
echo "Backing up WHMCS files..."
rsync -avz --exclude='storage/logs/*' \
           --exclude='storage/cache/*' \
           --exclude='storage/uploads/*' \
           --exclude='templates_c/*' \
           "$WHMCS_ROOT/" "$BACKUP_DIR/files/"

if [ $? -eq 0 ]; then
    echo "File backup completed"
else
    echo "ERROR: File backup failed"
    exit 1
fi

# 3. Backup configuration
echo "Backing up configuration files..."
cp "$WHMCS_ROOT/includes/config.php" "$BACKUP_DIR/config.php"
cp "$WHMCS_ROOT/includes/dbconnect.php" "$BACKUP_DIR/dbconnect.php" 2>/dev/null || true

# 4. Create backup manifest
cat > "$BACKUP_DIR/manifest.txt" << EOF
WHMCS Pre-Upgrade Backup
========================
Date: $(date)
Timestamp: $TIMESTAMP
WHMCS Version: $(grep "version" "$WHMCS_ROOT/version.php" | cut -d"'" -f4)
Backup Contents:
  - database.sql.gz (Complete database dump)
  - files/ (WHMCS file system excluding caches)
  - config.php (Main configuration)
  - dbconnect.php (Database connection)
EOF

# 5. Verify backup integrity
echo "Verifying backup integrity..."
if [ $(gzip -l "$BACKUP_DIR/database.sql.gz" | tail -1 | awk '{print $2}') -gt 0 ]; then
    echo "Backup verification passed"
else
    echo "ERROR: Backup verification failed"
    exit 1
fi

# 6. Upload to remote storage (if configured)
if [ -n "$REMOTE_BACKUP_ENABLED" ]; then
    echo "Uploading backup to remote storage..."
    rclone copy "$BACKUP_DIR" "s3:bucket-name/whmcs-backups/" --progress
fi

echo "Pre-upgrade backup completed successfully!"
echo "Backup location: $BACKUP_DIR"
```

### Step 3: Execute Upgrade

Perform the upgrade:

```bash
#!/bin/bash
# scripts/upgrade-whmcs.sh

set -e

TARGET_VERSION="${1:-latest}"
WHMCS_ROOT="/var/www/whmcs"

echo "Starting WHMCS upgrade to version: $TARGET_VERSION"
echo "Timestamp: $(date)"

# 1. Put site in maintenance mode
echo "Enabling maintenance mode..."
cat > "$WHMCS_ROOT/includes/maintenance.php" << 'EOF'
<?php
// Maintenance mode active
// Upgrade in progress
EOF

# 2. Clear sessions
echo "Clearing sessions..."
rm -rf "$WHMCS_ROOT/storage/sessions/*"

# 3. Download new version
echo "Downloading WHMCS $TARGET_VERSION..."
cd /tmp
if [ "$TARGET_VERSION" = "latest" ]; then
    # Download latest from WHMCS
    curl -sL "https://download.whmcs.com/?action=download&version=latest" -o whmcs.zip
else
    curl -sL "https://download.whmcs.com/?action=download&version=$TARGET_VERSION" -o whmcs.zip
fi

# 4. Extract new files
echo "Extracting new files..."
rm -rf whmcs_new
unzip -q whmcs.zip -d whmcs_new

# 5. Update WHMCS files
echo "Updating WHMCS files..."
rsync -avz --exclude='includes/config.php' \
           --exclude='includes/dbconnect.php' \
           --exclude='storage' \
           --exclude='attachments' \
           --exclude='crons' \
           "whmcs_new/" "$WHMCS_ROOT/"

# 6. Set permissions
echo "Setting permissions..."
chown -R www-data:www-data "$WHMCS_ROOT"
find "$WHMCS_ROOT" -type f -exec chmod 644 {} \;
find "$WHMCS_ROOT" -type d -exec chmod 755 {} \;
chmod 640 "$WHMCS_ROOT/includes/config.php"

# 7. Run database updates
echo "Running database updates..."
cd "$WHMCS_ROOT"
php cron.php -o updates

# 8. Clear all caches
echo "Clearing caches..."
php artisan cache:clear
rm -rf "$WHMCS_ROOT/templates_c/*"

# 9. Verify installation
echo "Verifying upgrade..."
php -r "
require 'includes/init.php';
echo 'WHMCS Version: ' . App::VERSION . PHP_EOL;
echo 'Upgrade verification: ' . (App::VERSION ? 'PASSED' : 'FAILED') . PHP_EOL;
"

# 10. Disable maintenance mode
echo "Disabling maintenance mode..."
rm -f "$WHMCS_ROOT/includes/maintenance.php"

echo "Upgrade completed successfully!"
echo "New version: $(grep "version" "$WHMCS_ROOT/version.php" | cut -d"'" -f4)"
```

### Step 4: Post-Upgrade Verification

Verify upgrade success:

```php
// scripts/post-upgrade-verify.php

class PostUpgradeVerifier
{
    /**
     * Verify upgrade was successful
     */
    public function verify(): array
    {
        $results = [
            'success' => true,
            'checks' => [],
        ];
        
        // Check 1: Version correct
        $results['checks']['version'] = $this->verifyVersion();
        
        // Check 2: Database up to date
        $results['checks']['database'] = $this->verifyDatabase();
        
        // Check 3: Modules loaded
        $results['checks']['modules'] = $this->verifyModules();
        
        // Check 4: Templates work
        $results['checks']['templates'] = $this->verifyTemplates();
        
        // Check 5: API accessible
        $results['checks']['api'] = $this->verifyAPI();
        
        // Check 6: Cron jobs work
        $results['checks']['cron'] = $this->verifyCron();
        
        // Overall success
        $results['success'] = !in_array(false, array_column($results['checks'], 'passed'));
        
        return $results;
    }
    
    private function verifyVersion(): array
    {
        $version = App::VERSION;
        $expectedVersion = get_config('Version');
        
        $passed = version_compare($version, $expectedVersion) >= 0;
        
        return [
            'name' => 'Version Check',
            'passed' => $passed,
            'details' => "Expected: {$expectedVersion}, Current: {$version}",
        ];
    }
    
    private function verifyDatabase(): array
    {
        try {
            // Check for pending schema changes
            $pending = Capsule::select("
                SELECT COUNT(*) as count 
                FROM information_schema.tables 
                WHERE table_schema = DATABASE() 
                AND TABLE_NAME LIKE 'tmp_%'
            ");
            
            $passed = $pending[0]->count == 0;
            
            return [
                'name' => 'Database Schema',
                'passed' => $passed,
                'details' => $passed ? 'All migrations applied' : 'Pending migrations',
            ];
        } catch (Exception $e) {
            return [
                'name' => 'Database Schema',
                'passed' => false,
                'details' => $e->getMessage(),
            ];
        }
    }
    
    private function verifyModules(): array
    {
        $issues = [];
        
        // Check if modules can be loaded
        $modules = glob(ROOTDIR . '/modules/servers/*');
        
        foreach ($modules as $modulePath) {
            $moduleName = basename($modulePath);
            
            try {
                $module = Module::factory($moduleName);
                
                if (!$module) {
                    $issues[] = "Module {$moduleName} failed to load";
                }
            } catch (Exception $e) {
                $issues[] = "Module {$moduleName}: " . $e->getMessage();
            }
        }
        
        return [
            'name' => 'Modules',
            'passed' => empty($issues),
            'details' => empty($issues) ? 'All modules loaded successfully' : implode(', ', $issues),
        ];
    }
    
    private function verifyAPI(): array
    {
        try {
            $testParams = [
                'action' => 'GetConfiguration',
                'api_key' => ADMIN_API_KEY,
            ];
            
            $response = localAPI($testParams['action'], []);
            
            $passed = isset($response['result']) && $response['result'] === 'success';
            
            return [
                'name' => 'API Access',
                'passed' => $passed,
                'details' => $passed ? 'API responding correctly' : 'API error',
            ];
        } catch (Exception $e) {
            return [
                'name' => 'API Access',
                'passed' => false,
                'details' => $e->getMessage(),
            ];
        }
    }
}
```

### Step 5: Rollback Procedures

Implement safe rollback:

```bash
#!/bin/bash
# scripts/rollback-upgrade.sh

set -e

BACKUP_DIR="${1:-}"
WHMCS_ROOT="/var/www/whmcs"

if [ -z "$BACKUP_DIR" ]; then
    echo "Usage: $0 <backup_directory>"
    echo "Available backups:"
    ls -la /backups/whmcs/pre_upgrade_*/
    exit 1
fi

if [ ! -d "$BACKUP_DIR" ]; then
    echo "ERROR: Backup directory not found: $BACKUP_DIR"
    exit 1
fi

echo "Starting rollback from: $BACKUP_DIR"
echo "Timestamp: $(date)"

# 1. Enable maintenance mode
echo "Enabling maintenance mode..."
cat > "$WHMCS_ROOT/includes/maintenance.php" << 'EOF'
<?php
// Maintenance mode active
// Rollback in progress
EOF

# 2. Restore database
echo "Restoring database..."
DB_NAME=$(grep "db_name" "$WHMCS_ROOT/includes/config.php" | cut -d"'" -f4)
DB_USER=$(grep "db_username" "$WHMCS_ROOT/includes/config.php" | cut -d"'" -f4)
DB_PASS=$(grep "db_password" "$WHMCS_ROOT/includes/config.php" | cut -d"'" -f4)

# Drop all tables and restore
mysql -u "$DB_USER" -p"$DB_PASS" "$DB_NAME" << 'EOSQL'
SET FOREIGN_KEY_CHECKS = 0;
SET GROUP_CONCAT_MAX_LEN = 32768;
SET @tables = NULL;
SELECT GROUP_CONCAT(table_name) INTO @tables 
FROM information_schema.tables 
WHERE table_schema = DATABASE();
SET @tables = CONCAT('DROP TABLE IF EXISTS ', @tables);
PREPARE stmt FROM @tables;
EXECUTE stmt;
DEALLOCATE PREPARE stmt;
SET FOREIGN_KEY_CHECKS = 1;
EOSQL

gunzip < "$BACKUP_DIR/database.sql.gz" | mysql -u "$DB_USER" -p"$DB_PASS" "$DB_NAME"

echo "Database restored"

# 3. Restore files
echo "Restoring files..."
rm -rf "$WHMCS_ROOT.broken"
mv "$WHMCS_ROOT" "$WHMCS_ROOT.broken"
rsync -avz "$BACKUP_DIR/files/" "$WHMCS_ROOT/"
rm -rf "$WHMCS_ROOT.broken"

# 4. Restore configuration
echo "Restoring configuration..."
cp "$BACKUP_DIR/config.php" "$WHMCS_ROOT/includes/config.php"

# 5. Set permissions
echo "Setting permissions..."
chown -R www-data:www-data "$WHMCS_ROOT"
find "$WHMCS_ROOT" -type f -exec chmod 644 {} \;
find "$WHMCS_ROOT" -type d -exec chmod 755 {} \;
chmod 640 "$WHMCS_ROOT/includes/config.php"

# 6. Clear caches
echo "Clearing caches..."
rm -rf "$WHMCS_ROOT/storage/cache/*"
rm -rf "$WHMCS_ROOT/templates_c/*"

# 7. Disable maintenance mode
echo "Disabling maintenance mode..."
rm -f "$WHMCS_ROOT/includes/maintenance.php"

# 8. Verify rollback
echo "Verifying rollback..."
php -r "
require 'includes/init.php';
echo 'WHMCS Version: ' . App::VERSION . PHP_EOL;
"

echo "Rollback completed successfully!"
echo "Version restored to: $(cat $BACKUP_DIR/manifest.txt | grep 'WHMCS Version' | cut -d: -f2)"
```

## Best Practices

1. **Always backup first** - Before any upgrade attempt
2. **Test on staging** - Verify on non-production first
3. **Check module compatibility** - Update modules before WHMCS
4. **Read release notes** - Understand what's changing
5. **Schedule maintenance** - Do during low-traffic periods
6. **Monitor closely** - Watch for issues post-upgrade
7. **Have rollback ready** - Know how to undo
8. **Document changes** - Note any customizations affected

## Common Pitfalls to Avoid

1. **Skipping backups** - No way to recover
2. **Outdated modules** - Causes compatibility issues
3. **Major version jumps** - Upgrade incrementally
4. **Ignoring warnings** - System will warn you
5. **Not testing first** - Production will reveal bugs
6. **Forgetting customizations** - Templates and hooks affected
7. **Missing PHP version** - Wrong PHP causes errors
8. **Rushing post-upgrade** - Take time to verify
