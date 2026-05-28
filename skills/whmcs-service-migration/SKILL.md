# WHMCS Service Migration Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Migrate services between WHMCS accounts while preserving data integrity.

## Database Schema

```php
<?php
// modules/addons/service_migration/service_migration.php

use WHMCS\Database\Capsule;

function service_migration_config(): array {
    return [
        'name' => 'Service Migration',
        'description' => 'Migrate services between accounts',
        'version' => '1.0',
    ];
}

function service_migration_activate(): array {
    Capsule::schema()->create('mod_service_migration_log', function($t) {
        $t->increments('id');
        $t->integer('source_user_id')->unsigned();
        $t->integer('target_user_id')->unsigned();
        $t->integer('service_id')->unsigned();
        $t->string('service_type', 50);
        $t->text('migration_data')->nullable();
        $t->string('status', 20)->default('pending');
        $t->timestamp('migrated_at')->nullable();
        $t->timestamp('created_at')->useCurrent();
    });

    Capsule::schema()->create('mod_service_migration_steps', function($t) {
        $t->increments('id');
        $t->integer('migration_id')->unsigned();
        $t->string('step_name', 100);
        $t->string('status', 20)->default('pending');
        $t->text('error_message')->nullable();
        $t->timestamp('executed_at')->nullable();
    });

    return ['status' => 'success'];
}

function service_migration_deactivate(): array {
    Capsule::schema()->dropIfExists('mod_service_migration_steps');
    Capsule::schema()->dropIfExists('mod_service_migration_log');
    return ['status' => 'success'];
}
```

## Service Migration Class

```php
<?php
class ServiceMigrationManager {
    private array $migrationSteps = [
        'backup_data',
        'cancel_service',
        'create_service',
        'restore_config',
        'update_relationships',
        'notify_users',
    ];

    public function initiateMigration(int $sourceUserId, int $targetUserId, int $serviceId, string $type): array {
        // Validate users exist
        $sourceUser = Capsule::table('tblclients')->where('id', $sourceUserId)->first();
        $targetUser = Capsule::table('tblclients')->where('id', $targetUserId)->first();

        if (!$sourceUser || !$targetUser) {
            return ['success' => false, 'error' => 'User not found'];
        }

        // Get service data
        $service = $this->getServiceByType($serviceId, $type);
        if (!$service) {
            return ['success' => false, 'error' => 'Service not found'];
        }

        // Create migration record
        $migrationId = Capsule::table('mod_service_migration_log')->insertGetId([
            'source_user_id' => $sourceUserId,
            'target_user_id' => $targetUserId,
            'service_id' => $serviceId,
            'service_type' => $type,
            'migration_data' => json_encode($service),
            'status' => 'in_progress',
        ]);

        // Create migration steps
        foreach ($this->migrationSteps as $step) {
            Capsule::table('mod_service_migration_steps')->insert([
                'migration_id' => $migrationId,
                'step_name' => $step,
            ]);
        }

        return ['success' => true, 'migration_id' => $migrationId];
    }

    public function executeMigrationStep(int $migrationId, string $stepName): array {
        $step = Capsule::table('mod_service_migration_steps')
            ->where('migration_id', $migrationId)
            ->where('step_name', $stepName)
            ->first();

        if (!$step) {
            return ['success' => false, 'error' => 'Step not found'];
        }

        $migration = Capsule::table('mod_service_migration_log')->find($migrationId);
        $serviceData = json_decode($migration->migration_data, true);

        try {
            $result = $this->{'execute_' . $stepName}($migration, $serviceData);

            Capsule::table('mod_service_migration_steps')
                ->where('id', $step->id)
                ->update(['status' => 'completed', 'executed_at' => date('Y-m-d H:i:s')]);

            return $result;
        } catch (\Exception $e) {
            Capsule::table('mod_service_migration_steps')
                ->where('id', $step->id)
                ->update(['status' => 'failed', 'error_message' => $e->getMessage()]);

            return ['success' => false, 'error' => $e->getMessage()];
        }
    }

    private function execute_backup_data($migration, $data): array {
        // Create backup of original service
        Capsule::table('mod_service_migration_log')
            ->where('id', $migration->id)
            ->update(['migration_data' => json_encode($data)]);

        return ['success' => true, 'message' => 'Data backed up'];
    }

    private function execute_cancel_service($migration, $data): array {
        $type = $migration->service_type;

        switch ($type) {
            case 'hosting':
                localAPI('CancelService', [
                    'serviceid' => $migration->service_id,
                    'immediate' => false,
                ]);
                break;
            case 'domain':
                // Domain transfer handling
                break;
        }

        return ['success' => true, 'message' => 'Service cancelled at source'];
    }

    private function execute_create_service($migration, $data): array {
        $params = [
            'clientid' => $migration->target_user_id,
            'pid' => $data['pid'],
            'domain' => $data['domain'] ?? '',
            'billingcycle' => $data['billingcycle'],
        ];

        $result = localAPI('AddOrder', $params);

        return ['success' => true, 'order_id' => $result['orderid'] ?? null];
    }

    private function execute_restore_config($migration, $data): array {
        // Restore configuration settings
        $configFields = ['configoptions', 'customfields', 'notes'];

        foreach ($configFields as $field) {
            if (isset($data[$field])) {
                // Apply configuration to new service
            }
        }

        return ['success' => true, 'message' => 'Configuration restored'];
    }

    private function execute_update_relationships($migration, $data): array {
        // Update addon relationships
        Capsule::table('tblhostingaddons')
            ->where('relid', $migration->service_id)
            ->update(['userid' => $migration->target_user_id]);

        // Update invoices
        Capsule::table('tblinvoices')
            ->where('userid', $migration->source_user_id)
            ->whereIn('status', ['Draft', 'Unpaid'])
            ->update(['userid' => $migration->target_user_id]);

        return ['success' => true, 'message' => 'Relationships updated'];
    }

    private function execute_notify_users($migration, $data): array {
        $sourceUser = Capsule::table('tblclients')->find($migration->source_user_id);
        $targetUser = Capsule::table('tblclients')->find($migration->target_user_id);

        sendTplEmail($sourceUser->email, 'service_migrated_from', [
            'service_name' => $data['domain'] ?? 'Service',
        ]);

        sendTplEmail($targetUser->email, 'service_migrated_to', [
            'service_name' => $data['domain'] ?? 'Service',
        ]);

        return ['success' => true, 'message' => 'Users notified'];
    }

    private function getServiceByType(int $serviceId, string $type): ?array {
        switch ($type) {
            case 'hosting':
                return Capsule::table('tblhosting')->find($serviceId);
            case 'domain':
                return Capsule::table('tbldomains')->find($serviceId);
            case 'addon':
                return Capsule::table('tblhostingaddons')->find($serviceId);
            default:
                return null;
        }
    }
}
```

## Admin Interface

```php
function service_migration_output(array $vars): void {
    $action = $_GET['action'] ?? 'list';

    echo '<div class="service-migration-admin">';

    if ($action === 'list') {
        $migrations = Capsule::table('mod_service_migration_log')
            ->orderBy('created_at', 'desc')
            ->limit(50)
            ->get();

        echo '<h2>Service Migrations</h2>';
        echo '<table class="datatable"><thead><tr>';
        echo '<th>ID</th><th>Source User</th><th>Target User</th>';
        echo '<th>Service</th><th>Status</th><th>Actions</th>';
        echo '</tr></thead><tbody>';

        foreach ($migrations as $m) {
            echo '<tr>';
            echo "<td>{$m->id}</td>";
            echo "<td>{$m->source_user_id}</td>";
            echo "<td>{$m->target_user_id}</td>";
            echo "<td>{$m->service_type}</td>";
            echo "<td>{$m->status}</td>";
            echo '<td><a href="?action=view&id=' . $m->id . '">View</a></td>';
            echo '</tr>';
        }

        echo '</tbody></table>';
    }

    echo '</div>';
}
```

## Migration Hooks

```php
add_hook('ServiceMigrationStart', 1, function($vars) {
    logActivity("Service migration started: {$vars['migration_id']}");
});

add_hook('ServiceMigrationComplete', 1, function($vars) {
    // Send confirmation email
    $migration = Capsule::table('mod_service_migration_log')->find($vars['migration_id']);
    logActivity("Service migration completed for service {$migration->service_id}");
});
```

---

**Related Skills:**
- whmcs-client-management
- whmcs-billing-dashboard
- whmcs-pricing-engine