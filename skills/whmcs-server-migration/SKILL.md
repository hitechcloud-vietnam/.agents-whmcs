# WHMCS Server Migration Module Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Build modules for migrating services between servers.

## Migration Module Structure

```php
<?php
/**
 * Server Migration Module (Addon)
 * Location: modules/addons/{module}/
 */

function {module}_config(): array {
    return [
        'name' => 'Server Migration',
        'description' => 'Migration services between servers',
        'version' => '1.0',
    ];
}

function {module}_activate(): array {
    Capsule::schema()->create('mod_migration_jobs', function($t) {
        $t->increments('id');
        $t->integer('service_id')->unsigned();
        $t->integer('source_server_id')->unsigned();
        $t->integer('target_server_id')->unsigned();
        $t->string('status', 20)->default('pending');
        $t->text('migration_data');
        $t->text('error_log');
        $t->timestamp('started_at')->nullable();
        $t->timestamp('completed_at')->nullable();
        $t->timestamps();
    });

    Capsule::schema()->create('mod_migration_steps', function($t) {
        $t->increments('id');
        $t->integer('job_id')->unsigned();
        $t->string('step', 50);
        $t->string('status', 20)->default('pending');
        $t->text('output');
        $t->integer('duration')->default(0);
        $t->timestamps();
    });

    return ['status' => 'success', 'description' => 'Migration module activated'];
}

function {module}_deactivate(): array {
    Capsule::schema()->dropIfExists('mod_migration_jobs');
    Capsule::schema()->dropIfExists('mod_migration_steps');
    return ['status' => 'success'];
}
```

## Migration Pipeline

```php
<?php
class MigrationPipeline {
    private array $steps = [
        'validate',
        'backup',
        'prepare',
        'transfer',
        'configure',
        'verify',
        'cleanup',
    ];

    public function execute(int $jobId): array {
        $job = Capsule::table('mod_migration_jobs')->where('id', $jobId)->first();

        Capsule::table('mod_migration_jobs')
            ->where('id', $jobId)
            ->update([
                'status' => 'running',
                'started_at' => date('Y-m-d H:i:s'),
            ]);

        foreach ($this->steps as $step) {
            $stepId = $this->createStep($jobId, $step);
            $startTime = microtime(true);

            try {
                $result = $this->executeStep($job, $step);
                $duration = (int) ((microtime(true) - $startTime) * 1000);

                $this->completeStep($stepId, $result, $duration);

                if (!$result['success']) {
                    $this->failJob($jobId, $step, $result['error']);
                    return ['success' => false, 'failed_at' => $step];
                }
            } catch (\Exception $e) {
                $this->failStep($stepId, $e->getMessage());
                $this->failJob($jobId, $step, $e->getMessage());
                return ['success' => false, 'error' => $e->getMessage()];
            }
        }

        $this->completeJob($jobId);
        return ['success' => true];
    }

    private function executeStep($job, string $step): array {
        $method = 'step' . str_replace('_', '', ucwords($step, '_'));
        return $this->$method($job);
    }
}
```

## Migration Steps

```php
private function stepValidate($job): array {
    $sourceServer = Capsule::table('tblservers')->where('id', $job->source_server_id)->first();
    $targetServer = Capsule::table('tblservers')->where('id', $job->target_server_id)->first();

    $sourceModule = new ServerModule($sourceServer->type);
    $targetModule = new ServerModule($targetServer->type);

    $service = Capsule::table('tblhosting')->where('id', $job->service_id)->first();

    if (!$sourceModule->testConnection($sourceServer)) {
        return ['success' => false, 'error' => 'Cannot connect to source server'];
    }

    if (!$targetModule->testConnection($targetServer)) {
        return ['success' => false, 'error' => 'Cannot connect to target server'];
    }

    return ['success' => true];
}

private function stepBackup($job): array {
    $sourceServer = Capsule::table('tblservers')->where('id', $job->source_server_id)->first();
    $service = Capsule::table('tblhosting')->where('id', $job->service_id)->first();

    $backupModule = new BackupModule();
    $backupId = $backupModule->createBackup([
        'service_id' => $job->service_id,
        'server_id' => $job->source_server_id,
        'type' => 'full',
    ]);

    Capsule::table('mod_migration_jobs')
        ->where('id', $job->id)
        ->update([
            'migration_data' => json_encode(['backup_id' => $backupId]),
        ]);

    return ['success' => true, 'backup_id' => $backupId];
}

private function stepTransfer($job): array {
    $sourceServer = Capsule::table('tblservers')->where('id', $job->source_server_id)->first();
    $targetServer = Capsule::table('tblservers')->where('id', $job->target_server_id)->first();
    $service = Capsule::table('tblhosting')->where('id', $job->service_id)->first();

    $transferData = json_decode($job->migration_data, true);

    $transfer = new DataTransfer($sourceServer, $targetServer);
    $transferResult = $transfer->execute([
        'service' => $service,
        'backup_id' => $transferData['backup_id'],
    ]);

    return ['success' => true, 'transferred' => $transferResult];
}

private function stepConfigure($job): array {
    $targetServer = Capsule::table('tblservers')->where('id', $job->target_server_id)->first();
    $service = Capsule::table('tblhosting')->where('id', $job->service_id)->first();

    $targetModule = new ServerModule($targetServer->type);

    $result = $targetModule->CreateAccount([
        'serverid' => $job->target_server_id,
        'serviceid' => $service->id,
        'domain' => $service->domain,
        'username' => $service->username,
        'password' => decrypt($service->password),
    ]);

    if ($result !== 'success') {
        return ['success' => false, 'error' => "Configuration failed: $result"];
    }

    return ['success' => true];
}

private function stepVerify($job): array {
    $targetServer = Capsule::table('tblservers')->where('id', $job->target_server_id)->first();
    $service = Capsule::table('tblhosting')->where('id', $job->service_id)->first();

    $targetModule = new ServerModule($targetServer->type);

    $testResult = $targetModule->TestConnection([
        'serverid' => $job->target_server_id,
        'serviceid' => $service->id,
    ]);

    if (!$testResult['success']) {
        return ['success' => false, 'error' => 'Verification failed'];
    }

    return ['success' => true];
}
```

## Job Management

```php
public function createMigration(int $serviceId, int $targetServerId): int {
    $service = Capsule::table('tblhosting')->where('id', $serviceId)->first();

    $jobId = Capsule::table('mod_migration_jobs')->insertGetId([
        'service_id' => $serviceId,
        'source_server_id' => $service->serverid,
        'target_server_id' => $targetServerId,
        'status' => 'pending',
        'created_at' => date('Y-m-d H:i:s'),
    ]);

    return $jobId;
}

public function getMigrationStatus(int $jobId): array {
    $job = Capsule::table('mod_migration_jobs')->where('id', $jobId)->first();

    $steps = Capsule::table('mod_migration_steps')
        ->where('job_id', $jobId)
        ->orderBy('created_at')
        ->get();

    return [
        'job' => $job,
        'steps' => $steps,
        'progress' => $this->calculateProgress($steps),
    ];
}

public function rollback(int $jobId): string {
    $job = Capsule::table('mod_migration_jobs')->where('id', $jobId)->first();

    if ($job->status !== 'failed') {
        return 'Error: Can only rollback failed migrations';
    }

    Capsule::table('mod_migration_jobs')
        ->where('id', $jobId)
        ->update(['status' => 'rolling_back']);

    $this->rollbackSteps($job);

    Capsule::table('mod_migration_jobs')
        ->where('id', $jobId)
        ->update(['status' => 'rolled_back']);

    return 'success';
}
```

---

**Related Skills:**
- whmcs-addon-builder
- whmcs-backup-restore
- whmcs-server-builder