# WHMCS Webhook Events Reference

**Version:** 8.x
**Updated:** 2026-05-28
**Related Skills:** `api-endpoints-reference`, `hooks-reference`

## Overview

Webhooks allow external systems to receive real-time notifications when events occur in WHMCS. This reference documents all available webhook events, their payloads, and usage patterns.

## Webhook Configuration

### Setting Up Webhooks

```php
// Configuration > System Settings > Webhooks
// Or via API

$webhook = [
    'name' => 'Order Notifications',
    'url' => 'https://your-app.com/webhooks/whmcs',
    'secret' => 'your-webhook-secret',
    'events' => [
        'OrderCreated',
        'OrderPaid',
        'OrderCancelled',
    ],
    'enabled' => true,
];
```

### Webhook Security

```php
// Verify webhook signature
function verifyWebhookSignature(
    string $payload,
    string $signature,
    string $secret
): bool {
    $expected = hash_hmac('sha256', $payload, $secret);
    return hash_equals($expected, $signature);
}

// Usage in webhook endpoint
$payload = file_get_contents('php://input');
$signature = $_SERVER['HTTP_X_WHMCS_SIGNATURE'] ?? '';

if (!verifyWebhookSignature($payload, $signature, $webhookSecret)) {
    http_response_code(401);
    exit('Invalid signature');
}

$data = json_decode($payload, true);
```

## Client Events

### ClientCreated

**Triggered:** When a new client account is created.
**Payload:**
```json
{
    "event": "ClientCreated",
    "timestamp": "2026-05-28T12:00:00+00:00",
    "data": {
        "client_id": 123,
        "first_name": "John",
        "last_name": "Doe",
        "email": "john@example.com",
        "company": "Acme Corp",
        "country": "US",
        "date_created": "2026-05-28",
        "status": "Active"
    }
}
```

**Example Handler:**
```php
add_hook('AfterClientCreate', 1, function($vars) {
    $clientId = $vars['client_id'];

    // Create account in external system
    // Send welcome email via custom service
    // Initialize CRM record
});
```

### ClientUpdated

**Triggered:** When client information is updated.
**Payload:**
```json
{
    "event": "ClientUpdated",
    "timestamp": "2026-05-28T12:00:00+00:00",
    "data": {
        "client_id": 123,
        "changes": {
            "first_name": {
                "old": "John",
                "new": "Jonathan"
            }
        },
        "updated_at": "2026-05-28T12:00:00+00:00"
    }
}
```

### ClientDeleted

**Triggered:** When a client account is deleted.
**Payload:**
```json
{
    "event": "ClientDeleted",
    "timestamp": "2026-05-28T12:00:00+00:00",
    "data": {
        "client_id": 123,
        "email": "john@example.com",
        "reason": "Closed by admin"
    }
}
```

## Order Events

### OrderCreated

**Triggered:** When a new order is placed.
**Payload:**
```json
{
    "event": "OrderCreated",
    "timestamp": "2026-05-28T12:00:00+00:00",
    "data": {
        "order_id": 456,
        "order_num": "ORD-456",
        "client_id": 123,
        "status": "Pending",
        "items": [
            {
                "type": "product",
                "id": 1,
                "name": "Basic Hosting",
                "amount": 9.99,
                "qty": 1
            }
        ],
        "total": 9.99,
        "payment_method": "paypal",
        "created_at": "2026-05-28T12:00:00+00:00"
    }
}
```

### OrderPaid

**Triggered:** When an order is marked as paid.
**Payload:**
```json
{
    "event": "OrderPaid",
    "timestamp": "2026-05-28T12:00:00+00:00",
    "data": {
        "order_id": 456,
        "order_num": "ORD-456",
        "client_id": 123,
        "status": "Active",
        "invoice_id": 789,
        "invoice_num": "INV-789",
        "payment_method": "paypal",
        "transaction_id": "TXN-123456",
        "amount_paid": 9.99,
        "paid_at": "2026-05-28T12:00:00+00:00"
    }
}
```

### OrderCancelled

**Triggered:** When an order is cancelled.
**Payload:**
```json
{
    "event": "OrderCancelled",
    "timestamp": "2026-05-28T12:00:00+00:00",
    "data": {
        "order_id": 456,
        "order_num": "ORD-456",
        "client_id": 123,
        "reason": "Customer requested",
        "refund_status": "pending",
        "cancelled_at": "2026-05-28T12:00:00+00:00"
    }
}
```

### OrderFulfilled

**Triggered:** When all items in an order have been provisioned.
**Payload:**
```json
{
    "event": "OrderFulfilled",
    "timestamp": "2026-05-28T12:00:00+00:00",
    "data": {
        "order_id": 456,
        "order_num": "ORD-456",
        "client_id": 123,
        "services": [
            {
                "service_id": 101,
                "type": "hosting",
                "domain": "example.com",
                "username": "johndoe1",
                "status": "Active"
            }
        ],
        "fulfilled_at": "2026-05-28T12:00:00+00:00"
    }
}
```

## Invoice Events

### InvoiceCreated

**Triggered:** When a new invoice is created.
**Payload:**
```json
{
    "event": "InvoiceCreated",
    "timestamp": "2026-05-28T12:00:00+00:00",
    "data": {
        "invoice_id": 789,
        "invoice_num": "INV-789",
        "client_id": 123,
        "status": "Unpaid",
        "subtotal": 100.00,
        "tax": 20.00,
        "total": 120.00,
        "balance": 120.00,
        "due_date": "2026-06-28",
        "created_at": "2026-05-28T12:00:00+00:00"
    }
}
```

### InvoicePaid

**Triggered:** When an invoice is paid.
**Payload:**
```json
{
    "event": "InvoicePaid",
    "timestamp": "2026-05-28T12:00:00+00:00",
    "data": {
        "invoice_id": 789,
        "invoice_num": "INV-789",
        "client_id": 123,
        "status": "Paid",
        "amount_paid": 120.00,
        "payment_method": "paypal",
        "transaction_id": "TXN-123456",
        "paid_at": "2026-05-28T12:00:00+00:00"
    }
}
```

### InvoiceVoided

**Triggered:** When an invoice is voided/cancelled.
**Payload:**
```json
{
    "event": "InvoiceVoided",
    "timestamp": "2026-05-28T12:00:00+00:00",
    "data": {
        "invoice_id": 789,
        "invoice_num": "INV-789",
        "client_id": 123,
        "status": "Cancelled",
        "reason": "Duplicate invoice",
        "voided_at": "2026-05-28T12:00:00+00:00"
    }
}
```

### InvoicePaymentFailed

**Triggered:** When a payment attempt fails.
**Payload:**
```json
{
    "event": "InvoicePaymentFailed",
    "timestamp": "2026-05-28T12:00:00+00:00",
    "data": {
        "invoice_id": 789,
        "invoice_num": "INV-789",
        "client_id": 123,
        "payment_method": "stripe",
        "failure_reason": "Card declined",
        "attempted_at": "2026-05-28T12:00:00+00:00"
    }
}
```

## Service Events

### ServiceCreated

**Triggered:** When a new service is provisioned.
**Payload:**
```json
{
    "event": "ServiceCreated",
    "timestamp": "2026-05-28T12:00:00+00:00",
    "data": {
        "service_id": 101,
        "client_id": 123,
        "order_id": 456,
        "product_id": 1,
        "product_name": "Basic Hosting",
        "domain": "example.com",
        "status": "Pending",
        "first_payment": 9.99,
        "recurring_amount": 9.99,
        "billing_cycle": "Monthly",
        "next_due_date": "2026-06-28",
        "created_at": "2026-05-28T12:00:00+00:00"
    }
}
```

### ServiceActivated

**Triggered:** When a service is activated.
**Payload:**
```json
{
    "event": "ServiceActivated",
    "timestamp": "2026-05-28T12:00:00+00:00",
    "data": {
        "service_id": 101,
        "client_id": 123,
        "domain": "example.com",
        "status": "Active",
        "username": "johndoe1",
        "activated_at": "2026-05-28T12:00:00+00:00"
    }
}
```

### ServiceSuspended

**Triggered:** When a service is suspended.
**Payload:**
```json
{
    "event": "ServiceSuspended",
    "timestamp": "2026-05-28T12:00:00+00:00",
    "data": {
        "service_id": 101,
        "client_id": 123,
        "domain": "example.com",
        "status": "Suspended",
        "reason": "Overdue",
        "suspended_at": "2026-05-28T12:00:00+00:00"
    }
}
```

### ServiceUnsuspended

**Triggered:** When a suspended service is reactivated.
**Payload:**
```json
{
    "event": "ServiceUnsuspended",
    "timestamp": "2026-05-28T12:00:00+00:00",
    "data": {
        "service_id": 101,
        "client_id": 123,
        "domain": "example.com",
        "status": "Active",
        "unsuspended_at": "2026-05-28T12:00:00+00:00"
    }
}
```

### ServiceTerminated

**Triggered:** When a service is terminated.
**Payload:**
```json
{
    "event": "ServiceTerminated",
    "timestamp": "2026-05-28T12:00:00+00:00",
    "data": {
        "service_id": 101,
        "client_id": 123,
        "domain": "example.com",
        "status": "Terminated",
        "reason": "Cancelled",
        "terminated_at": "2026-05-28T12:00:00+00:00"
    }
}
```

### ServiceRenewed

**Triggered:** When a service is renewed.
**Payload:**
```json
{
    "event": "ServiceRenewed",
    "timestamp": "2026-05-28T12:00:00+00:00",
    "data": {
        "service_id": 101,
        "client_id": 123,
        "domain": "example.com",
        "status": "Active",
        "new_due_date": "2026-07-28",
        "amount_paid": 9.99,
        "renewed_at": "2026-05-28T12:00:00+00:00"
    }
}
```

## Domain Events

### DomainRegistered

**Triggered:** When a domain is registered.
**Payload:**
```json
{
    "event": "DomainRegistered",
    "timestamp": "2026-05-28T12:00:00+00:00",
    "data": {
        "domain_id": 201,
        "client_id": 123,
        "domain": "example.com",
        "registrar": "enom",
        "registration_period": 1,
        "registration_date": "2026-05-28",
        "expiry_date": "2027-05-28",
        "dns_management": true,
        "email_forwarding": false,
        "id_protection": true,
        "registered_at": "2026-05-28T12:00:00+00:00"
    }
}
```

### DomainTransferred

**Triggered:** When a domain transfer completes.
**Payload:**
```json
{
    "event": "DomainTransferred",
    "timestamp": "2026-05-28T12:00:00+00:00",
    "data": {
        "domain_id": 201,
        "client_id": 123,
        "domain": "example.com",
        "registrar": "enom",
        "status": "Active",
        "expiry_date": "2027-05-28",
        "transferred_at": "2026-05-28T12:00:00+00:00"
    }
}
```

### DomainRenewed

**Triggered:** When a domain is renewed.
**Payload:**
```json
{
    "event": "DomainRenewed",
    "timestamp": "2026-05-28T12:00:00+00:00",
    "data": {
        "domain_id": 201,
        "client_id": 123,
        "domain": "example.com",
        "new_expiry_date": "2028-05-28",
        "renewal_period": 1,
        "amount_paid": 14.99,
        "renewed_at": "2026-05-28T12:00:00+00:00"
    }
}
```

### DomainExpired

**Triggered:** When a domain expires.
**Payload:**
```json
{
    "event": "DomainExpired",
    "timestamp": "2026-05-28T12:00:00+00:00",
    "data": {
        "domain_id": 201,
        "client_id": 123,
        "domain": "example.com",
        "status": "Expired",
        "expiry_date": "2026-05-28",
        "expired_at": "2026-05-28T12:00:00+00:00"
    }
}
```

### DomainTransferredIn

**Triggered:** When a domain transfer request is initiated.
**Payload:**
```json
{
    "event": "DomainTransferredIn",
    "timestamp": "2026-05-28T12:00:00+00:00",
    "data": {
        "domain_id": 201,
        "client_id": 123,
        "domain": "example.com",
        "status": "Pending Transfer",
        "transfer_code": "AUTH_CODE_HERE",
        "requested_at": "2026-05-28T12:00:00+00:00"
    }
}
```

## Support Ticket Events

### TicketOpened

**Triggered:** When a new support ticket is created.
**Payload:**
```json
{
    "event": "TicketOpened",
    "timestamp": "2026-05-28T12:00:00+00:00",
    "data": {
        "ticket_id": 301,
        "ticket_number": "TKT-301",
        "client_id": 123,
        "subject": "Technical Support Request",
        "status": "Open",
        "priority": "Medium",
        "department": "Technical Support",
        "created_at": "2026-05-28T12:00:00+00:00"
    }
}
```

### TicketReply

**Triggered:** When a reply is added to a ticket.
**Payload:**
```json
{
    "event": "TicketReply",
    "timestamp": "2026-05-28T12:00:00+00:00",
    "data": {
        "ticket_id": 301,
        "ticket_number": "TKT-301",
        "client_id": 123,
        "message_id": 401,
        "type": "reply",
        "author_type": "client",
        "status": "Open",
        "created_at": "2026-05-28T12:00:00+00:00"
    }
}
```

### TicketClosed

**Triggered:** When a ticket is closed.
**Payload:**
```json
{
    "event": "TicketClosed",
    "timestamp": "2026-05-28T12:00:00+00:00",
    "data": {
        "ticket_id": 301,
        "ticket_number": "TKT-301",
        "client_id": 123,
        "status": "Closed",
        "closed_by": "admin",
        "closed_at": "2026-05-28T12:00:00+00:00"
    }
}
```

## Affiliate Events

### AffiliateComissionPaid

**Triggered:** When affiliate commission is paid.
**Payload:**
```json
{
    "event": "AffiliateComissionPaid",
    "timestamp": "2026-05-28T12:00:00+00:00",
    "data": {
        "affiliate_id": 501,
        "client_id": 123,
        "referral_id": 789,
        "order_id": 456,
        "commission_amount": 1.00,
        "payment_method": "credit",
        "paid_at": "2026-05-28T12:00:00+00:00"
    }
}
```

## Custom Webhook Implementation

### Registering Custom Webhooks

```php
// includes/hooks/custom_webhooks.php

/**
 * Register custom webhook events
 */
add_hook('RegisterWebhooks', 1, function($vars) {
    return [
        [
            'name' => 'custom_event',
            'description' => 'Triggers on custom event',
            'source' => 'module_name',
        ],
    ];
});

/**
 * Dispatch custom webhook
 */
function dispatchCustomWebhook(string $event, array $data): void
{
    $webhooks = Capsule::table('tblwebhooks')
        ->where('event', $event)
        ->where('enabled', 1)
        ->get();

    foreach ($webhooks as $webhook) {
        $payload = json_encode([
            'event' => $event,
            'timestamp' => date('c'),
            'data' => $data,
        ]);

        $signature = hash_hmac('sha256', $payload, $webhook->secret);

        $ch = curl_init($webhook->url);
        curl_setopt_array($ch, [
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => $payload,
            CURLOPT_HTTPHEADER => [
                'Content-Type: application/json',
                'X-WHMCS-Signature: ' . $signature,
            ],
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_TIMEOUT => 30,
        ]);

        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);

        // Log webhook delivery
        Capsule::table('tblwebhook_log')->insert([
            'webhook_id' => $webhook->id,
            'event' => $event,
            'url' => $webhook->url,
            'payload' => $payload,
            'response_code' => $httpCode,
            'response' => $response,
            'created_at' => date('Y-m-d H:i:s'),
        ]);
    }
}
```

## Webhook Best Practices

1. **Verify Signatures**: Always validate HMAC signatures
2. **Respond Quickly**: Return 200 immediately, process async
3. **Handle Retries**: WHMCS retries failed webhooks
4. **Idempotency**: Design handlers to handle duplicate events
5. **Use HTTPS**: Always use secure webhook endpoints
6. **Log Everything**: Maintain audit trail of all deliveries
7. **Monitor Failures**: Track and alert on webhook failures

## Related Documentation

- [API Endpoints Reference](api-endpoints-reference.md)
- [Hooks Reference](hooks-reference.md)
- [Email Template Variables](email-template-variables.md)
