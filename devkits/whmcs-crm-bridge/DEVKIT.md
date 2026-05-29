# WHMCS CRM Bridge Module

## Overview
Bidirectional CRM synchronization module for client data management.

## Module File: crm_bridge.php

```php
<?php
/**
 * WHMCS CRM Bridge Module
 * 
 * @package    WHMCS\Module\Addons
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

use WHMCS\Database\Capsule;

class WHMCS_CRM_Bridge
{
    protected $config;

    public function __construct()
    {
        $this->config = require __DIR__ . '/config.php';
    }

    /**
     * Sync client to CRM
     */
    public function syncClientToCRM(int $clientId, string $action = 'create'): bool
    {
        try {
            $client = Capsule::table('tblclients')
                ->where('id', $clientId)
                ->first();

            if (!$client) {
                return false;
            }

            $data = $this->formatClientData($client);
            $data['action'] = $action;
            $data['timestamp'] = date('c');

            $response = $this->sendWebhook($data);

            // Log sync
            Capsule::table('mod_crm_sync_log')->insert([
                'client_id' => $clientId,
                'direction' => 'outbound',
                'action' => $action,
                'status' => $response['success'] ? 'success' : 'failed',
                'response' => json_encode($response),
                'created_at' => date('Y-m-d H:i:s'),
            ]);

            return $response['success'];

        } catch (\Exception $e) {
            logModuleCall('CRMBridge', 'SyncClientToCRM', ['client_id' => $clientId], $e->getMessage(), '');
            return false;
        }
    }

    /**
     * Sync CRM to WHMCS
     */
    public function syncCRMToWHHCS(array $data): array
    {
        try {
            $email = $data['email'] ?? '';
            
            if (empty($email)) {
                return ['success' => false, 'error' => 'Email required'];
            }

            $existingClient = Capsule::table('tblclients')
                ->where('email', $email)
                ->first();

            if ($existingClient) {
                // Update existing client
                $updateData = $this->extractUpdateData($data);
                if (!empty($updateData)) {
                    Capsule::table('tblclients')
                        ->where('id', $existingClient->id)
                        ->update($updateData);
                }
                return ['success' => true, 'action' => 'updated', 'client_id' => $existingClient->id];
            } else {
                // Create new client
                $createData = $this->extractCreateData($data);
                $clientId = Capsule::table('tblclients')->insertGetId($createData);
                return ['success' => true, 'action' => 'created', 'client_id' => $clientId];
            }

        } catch (\Exception $e) {
            return ['success' => false, 'error' => $e->getMessage()];
        }
    }

    /**
     * Format client data for CRM
     */
    protected function formatClientData(object $client): array
    {
        return [
            'external_id' => $client->id,
            'email' => $client->email,
            'first_name' => $client->firstname,
            'last_name' => $client->lastname,
            'company' => $client->companyname ?? '',
            'phone' => $client->phonenumber ?? '',
            'address1' => $client->address1 ?? '',
            'city' => $client->city ?? '',
            'state' => $client->state ?? '',
            'postcode' => $client->postcode ?? '',
            'country' => $client->country ?? '',
            'status' => $client->status,
            'created_at' => $client->datecreated,
            'last_login' => $client->lastlogin,
        ];
    }

    /**
     * Extract update data from CRM
     */
    protected function extractUpdateData(array $data): array
    {
        $fields = [
            'first_name' => 'firstname',
            'last_name' => 'lastname',
            'company' => 'companyname',
            'phone' => 'phonenumber',
            'address1' => 'address1',
            'city' => 'city',
            'state' => 'state',
            'postcode' => 'postcode',
            'country' => 'country',
        ];

        $updateData = [];
        foreach ($fields as $crmField => $whmcsField) {
            if (isset($data[$crmField])) {
                $updateData[$whmcsField] = $data[$crmField];
            }
        }

        return $updateData;
    }

    /**
     * Extract create data from CRM
     */
    protected function extractCreateData(array $data): array
    {
        return [
            'firstname' => $data['first_name'] ?? '',
            'lastname' => $data['last_name'] ?? '',
            'email' => $data['email'] ?? '',
            'companyname' => $data['company'] ?? '',
            'phonenumber' => $data['phone'] ?? '',
            'address1' => $data['address1'] ?? '',
            'city' => $data['city'] ?? '',
            'state' => $data['state'] ?? '',
            'postcode' => $data['postcode'] ?? '',
            'country' => $data['country'] ?? 'US',
            'datecreated' => date('Y-m-d H:i:s'),
            'password' => md5(uniqid()),
        ];
    }

    /**
     * Send webhook to CRM
     */
    protected function sendWebhook(array $data): array
    {
        $webhookUrl = $this->config['crm_webhook_url'];
        $webhookSecret = $this->config['webhook_secret'];

        $response = wp_remote_post($webhookUrl, [
            'body' => json_encode($data),
            'headers' => [
                'Content-Type' => 'application/json',
                'X-Webhook-Secret' => $webhookSecret,
            ],
            'timeout' => 30,
        ]);

        if (is_wp_error($response)) {
            return ['success' => false, 'error' => $response->get_error_message()];
        }

        $statusCode = wp_remote_retrieve_response_code($response);

        return [
            'success' => $statusCode >= 200 && $statusCode < 300,
            'status_code' => $statusCode,
        ];
    }

    /**
     * Get sync status
     */
    public function getSyncStatus(int $clientId): array
    {
        $lastSync = Capsule::table('mod_crm_sync_log')
            ->where('client_id', $clientId)
            ->orderBy('created_at', 'desc')
            ->first();

        return [
            'synced' => $lastSync ? true : false,
            'last_sync' => $lastSync ? $lastSync->created_at : null,
            'last_status' => $lastSync ? $lastSync->status : null,
        ];
    }
}

/**
 * CRM Webhook Handler
 */
function crm_bridge_handle_webhook()
{
    if ($_SERVER['REQUEST_METHOD'] !== 'POST') {
        http_response_code(405);
        exit('Method not allowed');
    }

    $input = file_get_contents('php://input');
    $data = json_decode($input, true);

    if (!$data) {
        http_response_code(400);
        exit('Invalid JSON');
    }

    // Verify webhook secret
    $secret = $_SERVER['HTTP_X_WEBHOOK_SECRET'] ?? '';
    $config = require __DIR__ . '/config.php';

    if ($secret !== $config['webhook_secret']) {
        http_response_code(401);
        exit('Unauthorized');
    }

    $bridge = new WHMCS_CRM_Bridge();
    $result = $bridge->syncCRMToWHHCS($data);

    http_response_code($result['success'] ? 200 : 400);
    header('Content-Type: application/json');
    echo json_encode($result);
}

// Hooks
add_hook('ClientAdd', 1, function($params) {
    $bridge = new WHMCS_CRM_Bridge();
    $bridge->syncClientToCRM($params['user_id'], 'create');
});

add_hook('ClientEdit', 1, function($params) {
    $bridge = new WHMCS_CRM_Bridge();
    $bridge->syncClientToCRM($params['user_id'], 'update');
});
```

## Configuration File: config.php

```php
<?php
return [
    'crm_webhook_url' => '',
    'webhook_secret' => '',
    'auto_sync_on_create' => true,
    'auto_sync_on_update' => true,
    'sync_services' => false,
    'sync_orders' => false,
];
```

## Database Schema

```php
if (!Capsule::schema()->hasTable('mod_crm_sync_log')) {
    Capsule::schema()->create('mod_crm_sync_log', function ($table) {
        $table->increments('id');
        $table->integer('client_id')->unsigned();
        $table->enum('direction', ['inbound', 'outbound']);
        $table->string('action', 50);
        $table->string('status', 20);
        $table->longText('response')->nullable();
        $table->timestamp('created_at')->useCurrent();
        
        $table->index('client_id');
    });
}
```

## Activation & Deactivation

```php
<?php
function whmcs_crm_bridge_activate(): array
{
    try {
        if (!Capsule::schema()->hasTable('mod_crm_sync_log')) {
            Capsule::schema()->create('mod_crm_sync_log', function ($table) {
                $table->increments('id');
                $table->integer('client_id')->unsigned();
                $table->enum('direction', ['inbound', 'outbound']);
                $table->string('action', 50);
                $table->string('status', 20);
                $table->longText('response')->nullable();
                $table->timestamp('created_at')->useCurrent();
            });
        }
        return ['status' => 'success', 'description' => 'CRM Bridge activated'];
    } catch (\Exception $e) {
        return ['status' => 'error', 'description' => $e->getMessage()];
    }
}

function whmcs_crm_bridge_deactivate(): array
{
    return ['status' => 'success', 'description' => 'CRM Bridge deactivated'];
}

function whmcs_crm_bridge_config(): array
{
    return [
        'crm_webhook_url' => ['FriendlyName' => 'CRM Webhook URL', 'Type' => 'text', 'Size' => '50'],
        'webhook_secret' => ['FriendlyName' => 'Webhook Secret', 'Type' => 'password', 'Size' => '50'],
    ];
}
```

## Requirements

- WHMCS 8.0.0+
- PHP 7.4+
