# WHMCS API Proxy Module

## Overview
API proxy/gateway module for third-party integrations.

## Module File: api_proxy.php

```php
<?php
/**
 * WHMCS API Proxy Module
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

use WHMCS\Database\Capsule;

class WHMCS_API_Proxy
{
    protected $config;

    public function __construct()
    {
        $this->config = require __DIR__ . '/config.php';
    }

    /**
     * Handle API request
     */
    public function handleRequest(array $params): array
    {
        $endpoint = $params['endpoint'] ?? '';
        $method = $params['method'] ?? 'GET';
        $data = $params['data'] ?? [];

        // Log request
        $logId = $this->logRequest($endpoint, $method, $data);

        // Validate endpoint
        if (!$this->isAllowedEndpoint($endpoint)) {
            return ['success' => false, 'error' => 'Endpoint not allowed'];
        }

        // Make request
        $result = $this->makeRequest($endpoint, $method, $data);

        // Update log
        $this->logResponse($logId, $result);

        return $result;
    }

    /**
     * Check if endpoint is allowed
     */
    protected function isAllowedEndpoint(string $endpoint): bool
    {
        $allowed = $this->config['allowed_endpoints'] ?? [];
        return empty($allowed) || in_array($endpoint, $allowed);
    }

    /**
     * Make external API request
     */
    protected function makeRequest(string $endpoint, string $method, array $data): array
    {
        $baseUrl = $this->config['base_url'] ?? '';
        $apiKey = $this->config['api_key'] ?? '';

        $url = rtrim($baseUrl, '/') . '/' . ltrim($endpoint, '/');

        $args = [
            'headers' => [
                'Authorization' => 'Bearer ' . $apiKey,
                'Content-Type' => 'application/json',
            ],
            'timeout' => 30,
        ];

        if ($method === 'POST') {
            $args['body'] = json_encode($data);
            $response = wp_remote_post($url, $args);
        } else {
            $response = wp_remote_get($url, $args);
        }

        if (is_wp_error($response)) {
            return ['success' => false, 'error' => $response->get_error_message()];
        }

        $body = json_decode(wp_remote_retrieve_body($response), true);
        $statusCode = wp_remote_retrieve_response_code($response);

        return [
            'success' => $statusCode >= 200 && $statusCode < 300,
            'status_code' => $statusCode,
            'data' => $body,
        ];
    }

    /**
     * Log API request
     */
    protected function logRequest(string $endpoint, string $method, array $data): int
    {
        return Capsule::table('mod_api_proxy_logs')->insertGetId([
            'endpoint' => $endpoint,
            'method' => $method,
            'request_data' => json_encode($data),
            'ip_address' => $_SERVER['REMOTE_ADDR'] ?? 'unknown',
            'created_at' => date('Y-m-d H:i:s'),
        ]);
    }

    /**
     * Log API response
     */
    protected function logResponse(int $logId, array $result): void
    {
        Capsule::table('mod_api_proxy_logs')
            ->where('id', $logId)
            ->update([
                'response_data' => json_encode($result),
                'completed_at' => date('Y-m-d H:i:s'),
            ]);
    }
}

function whmcs_api_proxy_activate(): array
{
    try {
        if (!Capsule::schema()->hasTable('mod_api_proxy_logs')) {
            Capsule::schema()->create('mod_api_proxy_logs', function ($table) {
                $table->increments('id');
                $table->string('endpoint', 255);
                $table->string('method', 10);
                $table->longText('request_data')->nullable();
                $table->longText('response_data')->nullable();
                $table->string('ip_address', 45);
                $table->timestamp('created_at')->useCurrent();
                $table->timestamp('completed_at')->nullable();
            });
        }
        return ['status' => 'success', 'description' => 'API Proxy activated'];
    } catch (\Exception $e) {
        return ['status' => 'error', 'description' => $e->getMessage()];
    }
}

function whmcs_api_proxy_deactivate(): array
{
    return ['status' => 'success', 'description' => 'API Proxy deactivated'];
}

function whmcs_api_proxy_config(): array
{
    return [
        'base_url' => ['FriendlyName' => 'Base URL', 'Type' => 'text', 'Size' => '50'],
        'api_key' => ['FriendlyName' => 'API Key', 'Type' => 'password', 'Size' => '50'],
        'allowed_endpoints' => ['FriendlyName' => 'Allowed Endpoints', 'Type' => 'textarea', 'Description' => 'One per line, leave empty for all'],
    ];
}
```

## Configuration File: config.php

```php
<?php
return [
    'base_url' => '',
    'api_key' => '',
    'allowed_endpoints' => [],
];
```

## Requirements

- WHMCS 8.0.0+
- PHP 7.4+
