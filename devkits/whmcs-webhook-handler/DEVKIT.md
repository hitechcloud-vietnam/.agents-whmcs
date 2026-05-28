# WHMCS Webhook Handler Module

```php
<?php
/**
 * WHMCS Webhook Handler Module
 * 
 * Provides webhook integration handler with event routing,
 * payload validation, and retry logic.
 * 
 * @Author: HiTech Cloud DevKit
 * @Version: 1.0.0
 */

// Prevent direct access
if (!defined("WHMCS")) {
    die("Direct access prohibited");
}

function webhookhandler_MetaData()
{
    return array(
        'DisplayName' => 'Webhook Handler',
        'APIVersion' => '1.1',
        'RequiresServer' => false,
    );
}

function webhookhandler_ConfigArray()
{
    return array(
        'FriendlyName' => array(
            'Type' => 'System',
            'Value' => 'Webhook Handler',
        ),
        'EnableLogging' => array(
            'Type' => 'yesno',
            'Default' => 'yes',
            'Description' => 'Log all webhook events',
        ),
        'EnableVerification' => array(
            'Type' => 'yesno',
            'Default' => 'yes',
            'Description' => 'Verify webhook signatures',
        ),
        'MaxRetries' => array(
            'Type' => 'text',
            'Size' => '10',
            'Default' => '3',
            'Description' => 'Maximum retry attempts',
        ),
        'RetryDelay' => array(
            'Type' => 'text',
            'Size' => '10',
            'Default' => '60',
            'Description' => 'Retry delay in seconds',
        ),
        'Timeout' => array(
            'Type' => 'text',
            'Size' => '10',
            'Default' => '30',
            'Description' => 'Request timeout in seconds',
        ),
        'SecretKey' => array(
            'Type' => 'password',
            'Size' => '50',
            'Default' => '',
            'Description' => 'Webhook signing secret',
        ),
    );
}

function webhookhandler_activate()
{
    try {
        if (!function_exists('createTable')) {
            require_once dirname(__FILE__) . '/../../includes/modulefunctions.php';
        }
        
        // Webhooks table
        $webhooksTable = 'mod_webhookhandler_webhooks';
        $webhooksSchema = "
            CREATE TABLE `{$webhooksTable}` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `webhook_key` VARCHAR(100) UNIQUE NOT NULL,
                `name` VARCHAR(255) NOT NULL,
                `description` TEXT NULL,
                `event_type` VARCHAR(100) NOT NULL,
                `endpoint_url` VARCHAR(500) NOT NULL,
                `method` ENUM('GET', 'POST', 'PUT', 'PATCH', 'DELETE') DEFAULT 'POST',
                `headers` JSON NULL,
                `filters` JSON NULL,
                `transformations` JSON NULL,
                `is_active` TINYINT(1) DEFAULT 1,
                `secret` VARCHAR(255) NULL,
                `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
                `updated_at` DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
                INDEX `idx_event_type` (`event_type`),
                INDEX `idx_is_active` (`is_active`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ";
        createTable($webhooksTable, $webhooksSchema);
        
        // Deliveries table
        $deliveriesTable = 'mod_webhookhandler_deliveries';
        $deliveriesSchema = "
            CREATE TABLE `{$deliveriesTable}` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `webhook_id` INT NOT NULL,
                `event_id` VARCHAR(100) NOT NULL,
                `payload` JSON NOT NULL,
                `headers_sent` JSON NULL,
                `response_code` INT NULL,
                `response_body` TEXT NULL,
                `status` ENUM('pending', 'success', 'failed', 'retrying', 'cancelled') DEFAULT 'pending',
                `attempts` INT DEFAULT 0,
                `max_attempts` INT DEFAULT 3,
                `next_retry` DATETIME NULL,
                `error_message` TEXT NULL,
                `duration_ms` INT NULL,
                `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
                `delivered_at` DATETIME NULL,
                INDEX `idx_webhook_id` (`webhook_id`),
                INDEX `idx_status` (`status`),
                INDEX `idx_event_id` (`event_id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ";
        createTable($deliveriesTable, $deliveriesSchema);
        
        // Events table
        $eventsTable = 'mod_webhookhandler_events';
        $eventsSchema = "
            CREATE TABLE `{$eventsTable}` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `event_type` VARCHAR(100) NOT NULL,
                `event_id` VARCHAR(100) UNIQUE NOT NULL,
                `source` VARCHAR(100) NULL,
                `payload` JSON NOT NULL,
                `processed` TINYINT(1) DEFAULT 0,
                `processed_at` DATETIME NULL,
                `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
                INDEX `idx_event_type` (`event_type`),
                INDEX `idx_processed` (`processed`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ";
        createTable($eventsTable, $eventsSchema);
        
        return array(
            'status' => 'success',
            'description' => 'Webhook Handler module activated.',
        );
    } catch (\Exception $e) {
        return array(
            'status' => 'error',
            'description' => 'Failed to activate module: ' . $e->getMessage(),
        );
    }
}

function webhookhandler_deactivate()
{
    return array('status' => 'success', 'description' => 'Module deactivated.');
}

/**
 * Register a webhook
 */
function webhookhandler_RegisterWebhook($webhookData)
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) . '/../../includesWHMCS.php';
    }
    
    try {
        $webhookKey = 'wh-' . substr(md5(uniqid()), 0, 12);
        $secret = $webhookData['secret'] ?? bin2hex(random_bytes(16));
        
        $data = array(
            'webhook_key' => $webhookKey,
            'name' => $webhookData['name'],
            'description' => $webhookData['description'] ?? '',
            'event_type' => $webhookData['event_type'],
            'endpoint_url' => $webhookData['endpoint_url'],
            'method' => $webhookData['method'] ?? 'POST',
            'headers' => isset($webhookData['headers']) ? json_encode($webhookData['headers']) : null,
            'filters' => isset($webhookData['filters']) ? json_encode($webhookData['filters']) : null,
            'transformations' => isset($webhookData['transformations']) ? json_encode($webhookData['transformations']) : null,
            'secret' => $secret,
        );
        
        Capsule::table('mod_webhookhandler_webhooks')->insert($data);
        
        return array(
            'success' => true,
            'webhook_key' => $webhookKey,
            'secret' => $secret,
            'delivery_url' => webhookhandler_GetDeliveryUrl($webhookKey),
        );
    } catch (\Exception $e) {
        return array('success' => false, 'error' => $e->getMessage());
    }
}

/**
 * Get webhook delivery URL
 */
function webhookhandler_GetDeliveryUrl($webhookKey)
{
    $baseUrl = rtrim(\App::getSystemURL(), '/');
    return $baseUrl . '/includes/modules/servers/webhookhandler/deliver.php?key=' . $webhookKey;
}

/**
 * Get all webhooks
 */
function webhookhandler_GetWebhooks($filters = array())
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) . '/../../includesWHMCS.php';
    }
    
    $query = Capsule::table('mod_webhookhandler_webhooks');
    
    if (isset($filters['active_only']) && $filters['active_only']) {
        $query->where('is_active', 1);
    }
    
    if (isset($filters['event_type'])) {
        $query->where('event_type', $filters['event_type']);
    }
    
    $webhooks = $query->orderBy('name', 'asc')->get();
    
    foreach ($webhooks as &$webhook) {
        $webhook->headers = $webhook->headers ? json_decode($webhook->headers, true) : array();
        $webhook->filters = $webhook->filters ? json_decode($webhook->filters, true) : array();
        $webhook->transformations = $webhook->transformations ? json_decode($webhook->transformations, true) : array();
    }
    
    return $webhooks;
}

/**
 * Get webhook by key
 */
function webhookhandler_GetWebhook($webhookKey)
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) -> '/../../includesWHMCS.php';
    }
    
    $webhook = Capsule::table('mod_webhookhandler_webhooks')
        ->where('webhook_key', $webhookKey)
        ->first();
    
    if ($webhook) {
        $webhook->headers = $webhook->headers ? json_decode($webhook->headers, true) : array();
        $webhook->filters = $webhook->filters ? json_decode($webhook->filters, true) : array();
        $webhook->transformations = $webhook->transformations ? json_decode($webhook->transformations, true) : array();
    }
    
    return $webhook;
}

/**
 * Update webhook
 */
function webhookhandler_UpdateWebhook($webhookKey, $webhookData)
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) . '/../../includesWHMCS.php';
    }
    
    try {
        $updateData = array();
        
        if (isset($webhookData['name'])) {
            $updateData['name'] = $webhookData['name'];
        }
        if (isset($webhookData['description'])) {
            $updateData['description'] = $webhookData['description'];
        }
        if (isset($webhookData['endpoint_url'])) {
            $updateData['endpoint_url'] = $webhookData['endpoint_url'];
        }
        if (isset($webhookData['method'])) {
            $updateData['method'] = $webhookData['method'];
        }
        if (isset($webhookData['headers'])) {
            $updateData['headers'] = json_encode($webhookData['headers']);
        }
        if (isset($webhookData['filters'])) {
            $updateData['filters'] = json_encode($webhookData['filters']);
        }
        if (isset($webhookData['transformations'])) {
            $updateData['transformations'] = json_encode($webhookData['transformations']);
        }
        if (isset($webhookData['is_active'])) {
            $updateData['is_active'] = $webhookData['is_active'];
        }
        if (isset($webhookData['secret'])) {
            $updateData['secret'] = $webhookData['secret'];
        }
        
        $updateData['updated_at'] = date('Y-m-d H:i:s');
        
        Capsule::table('mod_webhookhandler_webhooks')
            ->where('webhook_key', $webhookKey)
            ->update($updateData);
        
        return array('success' => true);
    } catch (\Exception $e) {
        return array('success' => false, 'error' => $e->getMessage());
    }
}

/**
 * Delete webhook
 */
function webhookhandler_DeleteWebhook($webhookKey)
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) . '/../../includesWHMCS.php';
    }
    
    try {
        $webhook = webhookhandler_GetWebhook($webhookKey);
        
        if ($webhook) {
            // Cancel pending deliveries
            Capsule::table('mod_webhookhandler_deliveries')
                ->where('webhook_id', $webhook->id)
                ->whereIn('status', array('pending', 'retrying'))
                ->update(array('status' => 'cancelled'));
        }
        
        Capsule::table('mod_webhookhandler_webhooks')
            ->where('webhook_key', $webhookKey)
            ->delete();
        
        return array('success' => true);
    } catch (\Exception $e) {
        return array('success' => false, 'error' => $e->getMessage());
    }
}

/**
 * Trigger an event
 */
function webhookhandler_TriggerEvent($eventType, $payload, $source = 'whmcs')
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) -> '/../../includesWHMCS.php';
    }
    
    try {
        $eventId = 'evt-' . substr(md5(uniqid() . $eventType), 0, 16);
        
        // Store event
        Capsule::table('mod_webhookhandler_events')->insert(array(
            'event_type' => $eventType,
            'event_id' => $eventId,
            'source' => $source,
            'payload' => json_encode($payload),
        ));
        
        // Get matching webhooks and queue deliveries
        $webhooks = Capsule::table('mod_webhookhandler_webhooks')
            ->where('event_type', $eventType)
            ->where('is_active', 1)
            ->get();
        
        foreach ($webhooks as $webhook) {
            // Apply filters
            if (!webhookhandler_ApplyFilters($webhook, $payload)) {
                continue;
            }
            
            // Apply transformations
            $transformedPayload = webhookhandler_ApplyTransformations($webhook, $payload);
            
            // Queue delivery
            webhookhandler_QueueDelivery($webhook->id, $eventId, $transformedPayload);
        }
        
        return array('success' => true, 'event_id' => $eventId);
    } catch (\Exception $e) {
        return array('success' => false, 'error' => $e->getMessage());
    }
}

/**
 * Apply webhook filters
 */
function webhookhandler_ApplyFilters($webhook, $payload)
{
    if (empty($webhook->filters)) {
        return true;
    }
    
    foreach ($webhook->filters as $filter) {
        $field = $filter['field'];
        $operator = $filter['operator'] ?? 'equals';
        $value = $filter['value'];
        
        $fieldValue = webhookhandler_GetNestedValue($payload, $field);
        
        switch ($operator) {
            case 'equals':
                if ($fieldValue !== $value) return false;
                break;
            case 'not_equals':
                if ($fieldValue === $value) return false;
                break;
            case 'contains':
                if (strpos($fieldValue, $value) === false) return false;
                break;
            case 'starts_with':
                if (strpos($fieldValue, $value) !== 0) return false;
                break;
            case 'ends_with':
                if (strpos($fieldValue, $value) !== strlen($fieldValue) - strlen($value)) return false;
                break;
            case 'in':
                if (!in_array($fieldValue, (array)$value)) return false;
                break;
            case 'greater_than':
                if ($fieldValue <= $value) return false;
                break;
            case 'less_than':
                if ($fieldValue >= $value) return false;
                break;
        }
    }
    
    return true;
}

/**
 * Get nested value from array
 */
function webhookhandler_GetNestedValue($array, $key)
{
    $keys = explode('.', $key);
    $value = $array;
    
    foreach ($keys as $k) {
        if (!is_array($value) || !isset($value[$k])) {
            return null;
        }
        $value = $value[$k];
    }
    
    return $value;
}

/**
 * Apply webhook transformations
 */
function webhookhandler_ApplyTransformations($webhook, $payload)
{
    if (empty($webhook->transformations)) {
        return $payload;
    }
    
    $transformed = $payload;
    
    foreach ($webhook->transformations as $transform) {
        $type = $transform['type'];
        
        switch ($type) {
            case 'rename_field':
                $oldKey = $transform['from'];
                $newKey = $transform['to'];
                if (isset($transformed[$oldKey])) {
                    $transformed[$newKey] = $transformed[$oldKey];
                    unset($transformed[$oldKey]);
                }
                break;
            
            case 'add_field':
                $transformed[$transform['field']] = $transform['value'];
                break;
            
            case 'remove_field':
                unset($transformed[$transform['field']]);
                break;
            
            case 'map_values':
                $field = $transform['field'];
                $mapping = $transform['mapping'];
                if (isset($transformed[$field]) && isset($mapping[$transformed[$field]])) {
                    $transformed[$field] = $mapping[$transformed[$field]];
                }
                break;
        }
    }
    
    return $transformed;
}

/**
 * Queue webhook delivery
 */
function webhookhandler_QueueDelivery($webhookId, $eventId, $payload)
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) . '/../../includesWHMCS.php';
    }
    
    $webhook = Capsule::table('mod_webhookhandler_webhooks')
        ->where('id', $webhookId)
        ->first();
    
    Capsule::table('mod_webhookhandler_deliveries')->insert(array(
        'webhook_id' => $webhookId,
        'event_id' => $eventId,
        'payload' => json_encode($payload),
        'headers_sent' => json_encode(webhookhandler_BuildHeaders($webhook)),
    ));
}

/**
 * Build webhook headers
 */
function webhookhandler_BuildHeaders($webhook)
{
    $headers = array(
        'Content-Type' => 'application/json',
        'User-Agent' => 'WHMCSDevKit-WebhookHandler/1.0',
        'X-Webhook-Event' => $webhook->event_type,
        'X-Webhook-Key' => $webhook->webhook_key,
        'X-Delivery-ID' => '',
    );
    
    // Add custom headers
    if (!empty($webhook->headers)) {
        $headers = array_merge($headers, $webhook->headers);
    }
    
    return $headers;
}

/**
 * Sign payload
 */
function webhookhandler_SignPayload($payload, $secret)
{
    $timestamp = time();
    $signature = hash_hmac('sha256', $timestamp . '.' . json_encode($payload), $secret);
    
    return array(
        'timestamp' => $timestamp,
        'signature' => $signature,
    );
}

/**
 * Process pending deliveries
 */
function webhookhandler_ProcessDeliveries($maxDeliveries = 50)
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) . '/../../includesWHMCS.php';
    }
    
    $now = date('Y-m-d H:i:s');
    
    $deliveries = Capsule::table('mod_webhookhandler_deliveries')
        ->whereIn('status', array('pending', 'retrying'))
        ->where(function($query) use ($now) {
            $query->whereNull('next_retry')
                  ->orWhere('next_retry', '<=', $now);
        })
        ->limit($maxDeliveries)
        ->get();
    
    $results = array();
    
    foreach ($deliveries as $delivery) {
        $results[] = webhookhandler_DeliverWebhook($delivery->id);
    }
    
    return $results;
}

/**
 * Deliver a webhook
 */
function webhookhandler_DeliverWebhook($deliveryId)
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) . '/../../includesWHMCS.php';
    }
    
    $delivery = Capsule::table('mod_webhookhandler_deliveries')
        ->where('id', $deliveryId)
        ->first();
    
    if (!$delivery || !in_array($delivery->status, array('pending', 'retrying'))) {
        return array('success' => false, 'error' => 'Invalid delivery');
    }
    
    $webhook = Capsule::table('mod_webhookhandler_webhooks')
        ->where('id', $delivery->webhook_id)
        ->first();
    
    if (!$webhook || !$webhook->is_active) {
        Capsule::table('mod_webhookhandler_deliveries')
            ->where('id', $deliveryId)
            ->update(array('status' => 'cancelled'));
        
        return array('success' => false, 'error' => 'Webhook not active');
    }
    
    $startTime = microtime(true);
    $payload = json_decode($delivery->payload, true);
    $headers = json_decode($delivery->headers_sent, true);
    
    // Sign payload
    if ($webhook->secret) {
        $signature = webhookhandler_SignPayload($payload, $webhook->secret);
        $headers['X-Webhook-Signature'] = $signature['signature'];
        $headers['X-Webhook-Timestamp'] = $signature['timestamp'];
    }
    
    // Update with new headers
    Capsule::table('mod_webhookhandler_deliveries')
        ->where('id', $deliveryId)
        ->update(array(
            'headers_sent' => json_encode($headers),
            'attempts' => $delivery->attempts + 1,
        ));
    
    // Make HTTP request
    $result = webhookhandler_MakeRequest(
        $webhook->endpoint_url,
        $webhook->method,
        $payload,
        $headers
    );
    
    $duration = (microtime(true) - $startTime) * 1000;
    
    if ($result['success']) {
        Capsule::table('mod_webhookhandler_deliveries')
            ->where('id', $deliveryId)
            ->update(array(
                'status' => 'success',
                'response_code' => $result['code'],
                'response_body' => substr($result['body'], 0, 1000),
                'duration_ms' => (int) $duration,
                'delivered_at' => date('Y-m-d H:i:s'),
            ));
        
        return array('success' => true, 'code' => $result['code']);
    } else {
        $nextRetry = null;
        $status = 'failed';
        
        if ($delivery->attempts + 1 < $delivery->max_attempts) {
            $nextRetry = date('Y-m-d H:i:s', strtotime('+60 seconds'));
            $status = 'retrying';
        }
        
        Capsule::table('mod_webhookhandler_deliveries')
            ->where('id', $deliveryId)
            ->update(array(
                'status' => $status,
                'response_code' => $result['code'] ?? null,
                'response_body' => substr($result['body'] ?? '', 0, 1000),
                'error_message' => $result['error'] ?? 'Unknown error',
                'next_retry' => $nextRetry,
                'duration_ms' => (int) $duration,
            ));
        
        return array('success' => false, 'error' => $result['error']);
    }
}

/**
 * Make HTTP request
 */
function webhookhandler_MakeRequest($url, $method, $data, $headers)
{
    $ch = curl_init();
    
    curl_setopt($ch, CURLOPT_URL, $url);
    curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
    curl_setopt($ch, CURLOPT_TIMEOUT, 30);
    curl_setopt($ch, CURLOPT_SSL_VERIFYPEER, true);
    
    // Set method
    switch (strtoupper($method)) {
        case 'POST':
            curl_setopt($ch, CURLOPT_POST, true);
            curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($data));
            break;
        case 'PUT':
            curl_setopt($ch, CURLOPT_CUSTOMREQUEST, 'PUT');
            curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($data));
            break;
        case 'PATCH':
            curl_setopt($ch, CURLOPT_CUSTOMREQUEST, 'PATCH');
            curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($data));
            break;
        case 'DELETE':
            curl_setopt($ch, CURLOPT_CUSTOMREQUEST, 'DELETE');
            break;
    }
    
    // Set headers
    $headerArray = array();
    foreach ($headers as $key => $value) {
        if ($key === 'X-Delivery-ID') {
            $value = 'del-' . uniqid();
        }
        $headerArray[] = "{$key}: {$value}";
    }
    curl_setopt($ch, CURLOPT_HTTPHEADER, $headerArray);
    
    $body = curl_exec($ch);
    $code = curl_getinfo($ch, CURLINFO_HTTP_CODE);
    $error = curl_error($ch);
    
    curl_close($ch);
    
    if ($error) {
        return array('success' => false, 'error' => $error);
    }
    
    return array(
        'success' => $code >= 200 && $code < 300,
        'code' => $code,
        'body' => $body,
    );
}

/**
 * Get delivery status
 */
function webhookhandler_GetDelivery($deliveryId)
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) . '/../../includesWHMCS.php';
    }
    
    return Capsule::table('mod_webhookhandler_deliveries')
        ->where('id', $deliveryId)
        ->first();
}

/**
 * Retry failed delivery
 */
function webhookhandler_RetryDelivery($deliveryId)
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) . '/../../includesWHMCS.php';
    }
    
    Capsule::table('mod_webhookhandler_deliveries')
        ->where('id', $deliveryId)
        ->where('status', 'failed')
        ->update(array(
            'status' => 'pending',
            'next_retry' => null,
        ));
    
    return array('success' => true);
}

/**
 * Get webhook statistics
 */
function webhookhandler_GetStats($webhookKey = null)
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) -> '/../../includesWHMCS.php';
    }
    
    $query = Capsule::table('mod_webhookhandler_deliveries');
    
    if ($webhookKey) {
        $webhook = webhookhandler_GetWebhook($webhookKey);
        if ($webhook) {
            $query->where('webhook_id', $webhook->id);
        }
    }
    
    $stats = array(
        'total' => (clone $query)->count(),
        'pending' => (clone $query)->where('status', 'pending')->count(),
        'success' => (clone $query)->where('status', 'success')->count(),
        'failed' => (clone $query)->where('status', 'failed')->count(),
        'retrying' => (clone $query)->where('status', 'retrying')->count(),
    );
    
    $stats['success_rate'] = $stats['total'] > 0 
        ? round(($stats['success'] / $stats['total']) * 100, 2) 
        : 0;
    
    $avgDuration = Capsule::table('mod_webhookhandler_deliveries')
        ->selectRaw('AVG(duration_ms) as avg');
    
    if ($webhookKey && isset($webhook)) {
        $avgDuration->where('webhook_id', $webhook->id);
    }
    
    $stats['avg_duration_ms'] = (int) ($avgDuration->first()->avg ?? 0);
    
    return $stats;
}
