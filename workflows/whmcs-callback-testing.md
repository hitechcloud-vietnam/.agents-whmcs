# WHMCS Callback Testing Workflow

## Overview
This workflow provides comprehensive guidance for testing WHMCS payment gateway callbacks, ensuring reliable transaction processing.

## Prerequisites
- WHMCS installation (v8.0+)
- Payment gateway module
- Test payment credentials (sandbox accounts)
- ngrok or similar for local testing

## Step-by-Step Guide

### Step 1: Configure Payment Gateway for Testing

#### Gateway Configuration
```php
// modules/gateways/yourgateway/lib/CallbackHandler.php
<?php
namespace WHMCS\Module\Gateway\YourGateway;

class CallbackHandler
{
    private array $config;
    private string $apiKey;
    private string $webhookSecret;

    public function __construct()
    {
        $gatewayConfig = getGatewayVariables('yourgateway');
        $this->apiKey = $gatewayConfig['APIKey'] ?? '';
        $this->webhookSecret = $gatewayConfig['WebhookSecret'] ?? '';
        $this->config = [
            'testMode' => $gatewayConfig['testMode'] ?? 'on',
            'sandboxUrl' => 'https://sandbox.yourgateway.com/api',
            'productionUrl' => 'https://api.yourgateway.com/api',
        ];
    }

    public function getCallbackUrl(): string
    {
        $baseUrl = trim($this->config['testMode'] === 'on')
            ? $this->config['sandboxUrl']
            : $this->config['productionUrl'];
        return $baseUrl . '/callback';
    }

    public function processCallback(array $data): CallbackResult
    {
        // Verify callback authenticity
        if (!$this->verifyCallback($data)) {
            return new CallbackResult(false, 'Invalid signature');
        }

        // Extract transaction details
        $transactionId = $data['transaction_id'] ?? '';
        $invoiceId = $data['invoice_id'] ?? '';
        $amount = $data['amount'] ?? 0;
        $status = $data['status'] ?? '';

        // Validate transaction
        if (empty($transactionId)) {
            return new CallbackResult(false, 'Missing transaction ID');
        }

        // Process based on status
        switch ($status) {
            case 'completed':
                return $this->handleCompletedPayment($transactionId, $invoiceId, $amount);
            case 'pending':
                return $this->handlePendingPayment($transactionId, $invoiceId);
            case 'failed':
                return $this->handleFailedPayment($transactionId, $invoiceId);
            case 'refunded':
                return $this->handleRefund($transactionId, $invoiceId, $amount);
            default:
                return new CallbackResult(false, "Unknown status: $status");
        }
    }

    private function verifyCallback(array $data): bool
    {
        $signature = $data['signature'] ?? '';
        $payload = json_encode($data);
        
        $expected = hash_hmac('sha256', $payload, $this->webhookSecret);
        
        return hash_equals($expected, $signature);
    }

    private function handleCompletedPayment(string $transactionId, string $invoiceId, float $amount): CallbackResult
    {
        // Check if already processed (idempotency)
        $existing = \WHMCS\Database\Capsule::table('tblgatewaylog')
            ->where('transaction_id', $transactionId)
            ->where('gateway', 'yourgateway')
            ->first();

        if ($existing) {
            return new CallbackResult(true, 'Already processed');
        }

        // Log the transaction
        $this->logTransaction($transactionId, $invoiceId, $amount, 'completed');

        // Call WHMCS API to complete the order
        $invoiceIdNum = \WHMCS\Billing\Invoice::getIDFromKey($invoiceId);
        
        if ($invoiceIdNum) {
            addInvoicePayment(
                $invoiceIdNum,
                $transactionId,
                $amount,
                0,
                'yourgateway'
            );
        }

        return new CallbackResult(true, 'Payment processed successfully');
    }

    private function logTransaction(string $transactionId, string $invoiceId, float $amount, string $status): void
    {
        \WHMCS\Database\Capsule::table('tblgatewaylog')->insert([
            'gateway' => 'yourgateway',
            'transaction_id' => $transactionId,
            'invoice_id' => $invoiceId,
            'amount' => $amount,
            'status' => $status,
            'created_at' => date('Y-m-d H:i:s'),
        ]);
    }
}

class CallbackResult
{
    public bool $success;
    public string $message;

    public function __construct(bool $success, string $message)
    {
        $this->success = $success;
        $this->message = $message;
    }
}
```

### Step 2: Create Callback Endpoint
```php
// modules/gateways/yourgateway/callback.php
<?php
require_once __DIR__ . '/../../../init.php';
require_once __DIR__ . '/lib/CallbackHandler.php';

header('Content-Type: application/json');

try {
    // Get raw POST data
    $rawInput = file_get_contents('php://input');
    $data = json_decode($rawInput, true);

    if (json_last_error() !== JSON_ERROR_NONE) {
        // Try form data
        $data = $_POST;
    }

    // Log incoming callback for debugging
    logModuleCall(
        'yourgateway',
        'callback_received',
        json_encode($data),
        ''
    );

    $handler = new CallbackHandler();
    $result = $handler->processCallback($data);

    if ($result->success) {
        http_response_code(200);
        echo json_encode(['status' => 'success', 'message' => $result->message]);
    } else {
        http_response_code(400);
        echo json_encode(['status' => 'error', 'message' => $result->message]);
    }

} catch (Exception $e) {
    logModuleCall(
        'yourgateway',
        'callback_error',
        '',
        $e->getMessage()
    );
    
    http_response_code(500);
    echo json_encode(['status' => 'error', 'message' => 'Internal error']);
}
```

### Step 3: Write Callback Tests
```php
// tests/CallbackTest.php
<?php
namespace WHMCS\Tests;

use PHPUnit\Framework\TestCase;

class CallbackTest extends TestCase
{
    private CallbackHandler $handler;
    private string $testSecret = 'test_webhook_secret';

    protected function setUp(): void
    {
        parent::setUp();
        $this->handler = new CallbackHandler($this->testSecret);
    }

    public function testSuccessfulPaymentCallback()
    {
        $data = [
            'transaction_id' => 'txn_' . uniqid(),
            'invoice_id' => 'INV-' . rand(1000, 9999),
            'amount' => '99.99',
            'currency' => 'USD',
            'status' => 'completed',
            'customer_email' => 'test@example.com',
            'timestamp' => time(),
        ];

        $payload = json_encode($data);
        $data['signature'] = hash_hmac('sha256', $payload, $this->testSecret);

        $result = $this->handler->processCallback($data);

        $this->assertTrue($result->success);
        $this->assertEquals('Payment processed successfully', $result->message);
    }

    public function testDuplicateTransactionIgnored()
    {
        $transactionId = 'txn_duplicate_' . uniqid();
        
        $data = [
            'transaction_id' => $transactionId,
            'invoice_id' => 'INV-1001',
            'amount' => '50.00',
            'status' => 'completed',
        ];

        $payload = json_encode($data);
        $data['signature'] = hash_hmac('sha256', $payload, $this->testSecret);

        // First call
        $result1 = $this->handler->processCallback($data);
        
        // Second call (duplicate)
        $result2 = $this->handler->processCallback($data);

        $this->assertTrue($result1->success);
        $this->assertTrue($result2->success);
        $this->assertEquals('Already processed', $result2->message);
    }

    public function testInvalidSignatureRejected()
    {
        $data = [
            'transaction_id' => 'txn_test',
            'invoice_id' => 'INV-1001',
            'amount' => '50.00',
            'status' => 'completed',
            'signature' => 'invalid_signature',
        ];

        $result = $this->handler->processCallback($data);

        $this->assertFalse($result->success);
        $this->assertEquals('Invalid signature', $result->message);
    }

    public function testMissingTransactionIdRejected()
    {
        $data = [
            'invoice_id' => 'INV-1001',
            'amount' => '50.00',
            'status' => 'completed',
        ];

        $payload = json_encode($data);
        $data['signature'] = hash_hmac('sha256', $payload, $this->testSecret);

        $result = $this->handler->processCallback($data);

        $this->assertFalse($result->success);
        $this->assertStringContainsString('Missing transaction ID', $result->message);
    }

    public function testPendingPaymentHandled()
    {
        $data = [
            'transaction_id' => 'txn_pending_' . uniqid(),
            'invoice_id' => 'INV-1002',
            'amount' => '75.00',
            'status' => 'pending',
        ];

        $payload = json_encode($data);
        $data['signature'] = hash_hmac('sha256', $payload, $this->testSecret);

        $result = $this->handler->processCallback($data);

        $this->assertTrue($result->success);
    }

    public function testFailedPaymentHandled()
    {
        $data = [
            'transaction_id' => 'txn_failed_' . uniqid(),
            'invoice_id' => 'INV-1003',
            'amount' => '100.00',
            'status' => 'failed',
            'failure_reason' => 'Insufficient funds',
        ];

        $payload = json_encode($data);
        $data['signature'] = hash_hmac('sha256', $payload, $this->testSecret);

        $result = $this->handler->processCallback($data);

        $this->assertTrue($result->success);
    }

    public function testRefundHandled()
    {
        $data = [
            'transaction_id' => 'txn_refund_' . uniqid(),
            'invoice_id' => 'INV-1004',
            'amount' => '50.00',
            'status' => 'refunded',
            'original_transaction_id' => 'txn_original_123',
        ];

        $payload = json_encode($data);
        $data['signature'] = hash_hmac('sha256', $payload, $this->testSecret);

        $result = $this->handler->processCallback($data);

        $this->assertTrue($result->success);
    }

    public function testUnknownStatusRejected()
    {
        $data = [
            'transaction_id' => 'txn_unknown',
            'invoice_id' => 'INV-1005',
            'amount' => '25.00',
            'status' => 'unknown_status',
        ];

        $payload = json_encode($data);
        $data['signature'] = hash_hmac('sha256', $payload, $this->testSecret);

        $result = $this->handler->processCallback($data);

        $this->assertFalse($result->success);
        $this->assertStringContainsString('Unknown status', $result->message);
    }

    /**
     * @dataProvider amountProvider
     */
    public function testVariousAmounts(float $amount)
    {
        $data = [
            'transaction_id' => 'txn_' . uniqid(),
            'invoice_id' => 'INV-' . rand(1000, 9999),
            'amount' => (string) $amount,
            'status' => 'completed',
        ];

        $payload = json_encode($data);
        $data['signature'] = hash_hmac('sha256', $payload, $this->testSecret);

        $result = $this->handler->processCallback($data);

        $this->assertTrue($result->success);
    }

    public function amountProvider(): array
    {
        return [
            'minimum_amount' => [0.01],
            'small_amount' => [1.00],
            'medium_amount' => [99.99],
            'large_amount' => [9999.99],
            'zero_amount' => [0.00],
        ];
    }
}
```

### Step 4: Test with curl
```bash
# Test successful payment callback
curl -X POST http://localhost/modules/gateways/yourgateway/callback.php \
  -H "Content-Type: application/json" \
  -d '{
    "transaction_id": "txn_test_123",
    "invoice_id": "INV-1001",
    "amount": "99.99",
    "status": "completed",
    "customer_email": "test@example.com"
  }'

# Test with signature
SECRET="test_webhook_secret"
DATA='{"transaction_id":"txn_123","invoice_id":"INV-1001","amount":"99.99","status":"completed"}'
SIGNATURE=$(echo -n "$DATA" | openssl dgst -sha256 -hmac "$SECRET" | cut -d' ' -f2)

curl -X POST http://localhost/modules/gateways/yourgateway/callback.php \
  -H "Content-Type: application/json" \
  -d "$DATA" \
  -H "X-Signature: $SIGNATURE"
```

### Step 5: WHMCS Admin Testing

#### Test via WHMCS Admin
1. Go to WHMCS Admin > Settings > Payment Gateways
2. Enable Test Mode for your gateway
3. Go to Orders > Create New Order
4. Use test payment credentials
5. Check gateway log for callback status

#### View Gateway Logs
```php
// In WHMCS Admin, view logs
// Configuration > System Logs > Gateway Log
```

### Step 6: Run Tests
```bash
# Run callback tests
./vendor/bin/phpunit tests/CallbackTest.php

# Run with verbose output
./vendor/bin/phpunit tests/CallbackTest.php --testdox

# Generate coverage
./vendor/bin/phpunit tests/CallbackTest.php --coverage-html coverage/
```

## Callback Testing Checklist

### Security
- [ ] Signature verification implemented
- [ ] IP whitelisting configured
- [ ] HTTPS required for callbacks
- [ ] Sensitive data logged securely

### Idempotency
- [ ] Duplicate detection working
- [ ] Transaction IDs unique
- [ ] Re-processing prevented

### Error Handling
- [ ] Invalid data handled gracefully
- [ ] Timeout errors caught
- [ ] Database errors handled
- [ ] Appropriate HTTP codes returned

### Logging
- [ ] All callbacks logged
- [ ] Errors logged with details
- [ ] Performance metrics tracked

## Common Callback Issues

| Issue | Solution |
|-------|----------|
| Callback not received | Check URL, firewall, WHMCS gateway config |
| Signature mismatch | Verify secret key, encoding |
| Duplicate payments | Implement idempotency check |
| Invoice not found | Validate invoice ID format |
| Amount mismatch | Compare with invoice total |
