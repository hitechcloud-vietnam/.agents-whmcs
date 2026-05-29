# WHMCS Webhook Setup Workflow

## Overview
This workflow covers setting up and handling webhooks for real-time event notifications.

## Step 1: Webhook Handler

```php
<?php
// src/Service/WebhookService.php

namespace WHMCS\Module\Addon\YourModule\Service;

use WHMCS\Database\Capsule;

class WebhookService
{
    private $secretKey;
    private $supportedEvents = [
        'client.created',
        'client.updated',
        'client.deleted',
        'invoice.created',
        'invoice.paid',
        'invoice.cancelled',
        'order.created',
        'order.completed',
        'service.created',
        'service.suspended',
        'service.unsuspended',
        'service.terminated',
        'domain.registered',
        'domain.transferred',
        'domain.renewed'
    ];

    public function __construct(string $secretKey)
    {
        $this->secretKey = $secretKey;
    }

    public function handleWebhook(string $payload, array $headers): array
    {
        // Verify signature
        if (!$this->verifySignature($payload, $headers)) {
            return ['success' => false, 'error' => 'Invalid signature'];
        }

        $data = json_decode($payload, true);

        if (!$data || !isset($data['event'])) {
            return ['success' => false, 'error' => 'Invalid payload'];
        }

        $event = $data['event'];
        $eventData = $data['data'] ?? [];

        // Log webhook
        $this->logWebhook($event, $eventData);

        // Process event
        try {
            $result = $this->processEvent($event, $eventData);
            return ['success' => true, 'processed' => $result];
        } catch (\Exception $e) {
            $this->logError($event, $e->getMessage());
            return ['success' => false, 'error' => $e->getMessage()];
        }
    }

    private function verifySignature(string $payload, array $headers): bool
    {
        $signature = $headers['X-Webhook-Signature'] ?? '';

        if (!$signature) {
            return false;
        }

        $expected = hash_hmac('sha256', $payload, $this->secretKey);
        return hash_equals($expected, $signature);
    }

    private function processEvent(string $event, array $data): bool
    {
        return match ($event) {
            'client.created' => $this->handleClientCreated($data),
            'client.updated' => $this->handleClientUpdated($data),
            'invoice.paid' => $this->handleInvoicePaid($data),
            'order.completed' => $this->handleOrderCompleted($data),
            'service.suspended' => $this->handleServiceSuspended($data),
            'service.terminated' => $this->handleServiceTerminated($data),
            default => $this->handleGenericEvent($event, $data)
        };
    }

    private function handleClientCreated(array $data): bool
    {
        logActivity("Webhook: New client created - ID: " . ($data['id'] ?? 'unknown'));

        // Sync to external CRM
        $this->syncToExternal($data, 'create_client');

        return true;
    }

    private function handleClientUpdated(array $data): bool
    {
        logActivity("Webhook: Client updated - ID: " . ($data['id'] ?? 'unknown'));

        $this->syncToExternal($data, 'update_client');

        return true;
    }

    private function handleInvoicePaid(array $data): bool
    {
        $invoiceId = $data['id'] ?? null;

        if ($invoiceId) {
            $invoice = Capsule::table('tblinvoices')->where('id', $invoiceId)->first();
            $client = Capsule::table('tblclients')->where('id', $invoice->userid)->first();

            // Send to accounting system
            $this->sendToAccounting($invoice, $client);

            // Update CRM
            $this->updateCrmPaymentStatus($client->email, $invoice->total);
        }

        return true;
    }

    private function handleOrderCompleted(array $data): bool
    {
        $orderId = $data['id'] ?? null;

        if ($orderId) {
            // Trigger fulfillment
            $this->triggerFulfillment($orderId);

            // Send welcome email
            $this->sendWelcomeEmail($orderId);
        }

        return true;
    }

    private function handleServiceSuspended(array $data): bool
    {
        $serviceId = $data['id'] ?? null;

        if ($serviceId) {
            $service = Capsule::table('tblhosting')->where('id', $serviceId)->first();

            // Notify external systems
            $this->notifyExternalServiceStatus($service, 'suspended');
        }

        return true;
    }

    private function handleServiceTerminated(array $data): bool
    {
        $serviceId = $data['id'] ?? null;

        if ($serviceId) {
            // Clean up external resources
            $this->cleanupExternalResources($serviceId);

            // Archive data
            $this->archiveServiceData($serviceId);
        }

        return true;
    }

    private function handleGenericEvent(string $event, array $data): bool
    {
        logActivity("Webhook: Unhandled event - $event");
        return true;
    }

    private function syncToExternal(array $data, string $action): void
    {
        // Implementation for external sync
        $externalApi = new ExternalApiService(
            Capsule::config('external_api_url'),
            Capsule::config('external_api_key')
        );

        try {
            $externalApi->post("/webhooks/$action", $data);
        } catch (\Exception $e) {
            logActivity("External sync failed: " . $e->getMessage());
        }
    }

    private function sendToAccounting($invoice, $client): void
    {
        // Send invoice data to accounting system
    }

    private function updateCrmPaymentStatus(string $email, float $amount): void
    {
        // Update CRM with payment information
    }

    private function triggerFulfillment(int $orderId): void
    {
        // Trigger fulfillment processes
    }

    private function sendWelcomeEmail(int $orderId): void
    {
        // Send welcome email for new order
    }

    private function notifyExternalServiceStatus($service, string $status): void
    {
        // Notify external systems of status change
    }

    private function cleanupExternalResources(int $serviceId): void
    {
        // Clean up resources on external systems
    }

    private function archiveServiceData(int $serviceId): void
    {
        // Archive service data before deletion
    }

    private function logWebhook(string $event, array $data): void
    {
        Capsule::table('mod_webhook_logs')->insert([
            'event' => $event,
            'data' => json_encode($data),
            'received_at' => date('Y-m-d H:i:s'),
            'status' => 'received'
        ]);
    }

    private function logError(string $event, string $error): void
    {
        Capsule::table('mod_webhook_logs')->insert([
            'event' => $event,
            'data' => json_encode(['error' => $error]),
            'received_at' => date('Y-m-d H:i:s'),
            'status' => 'error'
        ]);
    }

    public function getSupportedEvents(): array
    {
        return $this->supportedEvents;
    }

    public function registerWebhook(string $webhookUrl): array
    {
        // Register webhook with external service
        return ['success' => true, 'webhook_id' => uniqid('wh_')];
    }
}
```

## Step 2: Webhook Endpoint

```php
<?php
// public/webhook.php

require_once __DIR__ . '/init.php';

use WHMCS\Module\Addon\YourModule\Service\WebhookService;

// Get webhook secret from config
$secretKey = Capsule::config('webhook_secret');

// Initialize webhook service
$webhookService = new WebhookService($secretKey);

// Get raw payload
$payload = file_get_contents('php://input');

// Get headers
$headers = [
    'X-Webhook-Signature' => $_SERVER['HTTP_X_WEBHOOK_SIGNATURE'] ?? '',
    'X-Webhook-Timestamp' => $_SERVER['HTTP_X_WEBHOOK_TIMESTAMP'] ?? '',
    'X-Webhook-Event' => $_SERVER['HTTP_X_WEBHOOK_EVENT'] ?? ''
];

// Handle webhook
$result = $webhookService->handleWebhook($payload, $headers);

// Return response
http_response_code($result['success'] ? 200 : 400);
header('Content-Type: application/json');
echo json_encode($result);
```

## Step 3: Webhook Configuration

```php
<?php
// Webhook configuration in module

function your_module_config()
{
    return [
        'name' => 'Your Module',
        'description' => 'Module with webhook support',
        'fields' => [
            'webhook_secret' => [
                'Type' => 'password',
                'Description' => 'Secret key for webhook verification'
            ],
            'webhook_url' => [
                'Type' => 'text',
                'Description' => 'Webhook endpoint URL'
            ],
            'enabled_events' => [
                'Type' => 'dropdown',
                'Options' => 'client.created,client.updated,invoice.paid,order.completed',
                'Description' => 'Events to receive'
            ]
        ]
    ];
}
```

## Verification Checklist

- [ ] Webhook handler implemented
- [ ] Signature verification working
- [ ] Event processing implemented
- [ ] Webhook logging configured
- [ ] Test webhook delivered successfully
- [ ] Error handling verified
