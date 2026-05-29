# WHMCS Service Provisioning Workflow

## Overview
This workflow automates the provisioning of services in WHMCS, from order completion to service activation.

## Provisioning Flow

```
Order Placed → Payment Verified → Order Approved → Service Provisioned → Service Activated → Welcome Email
```

## Step 1: Configure Product Provisioning

```php
<?php
// includes/hooks/provisioning_hook.php

use WHMCS\Database\Capsule;

// Hook: After order payment
add_hook('OrderPaid', 1, function($params) {
    $orderId = $params['order_id'];

    // Log provisioning request
    logActivity("Order paid - provisioning initiated", $params['user_id']);

    // Trigger provisioning
    run_hook('ProvisioningService', ['order_id' => $orderId]);

    return $params;
});

// Hook: Order status changed to Active
add_hook('OrderStatusChange', 1, function($params) {
    if ($params['status'] === 'Active') {
        $orderId = $params['order_id'];

        // Process provisioning for order
        $provisioner = new \WHMCS\Module\Addon\YourModule\Service\ProvisionService();
        $result = $provisioner->processOrder($orderId);

        if (!$result['success']) {
            logActivity("Provisioning failed for order $orderId: " . $result['error'], $params['user_id']);
        }
    }

    return $params;
});
```

## Step 2: Provisioning Service

```php
<?php
// src/Service/ProvisionService.php

namespace WHMCS\Module\Addon\YourModule\Service;

use WHMCS\Database\Capsule;
use WHMCS\Module\Server;
use WHMCS\Service\Status;

class ProvisionService
{
    private $serverModule;
    private $logger;

    public function __construct()
    {
        $this->logger = new ProvisioningLogger();
    }

    public function processOrder(int $orderId): array
    {
        $this->logger->log("Processing order: $orderId");

        try {
            // Get order details
            $order = $this->getOrder($orderId);
            if (!$order) {
                throw new \Exception("Order not found: $orderId");
            }

            // Get order items
            $items = $this->getOrderItems($orderId);
            if (empty($items)) {
                throw new \Exception("No items in order: $orderId");
            }

            $results = [];
            foreach ($items as $item) {
                $result = $this->provisionItem($order, $item);
                $results[] = $result;
            }

            // Check if all succeeded
            $allSuccess = !in_array(false, array_column($results, 'success'));

            if ($allSuccess) {
                $this->updateOrderStatus($orderId, 'Active');
                $this->sendProvisioningNotification($order, $results);
            }

            return [
                'success' => $allSuccess,
                'order_id' => $orderId,
                'results' => $results
            ];

        } catch (\Exception $e) {
            $this->logger->error("Provisioning failed: " . $e->getMessage());
            return [
                'success' => false,
                'order_id' => $orderId,
                'error' => $e->getMessage()
            ];
        }
    }

    private function provisionItem(array $order, array $item): array
    {
        $this->logger->log("Provisioning item: " . json_encode($item));

        try {
            switch ($item['type']) {
                case 'hostingaccount':
                    return $this->provisionHosting($order, $item);
                case 'server':
                    return $this->provisionServer($order, $item);
                case 'domain':
                    return $this->provisionDomain($order, $item);
                case 'addon':
                    return $this->provisionAddon($order, $item);
                default:
                    return ['success' => true, 'message' => 'No provisioning needed'];
            }
        } catch (\Exception $e) {
            $this->logger->error("Item provisioning failed: " . $e->getMessage());
            return [
                'success' => false,
                'item_id' => $item['id'],
                'error' => $e->getMessage()
            ];
        }
    }

    private function provisionHosting(array $order, array $item): array
    {
        // Get product details
        $product = Capsule::table('tblproducts')
            ->where('id', $item['productid'])
            ->first();

        // Determine server
        $serverId = $this->selectServer($product);

        // Generate credentials
        $username = $this->generateUsername($order['userid']);
        $password = $this->generatePassword();

        // Create hosting account
        $hostingId = Capsule::table('tblhosting')->insertGetId([
            'userid' => $order['userid'],
            'orderid' => $order['id'],
            'packageid' => $item['productid'],
            'serverid' => $serverId,
            'regdate' => date('Y-m-d H:i:s'),
            'domainstatus' => 'Active',
            'billingcycle' => $item['billingcycle'],
            'nextduedate' => date('Y-m-d'),
            'paymentmethod' => $order['paymentmethod'],
            'username' => $username,
            'password' => encrypt($password),
            'domain' => $item['domain'] ?? ''
        ]);

        // Update order item with hosting ID
        Capsule::table('tblorderitems')
            ->where('id', $item['id'])
            ->update(['relid' => $hostingId]);

        // Provision on server
        $serverResult = $this->provisionOnServer($serverId, $hostingId, [
            'username' => $username,
            'password' => $password,
            'domain' => $item['domain'] ?? '',
            'product' => $product
        ]);

        if (!$serverResult['success']) {
            // Rollback hosting account
            Capsule::table('tblhosting')
                ->where('id', $hostingId)
                ->update(['domainstatus' => 'Failed']);
        }

        return [
            'success' => $serverResult['success'],
            'hosting_id' => $hostingId,
            'username' => $username,
            'server_result' => $serverResult
        ];
    }

    private function provisionOnServer(int $serverId, int $hostingId, array $params): array
    {
        $server = Capsule::table('tblservers')
            ->where('id', $serverId)
            ->first();

        if (!$server) {
            return ['success' => false, 'error' => 'Server not found'];
        }

        // Load server module
        $moduleName = $server->type;
        if (!WHMCS\Module\Server::load($moduleName)) {
            return ['success' => false, 'error' => 'Server module not found'];
        }

        $module = new Server($moduleName);

        // Set server credentials
        $module->setServerCredential('hostname', $server->hostname);
        $module->setServerCredential('username', $server->username);
        $module->setServerCredential('password', decrypt($server->password));
        $module->setServerCredential('accesshash', $server->accesshash);
        $module->setServerCredential('secure', $server->secure);

        // Call provisioning function
        try {
            $result = $module->call('CreateAccount', [
                'serviceid' => $hostingId,
                'username' => $params['username'],
                'password' => $params['password'],
                'domain' => $params['domain'],
                'product' => $params['product']
            ]);

            return ['success' => true, 'result' => $result];
        } catch (\Exception $e) {
            return ['success' => false, 'error' => $e->getMessage()];
        }
    }

    private function selectServer($product): int
    {
        // If product has server group, select from group
        if (!empty($product->servergroup)) {
            $server = Capsule::table('tblservers')
                ->join('tblservergroupsrel', 'tblservers.id', '=', 'tblservergroupsrel.serverid')
                ->where('tblservergroupsrel.groupid', $product->servergroup)
                ->where('tblservers.active', 1)
                ->orderBy('tblservers.weight')
                ->first();

            return $server ? $server->serverid : 0;
        }

        // Otherwise, find first active server for product type
        $server = Capsule::table('tblservers')
            ->where('type', $product->type)
            ->where('active', 1)
            ->orderBy('weight')
            ->first();

        return $server ? $server->serverid : 0;
    }

    private function generateUsername(int $clientId): string
    {
        $prefix = Capsule::config('ProvisioningUsernamePrefix') ?: 'user';
        $random = substr(md5($clientId . time()), 0, 8);
        return strtolower($prefix . $random);
    }

    private function generatePassword(): string
    {
        $chars = 'abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789!@#$%';
        $password = '';
        for ($i = 0; $i < 16; $i++) {
            $password .= $chars[random_int(0, strlen($chars) - 1)];
        }
        return $password;
    }

    private function getOrder(int $orderId): ?array
    {
        $order = Capsule::table('tblorders')
            ->where('id', $orderId)
            ->first();

        return $order ? (array)$order : null;
    }

    private function getOrderItems(int $orderId): array
    {
        return Capsule::table('tblorderitems')
            ->where('orderid', $orderId)
            ->get()
            ->toArray();
    }

    private function updateOrderStatus(int $orderId, string $status): void
    {
        Capsule::table('tblorders')
            ->where('id', $orderId)
            ->update(['status' => $status]);
    }

    private function sendProvisioningNotification(array $order, array $results): void
    {
        $client = Capsule::table('tblclients')
            ->where('id', $order['userid'])
            ->first();

        send_email('Service Provisioned', $client->email, [
            'order_id' => $order['id'],
            'client_name' => $client->firstname . ' ' . $client->lastname,
            'items' => $results
        ]);
    }
}
```

## Step 3: Provisioning Logger

```php
<?php
// src/Service/ProvisioningLogger.php

namespace WHMCS\Module\Addon\YourModule\Service;

class ProvisioningLogger
{
    private $logPath;
    private $enabled;

    public function __construct()
    {
        $this->logPath = dirname(__DIR__, 3) . '/storage/logs/provisioning.log';
        $this->enabled = Capsule::config('ProvisioningDebugMode') ?? false;
    }

    public function log(string $message, array $context = []): void
    {
        if (!$this->enabled) {
            return;
        }

        $timestamp = date('Y-m-d H:i:s');
        $contextStr = !empty($context) ? ' | ' . json_encode($context) : '';
        $logLine = "[{$timestamp}] INFO: {$message}{$contextStr}\n";

        $this->write($logLine);
    }

    public function error(string $message, array $context = []): void
    {
        $timestamp = date('Y-m-d H:i:s');
        $contextStr = !empty($context) ? ' | ' . json_encode($context) : '';
        $logLine = "[{$timestamp}] ERROR: {$message}{$contextStr}\n";

        $this->write($logLine);
    }

    public function success(string $message, array $context = []): void
    {
        $timestamp = date('Y-m-d H:i:s');
        $contextStr = !empty($context) ? ' | ' . json_encode($context) : '';
        $logLine = "[{$timestamp}] SUCCESS: {$message}{$contextStr}\n";

        $this->write($logLine);
    }

    private function write(string $line): void
    {
        $dir = dirname($this->logPath);
        if (!is_dir($dir)) {
            mkdir($dir, 0755, true);
        }

        file_put_contents($this->logPath, $line, FILE_APPEND);
    }

    public function getRecentLogs(int $limit = 100): array
    {
        if (!file_exists($this->logPath)) {
            return [];
        }

        $lines = file($this->logPath);
        $lines = array_slice($lines, -$limit);

        return array_map('trim', $lines);
    }
}
```

## Step 4: Automated Provisioning Cron

```php
<?php
// includes/cron/provisioning_cron.php

require_once __DIR__ . '/../../init.php';

use WHMCS\Database\Capsule;
use WHMCS\Module\Addon\YourModule\Service\ProvisionService;

echo "=== Provisioning Cron ===\n";
echo "Started: " . date('Y-m-d H:i:s') . "\n\n";

// Process pending orders
echo "Processing pending orders...\n";
processPendingOrders();

// Retry failed provisioning
echo "Retrying failed provisioning...\n";
retryFailedProvisioning();

// Process pending server actions
echo "Processing server actions...\n";
processServerActions();

echo "\n=== Provisioning Cron Complete ===\n";

function processPendingOrders(): void
{
    // Get orders that are paid but not provisioned
    $pendingOrders = Capsule::table('tblorders')
        ->where('status', 'Pending')
        ->whereIn('paymentmethod', ['paypal', 'creditcard', 'banktransfer'])
        ->where('paymentstatus', 'Paid')
        ->get();

    $provisioner = new ProvisionService();
    $processed = 0;
    $failed = 0;

    foreach ($pendingOrders as $order) {
        $result = $provisioner->processOrder($order->id);

        if ($result['success']) {
            $processed++;
        } else {
            $failed++;
            logActivity("Auto-provisioning failed for order {$order->id}: " . $result['error']);
        }
    }

    echo "Processed: $processed, Failed: $failed\n";
}

function retryFailedProvisioning(): void
{
    // Get hosting accounts with failed status
    $failedHosting = Capsule::table('tblhosting')
        ->where('domainstatus', 'Failed')
        ->where('retry_count', '<', 3)
        ->limit(10)
        ->get();

    foreach ($failedHosting as $hosting) {
        $order = Capsule::table('tblorders')
            ->where('id', $hosting->orderid)
            ->first();

        if (!$order || $order->status !== 'Active') {
            continue;
        }

        // Increment retry count
        Capsule::table('tblhosting')
            ->where('id', $hosting->id)
            ->increment('retry_count');

        // Attempt re-provisioning
        $provisioner = new ProvisionService();
        $result = $provisioner->retryProvisioning($hosting->id);

        if ($result['success']) {
            echo "Retry successful for hosting ID: {$hosting->id}\n";
        } else {
            echo "Retry failed for hosting ID: {$hosting->id}\n";
        }
    }
}

function processServerActions(): void
{
    // Get pending server actions
    $pendingActions = Capsule::table('mod_server_actions')
        ->where('status', 'pending')
        ->where('scheduled_at', '<=', date('Y-m-d H:i:s'))
        ->limit(50)
        ->get();

    foreach ($pendingActions as $action) {
        $processor = new ServerActionProcessor();
        $processor->process($action);

        Capsule::table('mod_server_actions')
            ->where('id', $action->id)
            ->update([
                'status' => 'processed',
                'processed_at' => date('Y-m-d H:i:s')
            ]);
    }

    echo "Processed " . count($pendingActions) . " server actions\n";
}
```

## Step 5: Server Action Processor

```php
<?php
// src/Service/ServerActionProcessor.php

namespace WHMCS\Module\Addon\YourModule\Service;

use WHMCS\Database\Capsule;
use WHMCS\Module\Server;

class ServerActionProcessor
{
    public function process(object $action): array
    {
        $hosting = Capsule::table('tblhosting')
            ->where('id', $action->service_id)
            ->first();

        if (!$hosting) {
            return ['success' => false, 'error' => 'Service not found'];
        }

        $server = Capsule::table('tblservers')
            ->where('id', $hosting->serverid)
            ->first();

        if (!$server) {
            return ['success' => false, 'error' => 'Server not found'];
        }

        // Load server module
        $module = new Server($server->type);
        $module->setServerCredential('hostname', $server->hostname);
        $module->setServerCredential('username', $server->username);
        $module->setServerCredential('password', decrypt($server->password));

        // Execute action based on type
        switch ($action->action_type) {
            case 'suspend':
                return $this->executeSuspend($module, $hosting, $action);
            case 'unsuspend':
                return $this->executeUnsuspend($module, $hosting, $action);
            case 'terminate':
                return $this->executeTerminate($module, $hosting, $action);
            case 'change_package':
                return $this->executeChangePackage($module, $hosting, $action);
            case 'change_password':
                return $this->executeChangePassword($module, $hosting, $action);
            default:
                return ['success' => false, 'error' => 'Unknown action type'];
        }
    }

    private function executeSuspend(Server $module, object $hosting, object $action): array
    {
        try {
            $result = $module->call('SuspendAccount', [
                'serviceid' => $hosting->id
            ]);

            Capsule::table('tblhosting')
                ->where('id', $hosting->id)
                ->update(['domainstatus' => 'Suspended']);

            $this->logAction($action, 'success', $result);

            return ['success' => true, 'result' => $result];
        } catch (\Exception $e) {
            $this->logAction($action, 'failed', $e->getMessage());
            return ['success' => false, 'error' => $e->getMessage()];
        }
    }

    private function executeUnsuspend(Server $module, object $hosting, object $action): array
    {
        try {
            $result = $module->call('UnsuspendAccount', [
                'serviceid' => $hosting->id
            ]);

            Capsule::table('tblhosting')
                ->where('id', $hosting->id)
                ->update(['domainstatus' => 'Active']);

            $this->logAction($action, 'success', $result);

            return ['success' => true, 'result' => $result];
        } catch (\Exception $e) {
            $this->logAction($action, 'failed', $e->getMessage());
            return ['success' => false, 'error' => $e->getMessage()];
        }
    }

    private function executeTerminate(Server $module, object $hosting, object $action): array
    {
        try {
            $result = $module->call('TerminateAccount', [
                'serviceid' => $hosting->id
            ]);

            Capsule::table('tblhosting')
                ->where('id', $hosting->id)
                ->update(['domainstatus' => 'Terminated']);

            $this->logAction($action, 'success', $result);

            return ['success' => true, 'result' => $result];
        } catch (\Exception $e) {
            $this->logAction($action, 'failed', $e->getMessage());
            return ['success' => false, 'error' => $e->getMessage()];
        }
    }

    private function executeChangePackage(Server $module, object $hosting, object $action): array
    {
        $params = json_decode($action->params, true);

        try {
            $result = $module->call('ChangePackage', [
                'serviceid' => $hosting->id,
                'new_package' => $params['new_package_id']
            ]);

            $this->logAction($action, 'success', $result);

            return ['success' => true, 'result' => $result];
        } catch (\Exception $e) {
            $this->logAction($action, 'failed', $e->getMessage());
            return ['success' => false, 'error' => $e->getMessage()];
        }
    }

    private function executeChangePassword(Server $module, object $hosting, object $action): array
    {
        $params = json_decode($action->params, true);

        try {
            $result = $module->call('ChangePassword', [
                'serviceid' => $hosting->id,
                'new_password' => $params['new_password']
            ]);

            $this->logAction($action, 'success', $result);

            return ['success' => true, 'result' => $result];
        } catch (\Exception $e) {
            $this->logAction($action, 'failed', $e->getMessage());
            return ['success' => false, 'error' => $e->getMessage()];
        }
    }

    private function logAction(object $action, string $status, $result): void
    {
        Capsule::table('mod_server_action_logs')->insert([
            'action_id' => $action->id,
            'service_id' => $action->service_id,
            'action_type' => $action->action_type,
            'status' => $status,
            'result' => is_array($result) ? json_encode($result) : $result,
            'created_at' => date('Y-m-d H:i:s')
        ]);
    }
}
```

## Step 6: Provisioning Status Dashboard

```php
<?php
// admin/provisioning_dashboard.php

add_hook('AdminAreaHeaderOutput', 1, function() {
    $stats = getProvisioningStats();

    return "
    <script>
        window.provisioningStats = " . json_encode($stats) . ";
    </script>";
});

function getProvisioningStats(): array
{
    $db = Capsule::connection()->getPdo();

    // Get hosting status counts
    $stmt = $db->query("
        SELECT domainstatus, COUNT(*) as count
        FROM tblhosting
        GROUP BY domainstatus
    ");
    $statusCounts = $stmt->fetchAll(PDO::FETCH_KEY_PAIR);

    // Get recent provisioning
    $recent = Capsule::table('tblhosting')
        ->orderBy('regdate', 'desc')
        ->limit(10)
        ->get();

    // Get failed provisioning
    $failed = Capsule::table('tblhosting')
        ->where('domainstatus', 'Failed')
        ->orderBy('regdate', 'desc')
        ->limit(5)
        ->get();

    // Get provisioning success rate
    $totalProvisioned = Capsule::table('tblhosting')
        ->whereIn('domainstatus', ['Active', 'Suspended', 'Terminated'])
        ->count();

    $failedProvisioned = Capsule::table('tblhosting')
        ->where('domainstatus', 'Failed')
        ->count();

    $successRate = $totalProvisioned + $failedProvisioned > 0
        ? round(($totalProvisioned / ($totalProvisioned + $failedProvisioned)) * 100, 2)
        : 100;

    return [
        'status_counts' => $statusCounts,
        'total_active' => $statusCounts['Active'] ?? 0,
        'total_suspended' => $statusCounts['Suspended'] ?? 0,
        'total_terminated' => $statusCounts['Terminated'] ?? 0,
        'total_failed' => $failedProvisioned,
        'success_rate' => $successRate,
        'recent_provisioning' => $recent,
        'failed_provisioning' => $failed
    ];
}
```

## Verification Checklist

- [ ] Provisioning hooks registered
- [ ] Provision service class implemented
- [ ] Server selection logic working
- [ ] Username/password generation secure
- [ ] Hosting account creation working
- [ ] Server module integration working
- [ ] Provisioning logger configured
- [ ] Failed provisioning retry logic implemented
- [ ] Server action processor working
- [ ] Admin dashboard widget created
- [ ] Test order provisioned successfully
- [ ] Failed scenario handling verified
