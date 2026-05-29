---
name: whmcs-gitops-workflow
description: GitOps implementation for WHMCS
category: Automation & DevOps
version: 1.0.0
---

# WHMCS GitOps Workflow Skill

## Overview
This skill provides patterns and implementations for implementing GitOps workflows in WHMCS, including infrastructure as code, automated deployments, and configuration management.

## Implementation Patterns

### GitOps Workflow Manager
```php
<?php
/**
 * WHMCS GitOps Workflow Implementation
 * Manages GitOps-based deployments
 */

namespace WHMCS\Module\DevOps\GitOps;

class GitOpsWorkflowManager {
    private $db;
    private $gitClient;
    private $deployer;

    public function __construct() {
        $this->db = \WHMCS\Database\Capsule::connection();
        $this->gitClient = new GitClient();
        $this->deployer = new GitOpsDeployer();
    }

    /**
     * Setup GitOps for service
     */
    public function setupGitOps(array $params): array {
        $configId = 'go_' . bin2hex(random_bytes(12));

        $config = [
            'id' => $configId,
            'service_id' => $params['service_id'],
            'repository_url' => $params['repository_url'],
            'branch' => $params['branch'] ?? 'main',
            'deploy_path' => $params['deploy_path'],
            'ci_config_path' => $params['ci_config_path'] ?? '.whmcs/deploy.yaml',
            'auto_deploy' => $params['auto_deploy'] ?? true,
            'rollback_on_failure' => $params['rollback'] ?? true,
            'webhook_secret' => bin2hex(random_bytes(32)),
            'created_at' => date('Y-m-d H:i:s')
        ];

        $this->db->insert('mod_gitops_configs', $config);

        // Setup webhook
        $this->setupWebhook($config);

        // Initialize repository tracking
        $this->initializeTracking($configId);

        return [
            'success' => true,
            'config_id' => $configId,
            'webhook_url' => $this->getWebhookUrl($configId),
            'webhook_secret' => $config['webhook_secret']
        ];
    }

    /**
     * Handle webhook from Git provider
     */
    public function handleWebhook(string $configId, array $payload): array {
        $config = $this->getConfig($configId);

        if (!$config) {
            throw new \Exception("GitOps config not found");
        }

        // Verify webhook signature
        if (!$this->verifyWebhookSignature($payload, $config['webhook_secret'])) {
            throw new \Exception("Invalid webhook signature");
        }

        $event = $payload['event'] ?? $payload['action'] ?? 'push';

        switch ($event) {
            case 'push':
                return $this->handlePushEvent($config, $payload);
            case 'pull_request':
            case 'merge_request':
                return $this->handleMergeRequest($config, $payload);
            case 'tag':
                return $this->handleTagEvent($config, $payload);
            default:
                return ['handled' => false, 'reason' => "Unknown event: {$event}"];
        }
    }

    /**
     * Handle push event (deployment)
     */
    private function handlePushEvent(array $config, array $payload): array {
        $branch = $payload['ref'];
        $commit = $payload['after'] ?? $payload['head_commit']['id'] ?? null;

        // Check if we should deploy this branch
        if (!$this->shouldDeploy($config, $branch)) {
            return ['deployed' => false, 'reason' => 'Branch not configured for deployment'];
        }

        // Get changes
        $changes = $this->getChangedFiles($config, $payload);

        // Determine deployment type
        $deploymentType = $this->determineDeploymentType($changes);

        // Execute deployment
        $result = $this->deployer->deploy($config, [
            'branch' => $branch,
            'commit' => $commit,
            'type' => $deploymentType,
            'changes' => $changes
        ]);

        // Log deployment
        $this->logDeployment($config['id'], $result);

        return [
            'deployed' => true,
            'deployment_id' => $result['deployment_id'],
            'type' => $deploymentType,
            'commit' => $commit
        ];
    }

    /**
     * Deploy specific commit
     */
    public function deployCommit(string $configId, string $commit): array {
        $config = $this->getConfig($configId);

        if (!$config) {
            throw new \Exception("Config not found");
        }

        // Checkout and deploy specific commit
        $result = $this->deployer->deployCommit($config, $commit);

        $this->logDeployment($configId, $result);

        return [
            'success' => true,
            'deployment_id' => $result['deployment_id'],
            'commit' => $commit
        ];
    }

    /**
     * Rollback to previous version
     */
    public function rollback(string $configId): array {
        $config = $this->getConfig($configId);

        if (!$config) {
            throw new \Exception("Config not found");
        }

        // Get previous successful deployment
        $previous = $this->getPreviousDeployment($config['id']);

        if (!$previous) {
            throw new \Exception("No previous deployment to rollback to");
        }

        // Execute rollback
        $result = $this->deployer->rollback($config, $previous);

        $this->logDeployment($configId, $result, 'rollback');

        return [
            'success' => true,
            'rollback_to' => $previous['commit'],
            'deployment_id' => $result['deployment_id']
        ];
    }

    /**
     * Get deployment history
     */
    public function getDeploymentHistory(string $configId, int $limit = 50): array {
        $deployments = $this->db->select(
            "SELECT * FROM mod_gitops_deployments
             WHERE config_id = ?
             ORDER BY created_at DESC
             LIMIT ?",
            [$configId, $limit]
        );

        return array_map(function($dep) {
            return [
                'id' => $dep->id,
                'commit' => $dep->commit,
                'branch' => $dep->branch,
                'type' => $dep->type,
                'status' => $dep->status,
                'duration_seconds' => $dep->duration_seconds,
                'deployed_by' => $dep->deployed_by,
                'created_at' => $dep->created_at
            ];
        }, $deployments);
    }

    // Private helper methods

    private function setupWebhook(array $config): void {
        $webhookUrl = $this->getWebhookUrl($config['id']);

        $this->gitClient->createWebhook($config['repository_url'], [
            'url' => $webhookUrl,
            'secret' => $config['webhook_secret'],
            'events' => ['push', 'tag', 'pull_request']
        ]);
    }

    private function initializeTracking(string $configId): void {
        // Track initial state
        $this->db->insert('mod_gitops_tracking', [
            'config_id' => $configId,
            'tracked_at' => date('Y-m-d H:i:s')
        ]);
    }

    private function shouldDeploy(array $config, string $branch): bool {
        // Check if branch matches configured branch
        return $branch === "refs/heads/{$config['branch']}";
    }

    private function getChangedFiles(array $config, array $payload): array {
        $changed = $payload['changes'] ?? [];

        return array_map(function($change) {
            return $change['file'];
        }, $changed);
    }

    private function determineDeploymentType(array $changes): string {
        $types = [
            'full' => ['config', 'module', 'core'],
            'partial' => ['template', 'language', 'asset'],
            'config_only' => ['.whmcs.yaml']
        ];

        foreach ($types as $type => $patterns) {
            foreach ($patterns as $pattern) {
                foreach ($changes as $change) {
                    if (strpos($change, $pattern) !== false) {
                        return $type;
                    }
                }
            }
        }

        return 'standard';
    }

    private function getWebhookUrl(string $configId): string {
        return "https://" . $_SERVER['HTTP_HOST'] . "/api/gitops/webhook/{$configId}";
    }

    private function verifyWebhookSignature(array $payload, string $secret): bool {
        $signature = $payload['signature'] ?? '';
        $computed = hash_hmac('sha256', json_encode($payload), $secret);

        return hash_equals($signature, $computed);
    }
}

/**
 * GitOps Deployer
 */
class GitOpsDeployer {
    public function deploy(array $config, array $params): array {
        $deploymentId = 'dep_' . bin2hex(random_bytes(12));
        $startTime = microtime(true);

        try {
            // Clone/fetch repository
            $this->checkoutRepository($config['repository_url'], $params['branch']);

            // Apply configuration
            $this->applyConfiguration($config['deploy_path']);

            // Run post-deploy scripts
            $this->runPostDeployScripts($config['deploy_path']);

            // Verify deployment
            $this->verifyDeployment($config['deploy_path']);

            $duration = microtime(true) - $startTime;

            return [
                'deployment_id' => $deploymentId,
                'status' => 'success',
                'duration_seconds' => round($duration, 2)
            ];
        } catch (\Exception $e) {
            return [
                'deployment_id' => $deploymentId,
                'status' => 'failed',
                'error' => $e->getMessage()
            ];
        }
    }

    private function checkoutRepository(string $repoUrl, string $branch): void {
        $tempPath = "/tmp/gitops-" . uniqid();
        exec("git clone -b {$branch} {$repoUrl} {$tempPath}");
    }

    private function applyConfiguration(string $path): void {
        // Apply configurations from .whmcs/deploy.yaml
    }

    private function runPostDeployScripts(string $path): void {
        // Run any post-deploy hooks
    }

    private function verifyDeployment(string $path): void {
        // Verify deployment was successful
    }
}
```

## Database Schema
```sql
CREATE TABLE `mod_gitops_configs` (
  `id` VARCHAR(50) PRIMARY KEY,
  `service_id` INT NOT NULL,
  `repository_url` VARCHAR(500) NOT NULL,
  `branch` VARCHAR(100) DEFAULT 'main',
  `deploy_path` VARCHAR(500) NOT NULL,
  `ci_config_path` VARCHAR(255),
  `auto_deploy` TINYINT(1) DEFAULT 1,
  `rollback_on_failure` TINYINT(1) DEFAULT 1,
  `webhook_secret` VARCHAR(100) NOT NULL,
  `status` ENUM('active', 'paused', 'error') DEFAULT 'active',
  `created_at' DATETIME NOT NULL
);

CREATE TABLE `mod_gitops_deployments` (
  `id` VARCHAR(50) PRIMARY KEY,
  `config_id` VARCHAR(50) NOT NULL,
  `commit` VARCHAR(40) NOT NULL,
  `branch` VARCHAR(100) NOT NULL,
  `type` ENUM('full', 'partial', 'config_only', 'rollback') NOT NULL,
  `status` ENUM('pending', 'in_progress', 'success', 'failed') DEFAULT 'pending',
  `duration_seconds` INT,
  `deployed_by` VARCHAR(100),
  `error_message` TEXT,
  `created_at' DATETIME NOT NULL,
  FOREIGN KEY (`config_id`) REFERENCES `mod_gitops_configs`(`id`)
);
```

## GitOps Workflow

```
┌─────────────────────────────────────────────────────────────────┐
│                    GitOps Deployment Flow                        │
├─────────────────────────────────────────────────────────────────┤
│  1. Developer pushes code to Git repository                      │
│     └─> Git provider sends webhook to WHMCS                       │
│                                                                  │
│  2. WHMCS validates webhook signature                            │
│     └─> Parse webhook payload                                    │
│                                                                  │
│  3. Determine deployment type based on changed files             │
│     └─> Full deploy, partial deploy, or config only              │
│                                                                  │
│  4. Execute deployment                                           │
│     ├─> Clone/fetch repository                                   │
│     ├─> Apply configurations                                      │
│     ├─> Run post-deploy scripts                                  │
│     └─> Verify deployment                                        │
│                                                                  │
│  5. Log deployment result                                         │
│     └─> Send notification if configured                         │
└─────────────────────────────────────────────────────────────────┘
```

## Best Practices

1. **Branch Strategy**: Use separate branches for dev/staging/production
2. **Atomic Deployments**: Ensure deployments are all-or-nothing
3. **Rollback Plan**: Always have a way to revert quickly
4. **Secret Management**: Never store secrets in Git
5. **Testing**: Use CI/CD to test before deployment

## Related Skills

- whmcs-infrastructure-code
- whmcs-cicd-integration
- whmcs-github-actions
- whmcs-gitlab-ci