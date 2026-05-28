# WHMCS Hooks Module DevKit
# Version: 1.0 | Updated: 2026-05-28

## DevKit Structure

```
devkits/whmcs-hooks-module/
├── hooks.php           # Hook definitions and handlers
├── lib/
│   └── HookHandler.php # Hook handler class
├── templates/
│   └── admin.tpl       # Admin configuration template
└── hooks-config.php    # Hook configuration
```

## Hooks File Template

```php
<?php
/**
 * WHMCS Hooks Module: {Module}
 * DevKit Template
 * 
 * Usage: Include in hooks.php or upload as hooks.php
 */

if (!defined("WHMCS")) {
    die("Direct access denied");
}

// {Module} Configuration
$config = [
    'enabled' => true,
    'log_level' => 'debug', // debug, info, warning, error
    'webhook_url' => '',
    'api_key' => '',
];

/**
 * Hook: ClientAdd
 * Triggered when a new client is created
 */
add_hook('ClientAdd', 1, function(array $vars) use ($config) {
    if (!$config['enabled']) return;
    
    $clientId = $vars['userid'] ?? null;
    $email = $vars['email'] ?? '';
    $firstName = $vars['firstname'] ?? '';
    $lastName = $vars['lastname'] ?? '';
    
    logActivity("{Module}: New client added - ID: {$clientId}, Email: {$email}");
    
    // Your logic here (API call, notification, etc.)
    try {
        // Send webhook notification
        if ($config['webhook_url']) {
            sendWebhook($config['webhook_url'], [
                'event' => 'client_added',
                'client_id' => $clientId,
                'email' => $email,
                'first_name' => $firstName,
                'last_name' => $lastName,
            ]);
        }
    } catch (\Exception $e) {
        logActivity("{Module} Error: " . $e->getMessage());
    }
});

/**
 * Hook: AfterModuleCreate
 * Triggered after a product/service is provisioned
 */
add_hook('AfterModuleCreate', 1, function(array $vars) use ($config) {
    if (!$config['enabled']) return;
    
    $serviceId = $vars['serviceid'] ?? null;
    $module = $vars['module'] ?? '';
    $params = $vars['params'] ?? [];
    
    logActivity("{Module}: Service provisioned - ID: {$serviceId}, Module: {$module}");
    
    // Your logic here
});

/**
 * Hook: AfterModuleSuspend
 * Triggered after a service is suspended
 */
add_hook('AfterModuleSuspend', 1, function(array $vars) use ($config) {
    if (!$config['enabled']) return;
    
    $serviceId = $vars['serviceid'] ?? null;
    logActivity("{Module}: Service suspended - ID: {$serviceId}");
});

/**
 * Hook: AfterModuleUnsuspend
 * Triggered after a service is unsuspended
 */
add_hook('AfterModuleUnsuspend', 1, function(array $vars) use ($config) {
    if (!$config['enabled']) return;
    
    $serviceId = $vars['serviceid'] ?? null;
    logActivity("{Module}: Service unsuspended - ID: {$serviceId}");
});

/**
 * Hook: AfterModuleTerminate
 * Triggered after a service is terminated
 */
add_hook('AfterModuleTerminate', 1, function(array $vars) use ($config) {
    if (!$config['enabled']) return;
    
    $serviceId = $vars['serviceid'] ?? null;
    logActivity("{Module}: Service terminated - ID: {$serviceId}");
});

/**
 * Hook: InvoicePaid
 * Triggered when an invoice is paid
 */
add_hook('InvoicePaid', 1, function(array $vars) use ($config) {
    if (!$config['enabled']) return;
    
    $invoiceId = $vars['invoiceid'] ?? null;
    $amount = $vars['total'] ?? 0;
    
    logActivity("{Module}: Invoice paid - ID: {$invoiceId}, Amount: {$amount}");
});

/**
 * Hook: InvoiceCancelled
 * Triggered when an invoice is cancelled
 */
add_hook('InvoiceCancelled', 1, function(array $vars) use ($config) {
    if (!$config['enabled']) return;
    
    $invoiceId = $vars['invoiceid'] ?? null;
    logActivity("{Module}: Invoice cancelled - ID: {$invoiceId}");
});

/**
 * Hook: TicketOpen
 * Triggered when a support ticket is opened
 */
add_hook('TicketOpen', 1, function(array $vars) use ($config) {
    if (!$config['enabled']) return;
    
    $ticketId = $vars['ticketid'] ?? null;
    $subject = $vars['subject'] ?? '';
    
    logActivity("{Module}: Ticket opened - ID: {$ticketId}, Subject: {$subject}");
});

/**
 * Hook: TicketReply
 * Triggered when a ticket receives a reply
 */
add_hook('TicketReply', 1, function(array $vars) use ($config) {
    if (!$config['enabled']) return;
    
    $ticketId = $vars['ticketid'] ?? null;
    logActivity("{Module}: Ticket reply - ID: {$ticketId}");
});

/**
 * Hook: TicketClose
 * Triggered when a ticket is closed
 */
add_hook('TicketClose', 1, function(array $vars) use ($config) {
    if (!$config['enabled']) return;
    
    $ticketId = $vars['ticketid'] ?? null;
    logActivity("{Module}: Ticket closed - ID: {$ticketId}");
});

/**
 * Hook: DailyCronJob
 * Triggered daily by WHMCS cron
 */
add_hook('DailyCronJob', 1, function(array $vars) use ($config) {
    if (!$config['enabled']) return;
    
    logActivity("{Module}: Daily cron executed");
    
    // Perform daily tasks: cleanup, reports, sync, etc.
});

/**
 * Hook: AfterRegistrarRegistration
 * Triggered after domain registration
 */
add_hook('AfterRegistrarRegistration', 1, function(array $vars) use ($config) {
    if (!$config['enabled']) return;
    
    $domainId = $vars['domainid'] ?? null;
    $domain = $vars['domain'] ?? '';
    logActivity("{Module}: Domain registered - ID: {$domainId}, Domain: {$domain}");
});

/**
 * Hook: AfterRegistrarRenewal
 * Triggered after domain renewal
 */
add_hook('AfterRegistrarRenewal', 1, function(array $vars) use ($config) {
    if (!$config['enabled']) return;
    
    $domainId = $vars['domainid'] ?? null;
    logActivity("{Module}: Domain renewed - ID: {$domainId}");
});

/**
 * Hook: AcceptOrder
 * Triggered when an order is accepted
 */
add_hook('AcceptOrder', 1, function(array $vars) use ($config) {
    if (!$config['enabled']) return;
    
    $orderId = $vars['orderid'] ?? null;
    logActivity("{Module}: Order accepted - ID: {$orderId}");
});

/**
 * Hook: AfterServiceChangePackage
 * Triggered after a service upgrade/downgrade
 */
add_hook('AfterServiceChangePackage', 1, function(array $vars) use ($config) {
    if (!$config['enabled']) return;
    
    $serviceId = $vars['serviceid'] ?? null;
    logActivity("{Module}: Package changed - Service ID: {$serviceId}");
});

/**
 * Hook: AfterPaymentGatewayCheckout
 * Triggered after payment is initiated
 */
add_hook('AfterPaymentGatewayCheckout', 1, function(array $vars) use ($config) {
    if (!$config['enabled']) return;
    
    $invoiceId = $vars['invoiceid'] ?? null;
    logActivity("{Module}: Payment initiated - Invoice ID: {$invoiceId}");
});

/**
 * Helper: Send Webhook
 */
function sendWebhook(string $url, array $data): bool {
    $ch = curl_init();
    curl_setopt_array($ch, [
        CURLOPT_URL => $url,
        CURLOPT_POST => true,
        CURLOPT_POSTFIELDS => json_encode($data),
        CURLOPT_HTTPHEADER => ['Content-Type: application/json'],
        CURLOPT_RETURNTRANSFER => true,
        CURLOPT_TIMEOUT => 30,
    ]);
    
    $response = curl_exec($ch);
    $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
    curl_close($ch);
    
    return $httpCode >= 200 && $httpCode < 300;
}
```

## Hook Handler Class

```php
<?php
namespace {Module};

class HookHandler {
    private array $config;
    private string $logPath;
    
    public function __construct(array $config = []) {
        $this->config = array_merge([
            'enabled' => true,
            'log_level' => 'info',
            'webhook_url' => '',
            'api_key' => '',
        ], $config);
        
        $this->logPath = ROOTDIR . '/storage/logs/module_hooks.log';
    }
    
    public function handleClientAdd(array $vars): void {
        $this->log('info', 'Client added', $vars);
        
        if ($this->config['webhook_url']) {
            $this->sendWebhook('client_added', $vars);
        }
    }
    
    public function handleServiceCreated(array $vars): void {
        $this->log('info', 'Service created', $vars);
    }
    
    public function handleInvoicePaid(array $vars): void {
        $this->log('info', 'Invoice paid', $vars);
    }
    
    public function handleTicketOpen(array $vars): void {
        $this->log('info', 'Ticket opened', $vars);
    }
    
    public function handleDailyCron(array $vars): void {
        $this->log('info', 'Daily cron executed', $vars);
        $this->performDailyTasks();
    }
    
    private function sendWebhook(string $event, array $data): bool {
        if (empty($this->config['webhook_url'])) {
            return false;
        }
        
        $payload = [
            'event' => $event,
            'timestamp' => date('c'),
            'data' => $data,
        ];
        
        $ch = curl_init();
        curl_setopt_array($ch, [
            CURLOPT_URL => $this->config['webhook_url'],
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => json_encode($payload),
            CURLOPT_HTTPHEADER => [
                'Content-Type: application/json',
                'X-API-Key: ' . $this->config['api_key'],
            ],
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_TIMEOUT => 30,
        ]);
        
        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);
        
        return $httpCode >= 200 && $httpCode < 300;
    }
    
    private function log(string $level, string $message, array $context = []): void {
        if ($this->shouldLog($level)) {
            $entry = date('Y-m-d H:i:s') . " [{$level}] {$message}";
            if (!empty($context)) {
                $entry .= ' ' . json_encode($context);
            }
            $entry .= PHP_EOL;
            
            file_put_contents($this->logPath, $entry, FILE_APPEND);
        }
    }
    
    private function shouldLog(string $level): bool {
        $levels = ['debug' => 0, 'info' => 1, 'warning' => 2, 'error' => 3];
        $configLevel = $levels[$this->config['log_level']] ?? 1;
        $msgLevel = $levels[$level] ?? 1;
        
        return $msgLevel >= $configLevel;
    }
    
    private function performDailyTasks(): void {
        // Implement daily tasks here
    }
}
```

## Admin Configuration Template

```smarty
<div class="module-console">
    <div class="row">
        <div class="col-md-12">
            <div class="alert alert-info">
                <i class="fa fa-info-circle"></i>
                Configure your {Module} hooks settings below.
            </div>
        </div>
    </div>

    <form method="post" action="{$smarty.server.PHP_SELF}?action=module_settings&module={module}">
        <input type="hidden" name="csrf_token" value="{$csrf_token}">
        
        <div class="panel panel-default">
            <div class="panel-heading">
                <h3 class="panel-title">General Settings</h3>
            </div>
            <div class="panel-body">
                <div class="form-group">
                    <label for="enabled">Enable Hooks</label>
                    <select name="enabled" id="enabled" class="form-control">
                        <option value="1" {$enabled.selected}>Enabled</option>
                        <option value="0" {if !$enabled}selected{/if}>Disabled</option>
                    </select>
                </div>
                
                <div class="form-group">
                    <label for="log_level">Log Level</label>
                    <select name="log_level" id="log_level" class="form-control">
                        <option value="debug" {if $log_level == 'debug'}selected{/if}>Debug</option>
                        <option value="info" {if $log_level == 'info'}selected{/if}>Info</option>
                        <option value="warning" {if $log_level == 'warning'}selected{/if}>Warning</option>
                        <option value="error" {if $log_level == 'error'}selected{/if}>Error</option>
                    </select>
                </div>
            </div>
        </div>
        
        <div class="panel panel-default">
            <div class="panel-heading">
                <h3 class="panel-title">Webhook Configuration</h3>
            </div>
            <div class="panel-body">
                <div class="form-group">
                    <label for="webhook_url">Webhook URL</label>
                    <input type="url" name="webhook_url" id="webhook_url" 
                           class="form-control" value="{$webhook_url}"
                           placeholder="https://api.example.com/webhook">
                </div>
                
                <div class="form-group">
                    <label for="api_key">API Key</label>
                    <input type="password" name="api_key" id="api_key" 
                           class="form-control" value="{$api_key}">
                </div>
            </div>
        </div>
        
        <div class="panel panel-default">
            <div class="panel-heading">
                <h3 class="panel-title">Active Hooks</h3>
            </div>
            <div class="panel-body">
                <table class="table table-striped">
                    <thead>
                        <tr>
                            <th>Hook Name</th>
                            <th>Status</th>
                            <th>Last Triggered</th>
                        </tr>
                    </thead>
                    <tbody>
                        {foreach $hooks as $hook}
                        <tr>
                            <td>{$hook.name}</td>
                            <td>
                                <span class="label label-{$hook.status_class}">
                                    {$hook.status}
                                </span>
                            </td>
                            <td>{$hook.last_triggered}</td>
                        </tr>
                        {/foreach}
                    </tbody>
                </table>
            </div>
        </div>
        
        <div class="panel panel-default">
            <div class="panel-heading">
                <h3 class="panel-title">Recent Activity Log</h3>
            </div>
            <div class="panel-body">
                <pre>{$activity_log}</pre>
            </div>
        </div>
        
        <button type="submit" class="btn btn-primary">
            <i class="fa fa-save"></i> Save Settings
        </button>
    </form>
</div>
```

## Checklist

```
Pre-Dev:
□ Identify which WHMCS hooks to use
□ Document hook parameters available
□ Plan data flow and actions for each hook
□ Determine if hooks need configuration UI
□ Identify external service integrations

Development:
□ Create hooks.php file
□ Define configuration array
□ Add ClientAdd hook handler
□ Add AfterModuleCreate hook handler
□ Add AfterModuleSuspend hook handler
□ Add AfterModuleUnsuspend hook handler
□ Add AfterModuleTerminate hook handler
□ Add InvoicePaid hook handler
□ Add InvoiceCancelled hook handler
□ Add TicketOpen hook handler
□ Add TicketReply hook handler
□ Add TicketClose hook handler
□ Add DailyCronJob hook handler
□ Create HookHandler class (optional)
□ Add webhook helper function
□ Add logging functionality
□ Create admin configuration template

Testing:
□ Test each hook fires correctly
□ Verify hook parameters are correct
□ Test error handling in hooks
□ Verify logging works
□ Test webhook delivery
□ Test with real WHMCS events
□ Test cron hook execution
```

## Common WHMCS Hooks Reference

| Hook Name | Trigger | Parameters |
|-----------|---------|------------|
| ClientAdd | New client created | userid, email, firstname, lastname |
| ClientEdit | Client updated | userid |
| ClientDelete | Client deleted | userid |
| AfterModuleCreate | Service provisioned | serviceid, params |
| AfterModuleSuspend | Service suspended | serviceid |
| AfterModuleUnsuspend | Service unsuspended | serviceid |
| AfterModuleTerminate | Service terminated | serviceid |
| InvoicePaid | Invoice paid | invoiceid, total, userid |
| InvoiceCancelled | Invoice cancelled | invoiceid |
| TicketOpen | Ticket opened | ticketid, subject, body |
| TicketReply | Ticket replied | ticketid |
| TicketClose | Ticket closed | ticketid |
| DailyCronJob | Daily cron runs | - |
| AcceptOrder | Order accepted | orderid |
| AfterServiceChangePackage | Package changed | serviceid |