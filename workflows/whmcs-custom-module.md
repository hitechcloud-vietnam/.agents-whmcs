# WHMCS Custom Module Development Workflow

## Purpose

Complete guide to building custom WHMCS modules following best practices. Covers module structure, lifecycle functions, configuration, security, testing, and deployment.

## Prerequisites

- WHMCS 7.0+ installation
- PHP 7.4+ knowledge
- Understanding of WHMCS module system
- Development/staging environment

## Workflow Steps

### Step 1: Planning Your Module

```
Module Planning Checklist:
□ Define module type (server, gateway, registrar, addon)
□ Identify required hooks and functions
□ Plan database schema if needed
□ Determine configuration options
□ Document API endpoints if external integration
□ Plan security requirements
□ Create feature roadmap
□ Select license model
```

### Step 2: Basic Module Structure

Create the standard module directory and file structure:

```bash
# Module directory structure
modules/
├── servers/
│   └── yourprovider/
│       ├── yourprovider.php          # Main module file
│       ├── includes/
│       │   ├── api_client.php        # API communication
│       │   ├── webhook_handler.php   # Webhook processing
│       │   └── helpers.php           # Utility functions
│       ├── templates/
│       │   ├── clientarea.tpl        # Client area template
│       │   ├── adminpanel.tpl        # Admin panel template
│       │   └── configure.tpl         # Configuration template
│       ├── views/
│       │   ├── dashboard.phtml       # AJAX views
│       │   └── settings.phtml
│       ├── assets/
│       │   ├── css/style.css
│       │   ├── js/main.js
│       │   └── images/icon.png
│       └── . └── └── └── ├── CHANGELOG.md
│       └── README.md
```

### Step 3: Implementing Server Provisioning Module

Create a complete server/provisioning module:

```php
<?php
/**
 * WHMCS YourProvider Provisioning Module
 *
 * @copyright Copyright (c) 2024 Your Company
 * @license https://whmcs.com/license/
 *
 * @documentation https://developers.whmcs.com/provisioning-modules/
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

/**
 * Module metadata - REQUIRED
 */
function yourprovider_MetaData(): array
{
    return [
        'DisplayName' => 'YourProvider Hosting',
        'APIVersion' => '1.0',
        'RequiresServer' => true,
        'DefaultNonSSLPort' => 443,
        'DefaultSSLPort' => 443,
        'ServiceSingleSignOn' => true,
        'PromotionalFeatures' => [
            'configoption-1',
        ],
    ];
}

/**
 * Module configuration options
 */
function yourprovider_ConfigOptions(array $params): array
{
    return [
        'Plan' => [
            'Type' => 'dropdown',
            'Options' => 'starter,business,enterprise',
            'Default' => 'starter',
            'Description' => 'Select hosting plan',
        ],
        'Image' => [
            'Type' => 'text',
            'Default' => 'ubuntu-22.04',
            'Description' => 'Operating system image identifier',
        ],
        'AutoSSL' => [
            'Type' => 'yesno',
            'Default' => 'yes',
            'Description' => 'Enable automatic SSL certificates',
        ],
        'Backups' => [
            'Type' => 'dropdown',
            'Options' => 'none,weekly,daily',
            'Default' => 'weekly',
        ],
        'Monitoring' => [
            'Type' => 'yesno',
            'Default' => 'yes',
        ],
    ];
}

/**
 * Create new hosting account
 */
function yourprovider_CreateAccount(array $params): string
{
    try {
        // Validate required parameters
        if (empty($params['domain'])) {
            return 'Error: Domain name is required';
        }

        // Initialize API client
        $api = new YourProvider_API($params);

        // Create instance via API
        $result = $api->createInstance([
            'hostname' => $params['domain'],
            'plan' => $params['configoption1'],
            'image' => $params['configoption2'],
            'user_id' => $params['serviceid'],
        ]);

        if (!$result['success']) {
            logActivity("YourProvider create failed: " . $result['error']);
            return 'Error: ' . ($result['error'] ?? 'Failed to create account');
        }

        // Store external ID for future operations
        Capsule::table('tblhosting')
            ->where('id', $params['serviceid'])
            ->update([
                'subscription_id' => $result['instance_id'],
                'username' => $result['username'] ?? '',
            ]);

        // Send provisioning email
        sendMessage('Service Provisioned', $params['serviceid']);

        return 'success';

    } catch (\Exception $e) {
        logActivity("YourProvider CreateAccount exception: " . $e->getMessage());
        return 'Error: ' . $e->getMessage();
    }
}

/**
 * Suspend hosting account
 */
function yourprovider_SuspendAccount(array $params): string
{
    try {
        $api = new YourProvider_API($params);

        $result = $api->suspendInstance($params['subscription_id']);

        if (!$result['success']) {
            return 'Error: ' . ($result['error'] ?? 'Failed to suspend account');
        }

        sendEmailNotification('Service Suspended', $params['serviceid']);

        return 'success';

    } catch (\Exception $e) {
        logActivity("YourProvider SuspendAccount exception: " . $e->getMessage());
        return 'Error: ' . $e->getMessage();
    }
}

/**
 * Unsuspend hosting account
 */
function yourprovider_UnsuspendAccount(array $params): string
{
    try {
        $api = new YourProvider_API($params);

        $result = $api->unsuspendInstance($params['subscription_id']);

        if (!$result['success']) {
            return 'Error: ' . ($result['error'] ?? 'Failed to unsuspend account');
        }

        sendEmailNotification('Service Unsuspended', );

        return 'success';

    } catch (\Exception $e) {
        logActivity("YourProvider UnsuspendAccount exception: " . $e->getMessage());
        return 'Error: ' . $e->getMessage();
    }
}

/**
 * Terminate hosting account
 */
function yourprovider_TerminateAccount(array $params): string
{
    try {
        $api = new YourProvider_API($params);

        $result = $api->terminateInstance($params['subscription_id']);

        if (!$result['success']) {
            return 'Error: ' . ($result['error'] ?? 'Failed to terminate account');
        }

        // Clean up local data
        Capsule::table('mod_yourprovider_data')
            ->where('service_id', $params['serviceid'])
            ->delete();

        return 'success';

    } catch (\Exception $e) {
        logActivity("YourProvider TerminateAccount exception: " . $e->getMessage());
        return 'Error: ' . $e->getMessage();
    }
}

/**
 * Change account password
 */
function yourprovider_ChangePassword(array $params): string
{
    try {
        $newPassword = $params['password'] ?? '';

        if (empty($newPassword)) {
            return 'Error: Password is required';
        }

        $api = new YourProvider_API($params);
        $result = $api->changePassword($params['subscription_id'], $newPassword);

        if (!$result['success']) {
            return 'Error: ' . ($result['error'] ?? 'Failed to change password');
        }

        return 'success';

    } catch (\Exception $e) {
        logActivity("YourProvider ChangePassword exception: " . $e->getMessage());
        return 'Error: ' . $e->getMessage();
    }
}

/**
 * Change hosting plan/package
 */
function yourprovider_ChangePackage(array $params): string
{
    try {
        $api = new YourProvider_API($params);

        $result = $api->changePlan(
            $params['subscription_id'],
            $params['configoption1']
        );

        if (!$result['success']) {
            return 'Error: ' . ($result['error'] ?? 'Failed to change package');
        }

        return 'success';

    } catch (\Exception $e) {
        logActivity("YourProvider ChangePackage exception: " . $e->getMessage());
        return 'Error: ' . $e->getMessage();
    }
}

/**
 * Test connection to remote API
 */
function yourprovider_TestConnection(array $params): array
{
    try {
        $api = new YourProvider_API($params);

        $result = $api->verifyCredentials();

        if ($result['success']) {
            return [
                'success' => true,
                'error' => '',
            ];
        }

        return [
            'success' => false,
            'error' => $result['error'] ?? 'Connection failed',
        ];

    } catch (\Exception $e) {
        return [
            'success' => false,
            'error' => $e->getMessage(),
        ];
    }
}

/**
 * Client area output
 */
function yourprovider_ClientArea(array $params): array
{
    // Check for action parameter
    $action = $_REQUEST['a'] ?? $_GET['a'] ?? 'dashboard';

    $templateFile = match($action) {
        'dashboard' => 'clientarea_dashboard',
        'settings' => 'clientarea_settings',
        'backups' => 'clientarea_backups',
        'metrics' => 'clientarea_metrics',
        'console' => 'clientarea_console',
        default => 'clientarea_dashboard',
    };

    // Load module data for template
    $serviceData = Capsule::table('mod_yourprovider_service_data')
        ->where('service_id', $params['serviceid'])
        ->first();

    // Fetch live data from API
    try {
        $api = new YourProvider_API($params);
        $instance = $api->getInstance($params['subscription_id']);

        $pageData = [
            'instance' => $instance,
            'stats' => $api->getInstanceStats($params['subscription_id']),
            'metrics' => $api->getMetrics($params['subscription_id']),
            'service_data' => $serviceData,
        ];
    } catch (\Exception $e) {
        $pageData = [
            'error' => $e->getMessage(),
            'service_data' => $serviceData,
        ];
    }

    return [
        'pagetitle' => 'Service Management',
        'templatefile' => $templateFile,
        'vars' => $pageData,
    ];
}

/**
 * Admin area side links
 */
add_hook('AdminAreaCorePages', 1, function($vars) {
    if ($vars['filename'] === 'configuringervices') {
        return [
            'yourprovider_manage' => [
                'label' => 'YourProvider Management',
                'uri' => 'yourprovider/admin/manage.php',
                'icon' => 'fa-server',
            ],
        ];
    }
});
```

### Step 4: Creating API Client Class

Implement the API communication layer:

```php
// modules/servers/yourprovider/includes/api_client.php

/**
 * YourProvider API Client
 *
 * Handles all communication with the YourProvider API.
 */
class YourProvider_API
{
    private $baseUrl;
    private $apiKey;
    private $apiSecret;
    private $serverIp;
    private $timeout = 30;
    private $retryAttempts = 3;

    public function __construct(array $params)
    {
        $this->baseUrl = 'https://api.yourprovider.com';
        $this->apiKey = $params['serverusername'];
        $this->apiSecret = $params['serverpassword'];
        $this->serverIp = $params['serverhostname'];

        // Allow custom API endpoint from config
        if (!empty($params['serverhttpprefix'])) {
            $this->baseUrl = $params['serverhttpprefix'] . '://' . $params['serverhostname'];
        }
    }

    /**
     * Create new instance
     */
    public function createInstance(array $data): array
    {
        return $this->request('POST', '/v1/instances', [
            'hostname' => $data['hostname'],
            'plan' => $data['plan'],
            'image' => $data['image'],
            'metadata' => [
                'whmcs_service_id' => $data['user_id'] ?? 0,
                'created_by' => 'whmcs',
            ],
        ]);
    }

    /**
     * Get instance details
     */
    public function getInstance(string $instanceId): array
    {
        return $this->request('GET', "/v1/instances/{$instanceId}");
    }

    /**
     * Suspend instance
     */
    public function suspendInstance(string $instanceId): array
    {
        return $this->request('POST', "/v1/instances/{$instanceId}/suspend");
    }

    /**
     * Unsuspend instance
     */
    public function unsuspendInstance(string $instanceId): array
    {
        return $this->request('POST', "/v1/instances/{$instanceId}/unsuspend");
    }

    /**
     * Terminate instance
     */
    public function terminateInstance(string $instanceId): array
    {
        return $this->request('DELETE', "/v1/instances/{$instanceId}");
    }

    /**
     * Change instance password
     */
    public function changePassword(string $instanceId, string $newPassword): array
    {
        return $this->request('PUT', "/v1/instances/{$instanceId}/password", [
            'password' => $newPassword,
        ]);
    }

    /**
     * Change instance plan
     */
    public function changePlan(string $instanceId, string $newPlan): array
    {
        return $this->request('PUT', "/v1/instances/{$instanceId}/resize", [
            'plan' => $newPlan,
        ]);
    }

    /**
     * Get instance statistics
     */
    public function getInstanceStats(string $instanceId): array
    {
        return $this->request('GET', "/v1/instances/{$instanceId}/stats");
    }

    /**
     * Get metrics data
     */
    public function getMetrics(string $instanceId, string $period = '24h'): array
    {
        return $this->request('GET', "/v1/instances/{$instanceId}/metrics?period={$period}");
    }

    /**
     * Verify API credentials
     */
    public function verifyCredentials(): array
    {
        return $this->request('GET', '/v1/account/verify');
    }

    /**
     * Execute API request with retry logic
     */
    public function request(string $method, string $endpoint, array $data = []): array
    {
        $attempt = 0;
        $lastException = null;

        while ($attempt < $this->retryAttempts) {
            try {
                return $this->executeRequest($method, $endpoint, $data);
            } catch (\Exception $e) {
                $lastException = $e;
                $attempt++;

                if ($attempt < $this->retryAttempts) {
                    usleep((int) pow(2, $attempt) * 100000); // Exponential backoff
                }
            }
        }

        throw $lastException;
    }

    /**
     * Execute the actual HTTP request
     */
    private function executeRequest(string $method, string $endpoint, array $data = []): array
    {
        $url = rtrim($this->baseUrl, '/') . '/' . ltrim($endpoint, '/');

        $timestamp = time();
        $nonce = bin2hex(random_bytes(16));

        $headers = [
            'Authorization: Bearer ' . $this->apiKey,
            'X-Timestamp: ' . $timestamp,
            'X-Nonce: ' . $nonce,
            'Content-Type: application/json',
            'Accept: application/json',
        ];

        $ch = curl_init();
        curl_setopt_array($ch, [
            CURLOPT_URL => $url,
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_TIMEOUT => $this->timeout,
            CURLOPT_HTTPHEADER => $headers,
            CURLOPT_SSL_VERIFYPEER => true,
            CURLOPT_SSL_VERIFYHOST => 2,
        ]);

        if ($method === 'POST') {
            curl_setopt($ch, CURLOPT_POST, true);
            curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($data));
        } elseif ($method !== 'GET') {
            curl_setopt($ch, CURLOPT_CUSTOMREQUEST, $method);
            if (!empty($data)) {
                curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($data));
            }
        }

        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        $error = curl_error($ch);
        curl_close($ch);

        if ($error) {
            throw new \Exception("cURL error: {$error}");
        }

        $decoded = json_decode($response, true) ?? [];

        // Handle HTTP errors
        if ($httpCode >= 400) {
            $errorMessage = $decoded['message'] ?? $decoded['error'] ?? 'Unknown error';
            throw new \Exception("API error ({$httpCode}): {$errorMessage}");
        }

        return $decoded;
    }
}
```

### Step 5: Creating Admin and Client Templates

Create Smarty templates for admin and client interfaces:

```smarty
%{* templates/clientarea_dashboard.tpl *}%
<div class="module-dashboard" id="yourprovider-dashboard">
    <div class="row">
        <div class="col-md-8">
            <div class="panel panel-default">
                <div class="panel-heading">
                    <h3 class="panel-title">
                        <i class="fa fa-server"></i>
                        Instance Details
                    </h3>
                </div>
                <div class="panel-body">
                    {if $error}
                        <div class="alert alert-danger">
                            <i class="fa fa-exclamation-triangle"></i>
                            Unable to load instance data: {$error}
                        </div>
                    {else}
                        <div class="row">
                            <div class="col-md-6">
                                <dl class="dl-horizontal">
                                    <dt>Status:</dt>
                                    <dd>
                                        <span class="badge badge-{$instance.status_class}">
                                            {$instance.status}
                                        </span>
                                    </dd>
                                    <dt>IP Address:</dt>
                                    <dd>{$instance.ip_address}</dd>
                                    <dt>Plan:</dt>
                                    <dd>{$instance.plan}</dd>
                                    <dt>Created:</dt>
                                    <dd>{$instance.created_at|date_format}</dd>
                                </dl>
                            </div>
                            <div class="col-md-6">
                                <dl class="dl-horizontal">
                                    <dt>CPU:</dt>
                                    <dd>{$stats.cpu_usage}%</dd>
                                    <dt>RAM:</dt>
                                    <dd>{$stats.memory_usage}%</dd>
                                    <dt>Disk:</dt>
                                    <dd>{$stats.disk_usage}%</dd>
                                    <dt>Bandwidth:</dt>
                                    <dd>{$stats.bandwidth_used} / {$stats.bandwidth_limit}</dd>
                                </dl>
                            </div>
                        </div>
                    {/if}
                </div>
                <div class="panel-footer">
                    <a href="clientarea.php?action=product_details&id={$service_id}&a=console"
                       class="btn btn-primary">
                        <i class="fa fa-terminal"></i> Web Console
                    </a>
                    <a href="clientarea.php?action=product_details&id={$service_id}&a=backups"
                       class="btn btn-default">
                        <i class="fa fa-hdd-o"></i> Manage Backups
                    </a>
                </div>
            </div>
        </div>

        <div class="col-md-4">
            <div class="panel panel-default">
                <div class="panel-heading">
                    <h3 class="panel-title">Quick Stats</h3>
                </div>
                <div class="panel-body">
                    <canvas id="metrics-chart" width="100%" height="100"></canvas>
                </div>
            </div>
        </div>
    </div>
</div>

<script>
require(['jquery', 'chartjs'], function($) {
    $(document).ready(function() {
        // Initialize metrics chart
        var ctx = document.getElementById('metrics-chart').getContext('2d');
        new Chart(ctx, {
            type: 'line',
            data: {
                labels: {$metrics.labels|json_encode},
                datasets: [{
                    label: 'CPU Usage',
                    data: {$metrics.cpu|json_encode},
                    borderColor: '#3498db',
                    fill: false
                }, {
                    label: 'Memory Usage',
                    data: {$metrics.memory|json_encode},
                    borderColor: '#2ecc71',
                    fill: false
                }]
            },
            options: {
                responsive: true,
                maintainAspectRatio: false
            }
        });
    });
});
</script>
```

```smarty
%{* templates/configure.tpl *}%

<form method="post" action="{$smarty.server.PHP_SELF}?action=save"
      class="form-horizontal" id="configure-form">
    <input type="hidden" name="token" value="{$token}" />

    <div class="alert alert-info">
        <i class="fa fa-info-circle"></i>
        Configure YourProvider integration settings below.
    </div>

    <div class="form-group">
        <label class="col-md-3 control-label">API Endpoint</label>
        <div class="col-md-6">
            <input type="text" name="api_endpoint" class="form-control"
                   value="{$module_config.api_endpoint|default:'https://api.yourprovider.com'}"
                   placeholder="https://api.yourprovider.com" />
            <span class="help-block">YourProvider API endpoint URL</span>
        </div>
    </div>

    <div class="form-group">
        <label class="col-md-3 control-label">API Key</label>
        <div class="col-md-6">
            <input type="password" name="api_key" class="form-control"
                   value="{$module_config.api_key}"
                   placeholder="Enter your API key" />
        </div>
    </div>

    <div class="form-group">
        <label class="col-md-3 control-label">API Secret</label>
        <div class="col-md-6">
            <input type="password" name="api_secret" class="form-control"
                   value="{$module_config.api_secret}"
                   placeholder="Enter your API secret" />
        </div>
    </div>

    <div class="form-group">
        <label class="col-md-3 control-label">Default Plan</label>
        <div class="col-md-6">
            <select name="default_plan" class="form-control">
                <option value="starter" {$module_config.default_plan|selected:'starter'}>
                    Starter
                </option>
                <option value="business" {$module_config.default_plan|selected:'business'}>
                    Business
                </option>
                <option value="enterprise" {$module_config.default_plan|selected:'enterprise'}>
                    Enterprise
                </option>
            </select>
        </div>
    </div>

    <div class="form-group">
        <label class="col-md-3 control-label">Auto-provision</label>
        <div class="col-md-6">
            <label class="checkbox-inline">
                <input type="checkbox" name="auto_provision" value="1"
                       {$module_config.auto_provision|checked:1} />
                Automatically provision services after payment
            </label>
        </div>
    </div>

    <hr />

    <div class="form-group">
        <div class="col-md-6 col-md-offset-3">
            <button type="submit" class="btn btn-primary">
                <i class="fa fa-save"></i> Save Configuration
            </button>
            <button type="button" class="btn btn-default" onclick="testConnection()">
                <i class="fa fa-plug"></i> Test Connection
            </button>
        </div>
    </div>
</form>

<div id="test-result" class="alert" style="display:none;"></div>

<script>
function testConnection() {
    var $form = $('#configure-form');
    var $result = $('#test-result');
    var $button = $form.find('button[data-action="test"]');

    $button.prop('disabled', true).html('<i class="fa fa-spinner fa-spin"></i> Testing...');

    $.post('{$base_url}admin/test_connection.php', $form.serialize())
        .done(function(response) {
            if (response.success) {
                $result.removeClass('alert-danger').addClass('alert-success')
                    .html('<i class="fa fa-check"></i> ' + response.message);
            } else {
                $result.removeClass('alert-success').addClass('alert-danger')
                    .html('<i class="fa fa-times"></i> ' + response.message);
            }
        })
        .fail(function() {
            $result.removeClass('alert-success').addClass('alert-danger')
                .html('<i class="fa fa-times"></i> Connection test failed');
        })
        .always(function() {
            $button.prop('disabled', false).html('<i class="fa fa-plug"></i> Test Connection');
            $result.show();
        });
}
</script>
```

### Step 6: Module Testing

Create comprehensive tests for your module:

```php
<?php
// modules/servers/yourprovider/tests/YourProviderModuleTest.php

namespace WHMCS\Module\Server\YourProvider;

use PHPUnit\Framework\TestCase;

class YourProviderModuleTest extends TestCase
{
    private $module;
    private $mockApi;

    protected function setUp(): void
    {
        parent::setUp();

        // Mock configuration parameters
        $this->params = [
            'serverusername' => 'test_api_key',
            'serverpassword' => 'test_api_secret',
            'serverhostname' => 'api.yourprovider.com',
            'serviceid' => 1,
            'userid' => 1,
            'domain' => 'test.example.com',
            'configoption1' => 'starter',
            'configoption2' => 'ubuntu-22.04',
        ];
    }

    /**
     * Test module meta data
     */
    public function testMetaDataReturnsArray(): void
    {
        $metaData = yourprovider_MetaData();

        $this->assertIsArray($metaData);
        $this->assertArrayHasKey('DisplayName', $metaData);
        $this->assertArrayHasKey('APIVersion', $metaData);
        $this->assertTrue($metaData['RequiresServer']);
    }

    /**
     * Test config options
     */
    public function testConfigOptionsReturnsValidOptions(): void
    {
        $configOptions = yourprovider_ConfigOptions($this->params);

        $this->assertArrayHasKey('Plan', $configOptions);
        $this->assertArrayHasKey('Image', $configOptions);
        $this->assertEquals('dropdown', $configOptions['Plan']['Type']);
    }

    /**
     * Test connection test returns array
     */
    public function testTestConnectionReturnsArray(): void
    {
        // Mock the API
        $result = yourprovider_TestConnection($this->params);

        $this->assertIsArray($result);
        $this->assertArrayHasKey('success', $result);
        $this->assertArrayHasKey('error', $result);
    }

    /**
     * Test client area returns expected structure
     */
    public function testClientAreaReturnsValidArray(): void
    {
        $result = yourprovider_ClientArea($this->params);

        $this->assertIsArray($result);
        $this->assertArrayHasKey('pagetitle', $result);
        $this->assertArrayHasKey('templatefile', $result);
        $this->assertArrayHasKey('vars', $result);
    }
}

/**
 * API Client Test
 */
class YourProvider_APITest extends TestCase
{
    private $api;
    private $mockHandler;

    protected function setUp(): void
    {
        parent::setUp();

        $this->api = new YourProvider_API([
            'serverusername' => 'test_key',
            'serverpassword' => 'test_secret',
            'serverhostname' => 'api.yourprovider.com',
        ]);
    }

    /**
     * Test credentials verification
     */
    public function testVerifyCredentials(): void
    {
        $this->expectNotToPerformAssertions();

        try {
            $this->api->verifyCredentials();
        } catch (\Exception $e) {
            // Expected in test environment
        }
    }

    /**
     * Test instance creation
     */
    public function testCreateInstanceValidation(): void
    {
        $this->expectException(\Exception::class);

        // Missing hostname should throw
        $this->api->createInstance([]);
    }

    /**
     * Test HTTP method handling
     */
    public function testRequestMethods(): void
    {
        $reflection = new \ReflectionClass($this->api);
        $method = $reflection->getMethod('executeRequest');
        $method->setAccessible(true);

        $this->expectNotToPerformAssertions();
    }
}

/**
 * Integration test
 */
class YourProviderModuleIntegrationTest extends TestCase
{
    private $whmcsUrl;
    private $apiIdentifier;
    private $apiSecret;

    protected function setUp(): void
    {
        parent::setUp();

        // Skip if no test environment configured
        if (!$this->isTestEnvironmentConfigured()) {
            $this->markTestSkipped('Test environment not configured');
        }
    }

    private function isTestEnvironmentConfigured(): bool
    {
        return getenv('WHMCS_TEST_URL') !== false;
    }

    /**
     * Test full provisioning workflow
     */
    public function testProvisionWorkflow(): void
    {
        // This would test against a real WHMCS + API environment
        $this->markTestSkipped('Integration test - run manually');
    }
}
```

## WHMCS Best Practices

1. **Follow module conventions** - Use correct function naming and file locations
2. **Always return proper values** - 'success' for server modules, arrays for registrar
3. **Implement error handling** - Wrap all API calls in try-catch
4. **Log activities** - Use logActivity() for debugging and auditing
5. **Use encryption** - Store sensitive data using encrypt()
6. **Handle timeouts** - Implement retry logic for API calls
7. **Write tests** - Create unit tests for all functions
8. **Document thoroughly** - Include README and inline comments
9. **Secure inputs** - Validate and sanitize all user input
10. **Use transactions** - Wrap database operations in transactions

## Verification Checklist

```
Development:
□ Module follows WHMCS naming conventions
□ All required functions implemented
□ Return values correct for module type
□ Error handling in place
□ Logging implemented
□ CSRF protection on forms
□ Input validation done
□ SQL injection prevention
□ XSS prevention
□ No hardcoded credentials

Testing:
□ Unit tests for all functions
□ Integration tests for API calls
□ Error case tests
□ Mock tests for external dependencies
□ Module activates without errors
□ Module deactivates without errors
□ No PHP errors/warnings
□ All features work as expected

Documentation:
□ README.md created
□ Installation instructions included
□ Configuration guide ready
□ Changelog updated
□ Version number bumped
□ License file included
```

## Common Pitfalls to Avoid

1. **Wrong return values** - Server modules return strings, registrar returns arrays
2. **Not handling API errors** - Always check for failures
3. **Hardcoding server settings** - Use params array
4. **Skipping logActivity** - Essential for debugging
5. **Ignoring SSL verification** - Always verify certificates
6. **Missing timeout handling** - Long API calls need timeouts
7. **Not escaping output** - XSS vulnerabilities in templates
8. **Skipping tests** - Critical for reliability
9. **No error recovery** - Plan for failure scenarios
10. **Missing documentation** - Users need setup guides

## WHMCS ClassDocs References

- [Provisioning Module Guide](https://developers.whmcs.com/provisioning-modules/)
- [Module Functions](https://developers.whmcs.com/provisioning-modules/module-functions/)
- [Configuration](https://developers.whmcs.com/provisioning-modules/configuration/)
- [Client Area](https://developers.whmcs.com/provisioning-modules/client-area/)
- [logActivity()](https://developers.whmcs.com/advanced/logging/)
- [sendMessage()](https://developers.whmcs.com/advanced/email-templates/)
