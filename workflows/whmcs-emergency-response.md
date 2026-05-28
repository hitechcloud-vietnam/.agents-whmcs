# WHMCS Emergency Response Workflow
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Handle module-related emergencies and incidents in WHMCS.

## Emergency Levels

| Level | Description | Response Time | Examples |
|-------|-------------|---------------|----------|
| Critical | Complete service outage | 15 minutes | Payment gateway down, module crashes |
| High | Major functionality broken | 1 hour | API failures, data sync issues |
| Medium | Minor features affected | 4 hours | UI glitches, slow performance |
| Low | Cosmetic issues | 24 hours | Display errors, minor bugs |

## Incident Response Process

### 1. Initial Assessment
```php
<?php
// Emergency diagnostic hook
add_hook('AdminAreaPage', 1, function() {
    if (hasPermission('System Health')) {
        echo '<div class="alert alert-warning">';
        echo '<strong>Module Health Check</strong><br>';

        // Check each critical module
        $modules = ['gateway', 'registrar', 'addon'];
        foreach ($modules as $module) {
            $status = checkModuleHealth($module);
            $icon = $status ? '<i class="fa fa-check text-success"></i>' : '<i class="fa fa-times text-danger"></i>';
            echo "{$icon} {$module}<br>";
        }
        echo '</div>';
    }
});

function checkModuleHealth(string $module): bool {
    try {
        $tables = [
            'gateway' => ['mod_gateway_log', 'mod_gateway_config'],
            'registrar' => ['mod_registrar_domains'],
            'addon' => ['mod_addon_config'],
        ];

        foreach ($tables[$module] ?? [] as $table) {
            if (!Capsule::schema()->hasTable($table)) {
                return false;
            }
        }
        return true;
    } catch (\Exception $e) {
        logActivity("Module health check failed: " . $e->getMessage());
        return false;
    }
}
```

### 2. Quick Mitigation
```bash
#!/bin/bash
# emergency-fix.sh
MODULE_NAME="mymodule"

# Step 1: Disable module (prevent further issues)
/opt/whmcs/configcomposer.php
# Set module active = 0

# Step 2: Clear all caches
php /var/www/whmcs/admin/clearcache.php
rm -rf /var/www/whmcs/templates_c/*
rm -rf /var/www/whmcs/vendor/whmcs/whmcs/cache/*

# Step 3: Check recent changes
git log --oneline -10

# Step 4: Restore from backup if needed
./deploy.sh restore ${BACKUP_DATE}

# Step 5: Notify stakeholders
./notify.sh "Module {$MODULE_NAME} emergency fix completed"
```

### 3. Data Recovery
```php
<?php
class EmergencyRecovery {
    public function restoreModuleData(int $backupId): bool {
        $backup = Capsule::table('mod_backups')
            ->where('id', $backupId)
            ->first();

        if (!$backup) {
            throw new \Exception("Backup not found");
        }

        // Restore module configuration
        $config = json_decode($backup->config_data, true);
        Capsule::table('mod_mymodule_config')
            ->truncate();
        foreach ($config as $row) {
            Capsule::table('mod_mymodule_config')->insert($row);
        }

        // Log recovery
        logActivity("Module data restored from backup #{$backupId}");

        return true;
    }

    public function repairTable(string $table): bool {
        try {
            Capsule::statement("REPAIR TABLE {$table}");
            logActivity("Table {$table} repaired successfully");
            return true;
        } catch (\Exception $e) {
            logActivity("Table repair failed for {$table}: " . $e->getMessage());
            return false;
        }
    }
}
```

### 4. Communication Template
```php
<?php
function sendIncidentNotification(string $level, string $title, string $description): void {
    $admins = Capsule::table('tbladmins')
        ->where('disabled', 0)
        ->where('roleid', 1) // Administrator role
        ->get();

    foreach ($admins as $admin) {
        sendEmailTemplate([
            'id' => $admin->email,
            'template' => 'incident_notification',
            'vars' => [
                'incident_level' => strtoupper($level),
                'incident_title' => $title,
                'incident_description' => $description,
                'incident_time' => date('Y-m-d H:i:s T'),
                'affected_services' => getAffectedServices(),
                'action_taken' => getActionTaken(),
            ],
        ]);
    }
}

// Incident resolved notification
function sendResolutionNotification(int $incidentId): void {
    $incident = Capsule::table('mod_incidents')
        ->where('id', $incidentId)
        ->first();

    sendEmailTemplate([
        'template' => 'incident_resolved',
        'vars' => [
            'incident_id' => $incidentId,
            'title' => $incident->title,
            'duration' => calculateDuration($incident->created_at, $incident->resolved_at),
            'root_cause' => $incident->root_cause,
            'preventive_measures' => $incident->preventive_measures,
        ],
    ]);
}
```

## Rollback Procedures

### Module Rollback
```php
<?php
class ModuleRollback {
    private string $backupDir = '/backup/whmcs/modules';

    public function rollbackToVersion(string $module, string $version): bool {
        $backupPath = "{$this->backupDir}/{$module}/{$version}";

        if (!is_dir($backupPath)) {
            throw new \Exception("Backup not found for version {$version}");
        }

        // Backup current state first
        $currentBackup = "{$this->backupDir}/{$module}/emergency-" . date('Ymd-His');
        $this->backupCurrent($module, $currentBackup);

        // Remove current module
        $modulePath = \WHMCS\Application::getInstance()->getRootDir()
            . "/modules/servers/{$module}";
        $this->removeDir($modulePath);

        // Restore from backup
        $this->copyDir($backupPath, $modulePath);

        // Set permissions
        chown($modulePath, 'www-data');
        chmod("{$modulePath}/*.php", 0644);

        logActivity("Module {$module} rolled back to version {$version}");

        return true;
    }

    public function getAvailableVersions(string $module): array {
        $backups = glob("{$this->backupDir}/{$module}/*");
        return array_map('basename', $backups ?? []);
    }
}
```

## Post-Incident Review

### Root Cause Analysis
```php
<?php
function createIncidentReport(int $incidentId, array $data): void {
    Capsule::table('mod_incident_reports')->insert([
        'incident_id' => $incidentId,
        'timeline' => json_encode($data['timeline']),
        'root_cause' => $data['root_cause'],
        'impact' => json_encode($data['impact']),
        'lessons_learned' => $data['lessons_learned'],
        'action_items' => json_encode($data['action_items']),
        'created_by' => $_SESSION['adminid'],
        'created_at' => date('Y-m-d H:i:s'),
    ]);

    // Create follow-up tasks
    foreach ($data['action_items'] as $item) {
        createTodo([
            'title' => "Post-incident: " . $item['task'],
            'due_date' => $item['due_date'],
            'assigned_to' => $item['assignee'],
            'priority' => 'high',
        ]);
    }
}
```

## Checklist

```
Initial Response (0-15 min):
□ Assess severity level
□ Notify incident response team
□ Document initial findings
□ Implement immediate mitigation

Investigation (15-60 min):
□ Gather logs and metrics
□ Identify affected components
□ Determine root cause
□ Assess data integrity

Resolution (1-4 hours):
□ Apply fix or rollback
□ Verify fix effectiveness
□ Test related functionality
□ Update stakeholders

Recovery (4-24 hours):
□ Monitor for recurrence
□ Restore full service
□ Send resolution notification
□ Begin post-incident review
```

---

**Related Skills:**
- whmcs-backup-restore
- whmcs-logging
- whmcs-deployment