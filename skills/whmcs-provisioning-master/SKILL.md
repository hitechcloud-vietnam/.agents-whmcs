# WHMCS Provisioning Master

## Overview
Master skill for automated service provisioning in WHMCS. Covers server allocation, module development, provisioning automation, and error handling.

## Server Module Structure

```php
<?php
// /modules/servers/YourModule/YourModule.php
namespace WHMCS\Module\Server\YourModule;

use WHMCS\Module\Server\YourModule\Actions\CreateAccount;
use WHMCS\Module\Server\YourModule\Actions\TerminateAccount;
use WHMCS\Module\Server\YourModule\Actions\SuspendAccount;
use WHMCS\Module\Server\YourModule\Actions\UnsuspendAccount;
use WHMCS\Module\Server\YourModule\Actions\ChangePackage;
use WHMCS\Module\Server\YourModule\Actions\ChangePassword;
use WHMCS\Module\Server\YourModule\Actions\LoginLink;
use WHMCS\Module\Server\YourModule\Actions\AdminLoginLink;
use WHMCS\Module\Server\YourModule\Actions\UsageUpdate;

class YourModule
{
    protected $api;
    protected $config;

    public function __construct()
    {
        $this->config = $this->loadConfig();
        $this->api = new ApiClient($this->config);
    }

    public static function factory($params)
    {
        $instance = new self();
        $instance->params = $params;
        return $instance;
    }

    // Provision new service
    public function createAccount(array $params): array
    {
        try {
            $username = $this->generateUsername($params);
            $password = $this->generatePassword();

            $result = $this->api->createAccount([
                'username' => $username,
                'password' => $password,
                'email' => $params['clientsdetails']['email'],
                'package_id' => $params['configoption1'],
                'domain' => $params['domain'],
            ]);

            if ($result['success']) {
                return [
                    'success' => true,
                    'accountid' => $result['server_account_id'],
                    'username' => $username,
                    'password' => $password,
                ];
            }

            return [
                'success' => false,
                'error' => $result['error'] ?? 'Provisioning failed',
            ];
        } catch (\Exception $e) {
            return [
                'success' => false,
                'error' => 'Provisioning error: ' . $e->getMessage(),
            ];
        }
    }

    // Terminate service
    public function terminateAccount(array $params): array
    {
        try {
            $result = $this->api->deleteAccount($params['username']);

            return [
                'success' => $result['success'],
                'error' => $result['error'] ?? null,
            ];
        } catch (\Exception $e) {
            return [
                'success' => false,
                'error' => $e->getMessage(),
            ];
        }
    }

    // Suspend service
    public function suspendAccount(array $params): array
    {
        try {
            $result = $this->api->suspendAccount($params['username']);

            return [
                'success' => $result['success'],
                'error' => $result['error'] ?? null,
            ];
        } catch (\Exception $e) {
            return [
                'success' => false,
                'error' => $e->getMessage(),
            ];
        }
    }

    // Unsuspend service
    public function unsuspendAccount(array $params): array
    {
        try {
            $result = $this->api->unsuspendAccount($params['username']);

            return [
                'success' => $result['success'],
                'error' => $result['error'] ?? null,
            ];
        } catch (\Exception $e) {
            return [
                'success' => false,
                'error' => $e->getMessage(),
            ];
        }
    }

    // Change hosting package
    public function changePackage(array $params): array
    {
        try {
            $result = $this->api->updateAccount([
                'username' => $params['username'],
                'package_id' => $params['configoption1'],
            ]);

            return [
                'success' => $result['success'],
                'error' => $result['error'] ?? null,
            ];
        } catch (\Exception $e) {
            return [
                'success' => false,
                'error' => $e->getMessage(),
            ];
        }
    }

    // Change password
    public function changePassword(array $params): array
    {
        try {
            $result = $this->api->updatePassword([
                'username' => $params['username'],
                'password' => $params['password'],
            ]);

            return [
                'success' => $result['success'],
                'error' => $result['error'] ?? null,
            ];
        } catch (\Exception $e) {
            return [
                'success' => false,
                'error' => $e->getMessage(),
            ];
        }
    }

    // Client login link
    public function loginLink(array $params): string
    {
        $link = $this->api->getLoginLink($params['username']);
        return '<a href="' . $link . '" target="_blank">Login to Control Panel</a>';
    }

    // Admin login link
    public function adminLoginLink(array $params): string
    {
        $link = $this->api->getAdminLoginLink($params['username']);
        return '<a href="' . $link . '" target="_blank">Admin Login</a>';
    }

    // Usage/Bandwidth update (for metered billing)
    public function usageUpdate(array $params): array
    {
        try {
            $usage = $this->api->getUsage($params['username']);

            return [
                'success' => true,
                'results' => [
                    'diskusage' => $usage['disk'],
                    'disklimit' => $usage['disk_limit'],
                    'bwusage' => $usage['bandwidth'],
                    'bwlimit' => $usage['bandwidth_limit'],
                ],
            ];
        } catch (\Exception $e) {
            return [
                'success' => false,
                'error' => $e->getMessage(),
            ];
        }
    }

    // Test connection
    public function testConnection(array $params): array
    {
        try {
            $result = $this->api->ping();

            if ($result['success']) {
                return [
                    'success' => true,
                    'error' => '',
                ];
            }

            return [
                'success' => false,
                'error' => 'Connection failed',
            ];
        } catch (\Exception $e) {
            return [
                'success' => false,
                'error' => $e->getMessage(),
            ];
        }
    }

    private function generateUsername(array $params): string
    {
        $prefix = $params['configoption2'] ?? 'user';
        $domain = preg_replace('/[^a-z0-9]/', '', strtolower($params['domain']));
        return substr($prefix . $domain, 0, 20);
    }

    private function generatePassword(): string
    {
        return bin2hex(random_bytes(16));
    }

    private function loadConfig(): array
    {
        return [];
    }
}
```

## Provisioning Hooks

```php
<?php
// /includes/hooks/provisioning_hooks.php

use WHMCS\Service\Status;

// Pre-provisioning hook
add_hook('PreServiceCreate', 1, function(array $params) {
    // Validate before provisioning
    $errors = [];

    // Check domain availability
    if (strpos($params['domain'], 'banned-domain') !== false) {
        $errors[] = 'This domain is not allowed';
    }

    // Check server availability
    $server = \WHMCS\Service\Server::find($params['server']);
    if (!$server || !$server->isActive()) {
        $errors[] = 'Selected server is not available';
    }

    if (!empty($errors)) {
        return ['error' => implode(', ', $errors)];
    }

    return [];
});

// Post-provisioning hook
add_hook('AfterServiceCreate', 1, function(array $params) {
    $service = $params['service'];

    // Send welcome email
    send_email('Welcome', $service->client_id, [
        'service_id' => $service->id,
        'username' => $params['username'],
    ]);

    // Create DNS records
    create_dns_record($service->domain, $params['server_ip']);

    // Setup monitoring
    setup_monitoring($service->id);

    // Log to audit
    logActivity("Service provisioned: {$service->id}");
});

// Service created successfully
add_hook('ServiceCreated', 1, function(array $params) {
    $serviceId = $params['serviceid'];
    $service = \WHMCS\Service::find($serviceId);

    // Custom logic after successful provisioning
    run_automation('post_provision', $service);
});

// Provisioning failed
add_hook('ProvisioningFailed', 1, function(array $params) {
    $serviceId = $params['serviceid'];
    $error = $params['error'];

    // Alert administrators
    notify_admins("Provisioning failed for service {$serviceId}: {$error}");

    // Log for retry
    log_provisioning_failure($serviceId, $error);

    // Auto-retry logic
    if ($params['retry_count'] < 3) {
        queue_provisioning_retry($serviceId);
    }
});

// Pre-termination hook
add_hook('PreServiceTermination', 1, function(array $params) {
    $service = \WHMCS\Service::find($params['serviceid']);

    // Check for outstanding balances
    if ($service->client->getBalance() > 0) {
        return ['error' => 'Cannot terminate service with outstanding balance'];
    }

    // Check for data retention requirements
    if ($service->hasLegalHold()) {
        return ['error' => 'Service has legal hold - cannot terminate'];
    }

    return [];
});

// Post-termination hook
add_hook('AfterServiceTermination', 1, function(array $params) {
    $serviceId = $params['serviceid'];

    // Backup data before deletion
    backup_service_data($serviceId);

    // Remove DNS records
    remove_dns_records($params['domain']);

    // Release monitoring
    release_monitoring($serviceId);

    // Notify client
    send_email('ServiceTerminated', $params['userid'], [
        'service_id' => $serviceId,
        'domain' => $params['domain'],
    ]);
});

// Pre-suspend hook
add_hook('PreServiceSuspend', 1, function(array $params) {
    // Custom suspend validation
    $service = \WHMCS\Service::find($params['serviceid']);

    if ($service->isExemptFromSuspension()) {
        return ['error' => 'This service is exempt from suspension'];
    }

    return [];
});

// Post-unsuspend hook
add_hook('AfterServiceUnsuspend', 1, function(array $params) {
    $serviceId = $params['serviceid'];

    // Send reactivation notification
    send_email('ServiceReactivated', $params['userid'], [
        'service_id' => $serviceId,
    ]);
});
```

## Automated Provisioning Queue

```php
<?php
// /includes/automation/provisioning_queue.php

class ProvisioningQueue
{
    private $db;
    private $maxRetries = 3;
    private $retryDelay = 300; // 5 minutes

    public function __construct()
    {
        $this->db = \App::getDb();
    }

    // Add service to provisioning queue
    public function enqueue(int $serviceId, array $params): void
    {
        $stmt = $this->db->prepare("
            INSERT INTO mod_provisioning_queue
            (service_id, params, status, created_at, attempts)
            VALUES (?, ?, 'pending', NOW(), 0)
        ");

        $stmt->execute([$serviceId, json_encode($params)]);
    }

    // Process queue
    public function process(): void
    {
        $pending = $this->getPendingItems();

        foreach ($pending as $item) {
            $this->processItem($item);
        }
    }

    private function processItem(array $item): void
    {
        $this->updateStatus($item['id'], 'processing');

        try {
            $service = \WHMCS\Service::find($item['service_id']);
            $params = json_decode($item['params'], true);
            $params['server'] = $this->selectServer($service);

            $module = $service->getServerModule();
            $result = $module->createAccount($params);

            if ($result['success']) {
                $this->updateStatus($item['id'], 'completed');
                $this->updateService($service, $result);
                $this->runPostProvisioningHooks($service, $result);
            } else {
                $this->handleFailure($item, $result['error']);
            }
        } catch (\Exception $e) {
            $this->handleFailure($item, $e->getMessage());
        }
    }

    private function selectServer(\WHMCS\Service $service): \WHMCS\Server
    {
        // Server selection logic based on:
        // - Location
        // - Load
        // - Available resources
        // - User preferences

        $servers = \WHMCS\Server::active()
            ->where('type', $service->product->module)
            ->get();

        foreach ($servers as $server) {
            if ($this->serverHasCapacity($server, $service)) {
                return $server;
            }
        }

        throw new \Exception('No available servers');
    }

    private function serverHasCapacity(\WHMCS\Server $server, \WHMCS\Service $service): bool
    {
        // Check server load
        $load = $server->getAverageLoad();

        // Check available resources
        $hasDisk = $server->hasAvailableDisk($service->product->disk_limit);
        $hasBandwidth = $server->hasAvailableBandwidth($service->product->bandwidth_limit);

        return $load < 0.8 && $hasDisk && $hasBandwidth;
    }

    private function handleFailure(array $item, string $error): void
    {
        $attempts = $item['attempts'] + 1;

        if ($attempts >= $this->maxRetries) {
            $this->updateStatus($item['id'], 'failed');
            $this->notifyFailure($item['service_id'], $error);
        } else {
            $this->updateStatus($item['id'], 'retry', $attempts);
            $this->scheduleRetry($item['id'], $this->retryDelay * $attempts);
        }
    }

    private function scheduleRetry(int $itemId, int $delay): void
    {
        $this->db->prepare("
            UPDATE mod_provisioning_queue
            SET next_attempt_at = DATE_ADD(NOW(), INTERVAL ? SECOND)
            WHERE id = ?
        ")->execute([$delay, $itemId]);
    }

    private function getPendingItems(): array
    {
        return $this->db->query("
            SELECT * FROM mod_provisioning_queue
            WHERE status IN ('pending', 'retry')
            AND (next_attempt_at IS NULL OR next_attempt_at <= NOW())
            ORDER BY created_at ASC
            LIMIT 10
        ")->fetchAll();
    }

    private function updateStatus(int $id, string $status, int $attempts = null): void
    {
        if ($attempts !== null) {
            $this->db->prepare("
                UPDATE mod_provisioning_queue
                SET status = ?, attempts = ?
                WHERE id = ?
            ")->execute([$status, $attempts, $id]);
        } else {
            $this->db->prepare("
                UPDATE mod_provisioning_queue
                SET status = ?, completed_at = NOW()
                WHERE id = ?
            ")->execute([$status, $id]);
        }
    }
}
```

## Best Practices

1. **Idempotency**: Provisioning should be idempotent - running it multiple times should produce the same result
2. **Error Handling**: Always wrap API calls in try-catch and return structured error responses
3. **Logging**: Log all provisioning attempts and outcomes for debugging
4. **Timeouts**: Set appropriate timeouts for API calls (recommend 60 seconds)
5. **Rollback**: Implement rollback procedures for failed provisioning
6. **Validation**: Validate all inputs before making API calls
7. **Security**: Never store passwords in plain text; use secure password generation
8. **Queue Processing**: Use background queues for long-running provisioning tasks
9. **Monitoring**: Track provisioning success/failure rates
10. **Testing**: Test all provisioning scenarios including failure cases

## API Response Format

```json
{
    "success": true|false,
    "accountid": "server-account-id",
    "username": "service-username",
    "password": "encrypted-password",
    "error": "error-message-if-failed",
    "metadata": {
        "server_id": 123,
        "provisioned_at": "2024-01-01T00:00:00Z"
    }
}
```
