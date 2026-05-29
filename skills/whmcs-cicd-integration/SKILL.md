---
name: whmcs-cicd-integration
description: CI/CD pipeline for WHMCS
category: Automation & DevOps
version: 1.0.0
---

# WHMCS CI/CD Pipeline Integration Skill

## Overview
This skill provides patterns and implementations for integrating CI/CD pipelines with WHMCS, including pipeline configuration, deployment triggers, and artifact management.

## Implementation Patterns

### CI/CD Integration Manager
```php
<?php
/**
 * WHMCS CI/CD Pipeline Integration
 * Manages CI/CD pipeline integration
 */

namespace WHMCS\Module\DevOps\CI;

class CICDIntegrationManager {
    private $db;
    private $artifactManager;

    public function __construct() {
        $this->db = \WHMCS\Database\Capsule::connection();
        $this->artifactManager = new ArtifactManager();
    }

    /**
     * Create CI/CD configuration
     */
    public function createConfiguration(array $params): array {
        $configId = 'cicd_' . bin2hex(random_bytes(12));

        $config = [
            'id' => $configId,
            'service_id' => $params['service_id'],
            'provider' => $params['provider'], // github, gitlab, jenkins
            'repository' => $params['repository'],
            'branch' => $params['branch'] ?? 'main',
            'pipeline_path' => $params['pipeline_path'] ?? '.whmcs/pipeline.yml',
            'deploy_on' => json_encode($params['deploy_on'] ?? ['push', 'tag']),
            'environment' => $params['environment'] ?? 'production',
            'webhook_secret' => bin2hex(random_bytes(32)),
            'created_at' => date('Y-m-d H:i:s')
        ];

        $this->db->insert('mod_cicd_configs', $config);

        // Create webhook
        $this->setupWebhook($config);

        return [
            'success' => true,
            'config_id' => $configId,
            'webhook_url' => $this->getWebhookUrl($configId)
        ];
    }

    /**
     * Handle CI/CD webhook
     */
    public function handleWebhook(string $configId, array $payload): array {
        $config = $this->getConfig($configId);

        if (!$config) {
            throw new \Exception("Config not found");
        }

        // Verify webhook
        if (!$this->verifyWebhook($payload, $config['webhook_secret'])) {
            throw new \Exception("Invalid webhook signature");
        }

        $event = $payload['event'] ?? $payload['action'] ?? 'pipeline.run';

        // Check if this event should trigger deployment
        $deployOn = json_decode($config['deploy_on'], true);
        if (!in_array($event, $deployOn)) {
            return ['triggered' => false, 'reason' => "Event not configured for deployment"];
        }

        // Trigger deployment
        $deploymentId = $this->triggerDeployment($config, $payload);

        return [
            'triggered' => true,
            'deployment_id' => $deploymentId,
            'event' => $event
        ];
    }

    /**
     * Execute pipeline stage
     */
    public function executeStage(string $deploymentId, string $stage): array {
        $deployment = $this->getDeployment($deploymentId);

        if (!$deployment) {
            throw new \Exception("Deployment not found");
        }

        $startTime = microtime(true);

        switch ($stage) {
            case 'build':
                $result = $this->executeBuild($deployment);
                break;
            case 'test':
                $result = $this->executeTests($deployment);
                break;
            case 'deploy':
                $result = $this->executeDeploy($deployment);
                break;
            case 'verify':
                $result = $this->executeVerify($deployment);
                break;
            default:
                throw new \Exception("Unknown stage: {$stage}");
        }

        $duration = microtime(true) - $startTime;

        // Log stage result
        $this->logStage($deploymentId, $stage, $result, $duration);

        return [
            'stage' => $stage,
            'success' => $result['success'],
            'duration' => round($duration, 2),
            'output' => $result['output']
        ];
    }

    /**
     * Get pipeline status
     */
    public function getPipelineStatus(string $deploymentId): array {
        $stages = $this->db->select(
            "SELECT * FROM mod_cicd_stages WHERE deployment_id = ? ORDER BY stage_order ASC",
            [$deploymentId]
        );

        $status = array_map(function($stage) {
            return [
                'name' => $stage->stage_name,
                'status' => $stage->status,
                'duration' => $stage->duration_seconds,
                'started_at' => $stage->started_at,
                'completed_at' => $stage->completed_at
            ];
        }, $stages);

        $overallStatus = 'in_progress';
        if (count($status) > 0) {
            $failed = array_filter($status, fn($s) => $s['status'] === 'failed');
            if (!empty($failed)) $overallStatus = 'failed';
            else {
                $completed = array_filter($status, fn($s) => $s['status'] === 'completed');
                if (count($completed) === count($status)) $overallStatus = 'success';
            }
        }

        return [
            'deployment_id' => $deploymentId,
            'stages' => $status,
            'overall_status' => $overallStatus
        ];
    }

    /**
     * Create deployment record
     */
    private function triggerDeployment(array $config, array $payload): string {
        $deploymentId = 'depl_' . bin2hex(random_bytes(8));

        $this->db->insert('mod_cicd_deployments', [
            'id' => $deploymentId,
            'config_id' => $config['id'],
            'commit_hash' => $payload['commit'] ?? $payload['after'] ?? null,
            'branch' => $payload['branch'] ?? $config['branch'],
            'triggered_by' => $payload['actor'] ?? 'unknown',
            'status' => 'pending',
            'created_at' => date('Y-m-d H:i:s')
        ]);

        // Start pipeline execution
        $this->executePipelineAsync($deploymentId);

        return $deploymentId;
    }

    // Private helper methods

    private function setupWebhook(array $config): void {
        $webhookUrl = $this->getWebhookUrl($config['id']);

        switch ($config['provider']) {
            case 'github':
                $this->setupGitHubWebhook($config['repository'], $webhookUrl, $config['webhook_secret']);
                break;
            case 'gitlab':
                $this->setupGitLabWebhook($config['repository'], $webhookUrl, $config['webhook_secret']);
                break;
        }
    }

    private function executeBuild(array $deployment): array {
        $output = shell_exec("cd /tmp/build-{$deployment['id']} && ./build.sh 2>&1");

        return [
            'success' => strpos($output, 'BUILD SUCCESS') !== false,
            'output' => $output
        ];
    }

    private function executeTests(array $deployment): array {
        $output = shell_exec("cd /tmp/build-{$deployment['id']} && ./run-tests.sh 2>&1");

        return [
            'success' => strpos($output, 'All tests passed') !== false,
            'output' => $output
        ];
    }

    private function executeDeploy(array $deployment): array {
        $output = shell_exec("cd /tmp/build-{$deployment['id']} && ./deploy.sh 2>&1");

        return [
            'success' => strpos($output, 'DEPLOY SUCCESS') !== false,
            'output' => $output
        ];
    }

    private function executeVerify(array $deployment): array {
        // Verify deployment was successful
        $healthCheck = shell_exec("curl -s http://{$deployment['service_id']}.whmcs.test/health");

        return [
            'success' => strpos($healthCheck, 'OK') !== false,
            'output' => $healthCheck
        ];
    }

    private function logStage(string $deploymentId, string $stage, array $result, float $duration): void {
        $status = $result['success'] ? 'completed' : 'failed';

        $this->db->insert('mod_cicd_stages', [
            'deployment_id' => $deploymentId,
            'stage_name' => $stage,
            'status' => $status,
            'duration_seconds' => round($duration, 2),
            'output' => substr($result['output'], 0, 5000),
            'started_at' => date('Y-m-d H:i:s'),
            'completed_at' => date('Y-m-d H:i:s')
        ]);

        // Update deployment status
        if (!$result['success']) {
            $this->db->update('mod_cicd_deployments', [
                'status' => 'failed'
            ], ['id' => $deploymentId]);
        }
    }

    private function getWebhookUrl(string $configId): string {
        return "https://" . $_SERVER['HTTP_HOST'] . "/api/cicd/webhook/{$configId}";
    }
}
```

## Database Schema
```sql
CREATE TABLE `mod_cicd_configs` (
  `id` VARCHAR(50) PRIMARY KEY,
  `service_id` INT NOT NULL,
  `provider` VARCHAR(50) NOT NULL,
  `repository` VARCHAR(500) NOT NULL,
  `branch` VARCHAR(100) DEFAULT 'main',
  `pipeline_path` VARCHAR(255),
  `deploy_on` TEXT,
  `environment` VARCHAR(50) DEFAULT 'production',
  `webhook_secret` VARCHAR(100),
  `created_at' DATETIME NOT NULL
);

CREATE TABLE `mod_cicd_deployments` (
  `id` VARCHAR(50) PRIMARY KEY,
  `config_id` VARCHAR(50) NOT NULL,
  `commit_hash` VARCHAR(40),
  `branch` VARCHAR(100),
  `triggered_by` VARCHAR(255),
  `status` ENUM('pending', 'running', 'success', 'failed', 'cancelled') DEFAULT 'pending',
  `created_at' DATETIME NOT NULL,
  FOREIGN KEY (`config_id`) REFERENCES `mod_cicd_configs`(`id`)
);

CREATE TABLE `mod_cicd_stages` (
  `id` INT AUTO_INCREMENT PRIMARY KEY,
  `deployment_id` VARCHAR(50) NOT NULL,
  `stage_name` VARCHAR(100) NOT NULL,
  `stage_order` INT,
  `status' ENUM('pending', 'running', 'completed', 'failed', 'skipped') DEFAULT 'pending',
  `duration_seconds` INT,
  `output' TEXT,
  `started_at' DATETIME,
  `completed_at' DATETIME,
  FOREIGN KEY (`deployment_id`) REFERENCES `mod_cicd_deployments`(`id`)
);
```

## Best Practices

1. **Pipeline Stages**: Implement clear stages (build, test, deploy, verify)
2. **Quality Gates**: Enforce testing before deployment
3. **Artifact Management**: Use artifact repositories
4. **Notifications**: Send notifications for pipeline status
5. **Rollback Plan**: Have rollback capability ready

## Related Skills

- whmcs-github-actions
- whmcs-gitlab-ci
- whmcs-jenkins-pipeline
- whmcs-gitops-workflow