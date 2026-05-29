# WHMCS Webhooks Integration

Complete guide for implementing webhook handlers in WHMCS.

## Overview

Webhooks allow real-time communication between WHMCS and external systems.

## Webhook Configuration

### Admin Configuration

```php
/**
 * Webhooks are configured in WHMCS Admin:
 * Setup > Automation > Webhooks
 */
```

### Webhook Endpoint Structure

```php
<?php
/**
 * modules/hooks/webhook_handler.php
 * 
 * Webhook handler endpoint
 */
require_once __DIR__ . '/../../init.php';

// Get webhook payload
$rawPayload = file_get_contents('php://input');
$payload = json_decode($rawPayload, true);

// Verify webhook signature
$signature = $_SERVER['HTTP_X_WHMCS_SIGNATURE'] ?? '';
$secret = \App::getApplication()->getConfig()->webhook_secret;

if (!verify_signature($payload, $signature, $secret)) {
    http_response_code(401);
    exit('Unauthorized');
}

// Process webhook
$event = $payload['event'] ?? 'unknown';

switch ($event) {
    case 'InvoiceCreated':
        handleInvoiceCreated($payload);
        break;
    case 'InvoicePaid':
        handleInvoicePaid($payload);
        break;
    case 'ServiceCreated':
        handleServiceCreated($payload);
        break;
    case 'ServiceSuspended':
        handleServiceSuspended($payload);
        break;
    case 'ServiceTerminated':
        handleServiceTerminated($payload);
        break;
    case 'TicketCreated':
        handleTicketCreated($payload);
        break;
    default:
        logActivity("Webhook: Unhandled event - {$event}");
}

http_response_code(200);
echo json_encode(['status' => 'received']);

/**
 * Verify webhook signature
 */
function verify_signature(array $payload, string $signature, string $secret): bool
{
    $payloadString = json_encode($payload);
    $expected = base64_encode(hash_hmac('sha256', $payloadString, $secret, true));
    
    return hash_equals($expected, $signature);
}
```

## Invoice Webhooks

### Invoice Created

```php
<?php
/**
 * Handle invoice created event
 */
function handleInvoiceCreated(array $payload): void
{
    $invoiceId = $payload['invoice_id'];
    $clientId = $payload['user_id'];
    $total = $payload['total'];
    
    logActivity("Webhook: Invoice #{$invoiceId} created for client #{$clientId}");
    
    // Send notification to external system
    $externalApi = new ExternalBillingSystem();
    $externalApi->notifyInvoiceCreated($invoiceId, [
        'client_id' => $clientId,
        'total' => $total,
        'created_at' => $payload['created_at'],
    ]);
}

/**
 * Handle invoice paid event
 */
function handleInvoicePaid(array $payload): void
{
    $invoiceId = $payload['invoice_id'];
    $transactionId = $payload['transaction_id'];
    $amount = $payload['amount'];
    
    logActivity("Webhook: Invoice #{$invoiceId} paid - Transaction: {$transactionId}");
    
    // Update external system
    $externalApi = new ExternalBillingSystem();
    $externalApi->recordPayment($invoiceId, [
        'transaction_id' => $transactionId,
        'amount' => $amount,
        'paid_at' => $payload['paid_at'],
    ]);
    
    // Trigger service provisioning if pending
    if ($payload['status'] === 'paid') {
        $pendingServices = Capsule::table('tblhosting')
            ->where('invoice_id', $invoiceId)
            ->where('domainstatus', 'Pending')
            ->get();
        
        foreach ($pendingServices as $service) {
            run_task('ProcessOrder', ['serviceid' => $service->id]);
        }
    }
}
```

## Service Webhooks

### Service Created

```php
<?php
/**
 * Handle service created event
 */
function handleServiceCreated(array $payload): void
{
    $serviceId = $payload['service_id'];
    $clientId = $payload['user_id'];
    $productId = $payload['product_id'];
    $domain = $payload['domain'];
    
    logActivity("Webhook: Service #{$serviceId} created - Domain: {$domain}");
    
    // Sync with external provisioning system
    $externalApi = new ExternalProvisioningSystem();
    
    $externalApi->registerService([
        'local_service_id' => $serviceId,
        'client_id' => $clientId,
        'product_id' => $productId,
        'domain' => $domain,
        'username' => $payload['username'],
        'created_at' => $payload['created_at'],
    ]);
}

/**
 * Handle service suspended event
 */
function handleServiceSuspended(array $payload): void
{
    $serviceId = $payload['service_id'];
    $reason = $payload['suspend_reason'] ?? 'Payment';
    
    logActivity("Webhook: Service #{$serviceId} suspended - Reason: {$reason}");
    
    // Update external system
    $externalApi = new ExternalProvisioningSystem();
    $externalApi->suspendService($serviceId, [
        'reason' => $reason,
        'suspended_at' => $payload['suspended_at'],
    ]);
}

/**
 * Handle service terminated event
 */
function handleServiceTerminated(array $payload): void
{
    $serviceId = $payload['service_id'];
    
    logActivity("Webhook: Service #{$serviceId} terminated");
    
    // Update external system and cleanup
    $externalApi = new ExternalProvisioningSystem();
    $externalApi->terminateService($serviceId);
}
```

## Ticket Webhooks

```php
<?php
/**
 * Handle ticket created event
 */
function handleTicketCreated(array $payload): void
{
    $ticketId = $payload['ticket_id'];
    $clientId = $payload['user_id'];
    $subject = $payload['subject'];
    $priority = $payload['priority'];
    
    logActivity("Webhook: Ticket #{$ticketId} created - Subject: {$subject}");
    
    // Forward to external support system
    $supportApi = new ExternalSupportSystem();
    $supportApi->createTicket([
        'external_id' => $ticketId,
        'client_id' => $clientId,
        'subject' => $subject,
        'priority' => $priority,
        'created_at' => $payload['created_at'],
    ]);
}

/**
 * Handle ticket reply event
 */
function handleTicketReply(array $payload): void
{
    $ticketId = $payload['ticket_id'];
    $replyId = $payload['reply_id'];
    
    $supportApi = new ExternalSupportSystem();
    $supportApi->addReply($ticketId, [
        'reply_id' => $replyId,
        'message' => $payload['message'],
        'is_admin' => $payload['is_admin_reply'],
    ]);
}
```

## Domain Webhooks

```php
<?php
/**
 * Handle domain registered event
 */
function handleDomainRegistered(array $payload): void
{
    $domainId = $payload['domain_id'];
    $domain = $payload['domain'];
    $registrar = $payload['registrar'];
    $expiryDate = $payload['expiry_date'];
    
    logActivity("Webhook: Domain {$domain} registered via {$registrar}");
    
    // Sync with DNS management system
    $dnsApi = new ExternalDNSService();
    $dnsApi->registerDomain([
        'domain_id' => $domainId,
        'domain' => $domain,
        'nameservers' => $payload['nameservers'],
    ]);
}

/**
 * Handle domain renewed event
 */
function handleDomainRenewed(array $payload): void
{
    $domainId = $payload['domain_id'];
    $domain = $payload['domain'];
    $newExpiry = $payload['expiry_date'];
    
    logActivity("Webhook: Domain {$domain} renewed until {$newExpiry}");
    
    // Update external DNS records
    $dnsApi = new ExternalDNSService();
    $dnsApi->updateExpiry($domainId, $newExpiry);
}

/**
 * Handle domain transferred event
 */
function handleDomainTransferred(array $payload): void
{
    $domainId = $payload['domain_id'];
    $domain = $payload['domain'];
    $newRegistrar = $payload['new_registrar'];
    
    logActivity("Webhook: Domain {$domain} transferred to {$newRegistrar}");
    
    // Update DNS and SSL certificates
    $sslApi = new ExternalSSLService();
    $sslApi->reissueCertificate($domain);
}
```

## Outgoing Webhooks

### Module Webhook Trigger

```php
<?php
/**
 * Trigger outgoing webhook from module
 */
function triggerWebhook(string $event, array $data): bool
{
    $webhookUrl = Capsule::table('tblconfiguration')
        ->where('setting', 'ExternalWebhookUrl')
        ->first();
    
    if (!$webhookUrl || empty($webhookUrl->value)) {
        return false;
    }
    
    $payload = [
        'event' => $event,
        'timestamp' => date('c'),
        'data' => $data,
    ];
    
    $ch = curl_init($webhookUrl->value);
    curl_setopt_array($ch, [
        CURLOPT_POST => true,
        CURLOPT_POSTFIELDS => json_encode($payload),
        CURLOPT_RETURNTRANSFER => true,
        CURLOPT_HTTPHEADER => [
            'Content-Type: application/json',
            'X-WHMCS-Signature: ' . generateSignature($payload),
        ],
        CURLOPT_TIMEOUT => 10,
    ]);
    
    $response = curl_exec($ch);
    $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
    curl_close($ch);
    
    return $httpCode >= 200 && $httpCode < 300;
}

/**
 * Generate webhook signature
 */
function generateSignature(array $payload): string
{
    $secret = Capsule::table('tblconfiguration')
        ->where('setting', 'ExternalWebhookSecret')
        ->first();
    
    return base64_encode(
        hash_hmac('sha256', json_encode($payload), $secret->value ?? '', true)
    );
}
```

## Retry Mechanism

```php
<?php
/**
 * Retry failed webhook deliveries
 */
function retryFailedWebhooks(): void
{
    $failed = Capsule::table('mod_webhook_deliveries')
        ->where('status', 'failed')
        ->where('attempts', '<', 5)
        ->where('next_retry', '<', date('Y-m-d H:i:s'))
        ->get();
    
    foreach ($failed as $delivery) {
        $payload = json_decode($delivery->payload, true);
        $success = deliverWebhook($delivery->url, $payload);
        
        if ($success) {
            Capsule::table('mod_webhook_deliveries')
                ->where('id', $delivery->id)
                ->update([
                    'status' => 'delivered',
                    'delivered_at' => date('Y-m-d H:i:s'),
                ]);
        } else {
            Capsule::table('mod_webhook_deliveries')
                ->where('id', $delivery->id)
                ->update([
                    'attempts' => $delivery->attempts + 1,
                    'last_attempt' => date('Y-m-d H:i:s'),
                    'next_retry' => calculateNextRetry($delivery->attempts + 1),
                ]);
        }
    }
}

/**
 * Calculate next retry time with exponential backoff
 */
function calculateNextRetry(int $attempt): string
{
    $delayMinutes = pow(2, $attempt); // 2, 4, 8, 16, 32 minutes
    return date('Y-m-d H:i:s', strtotime("+{$delayMinutes} minutes"));
}
```

## Best Practices

1. **Verify signatures** - Always validate webhook authenticity
2. **Respond quickly** - Return 200 immediately, process async
3. **Implement retries** - Queue failed deliveries for retry
4. **Log everything** - Keep audit trail of all webhooks
5. **Use HTTPS** - Only accept secure webhook connections
6. **Handle idempotently** - Same event may arrive multiple times

## Related Documentation

- [whmcs-integration-api.md](whmcs-integration-api.md)
- [whmcs-integration-webhooks.md](whmcs-integration-webhooks.md)
