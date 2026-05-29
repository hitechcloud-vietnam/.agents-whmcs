# WHMCS API Webhook Setup Workflow

## Purpose
Guide developers through configuring and implementing webhooks in WHMCS.

## Prerequisites
- WHMCS installation
- Admin access
- Webhook endpoint URL
- Basic PHP/JSON knowledge

## Steps

### Phase 1: Webhook Overview

1. Webhook concepts
   ```
   WHMCS Webhooks:
   ├── Event-driven notifications
   ├── Real-time data transfer
   ├── Automated workflows
   └── Third-party integrations
   ```

2. Webhook events
   ```
   Available Webhook Events:
   ├── Client Events (created, updated, deleted)
   ├── Order Events (placed, paid, cancelled)
   ├── Invoice Events (created, paid, overdue)
   ├── Service Events (created, suspended, terminated)
   ├── Domain Events (registered, transferred, renewed)
   ├── Ticket Events (opened, replied, closed)
   └── Custom Events
   ```

### Phase 2: Configure Webhook in WHMCS

1. Create webhook via admin
   ```
   1. Navigate: Configuration > System Settings > Webhooks
   2. Click "Add New Webhook"
   3. Configure webhook settings:
      - Name/Description
      - Webhook URL
      - Events to subscribe
      - Authentication method
      - Retry settings
   4. Save and test
   ```

2. Webhook configuration options
   ```
   Webhook Settings:
   ├── URL: https://your-app.com/webhook
   ├── Events: Order Paid, Invoice Created, etc.
   ├── Authentication: None, Basic, Token, Signature
   ├── Retry: Number of attempts, timeout
   └── Filters: Apply conditions for triggering
   ```

### Phase 3: Webhook Endpoint Implementation

1. Basic webhook handler
   ```php
   <?php
   // Endpoint: webhook.php
   
   // Verify request
   $signature = $_SERVER['HTTP_X_WHMCS_SIGNATURE'] ?? '';
   $secret = 'your_webhook_secret';
   
   $payload = file_get_contents('php://input');
   
   if (!verifySignature($payload, $signature, $secret)) {
       http_response_code(401);
       exit('Invalid signature');
   }
   
   // Parse JSON payload
   $data = json_decode($payload, true);
   
   // Process webhook
   processWebhookEvent($data);
   
   // Return success
   http_response_code(200);
   echo json_encode(['status' => 'success']);
   
   function verifySignature($payload, $signature, $secret) {
       $expected = hash_hmac('sha256', $payload, $secret);
       return hash_equals($expected, $signature);
   }
   
   function processWebhookEvent($data) {
       $eventType = $data['event_type'] ?? '';
       
       switch ($eventType) {
           case 'OrderPaid':
               handleOrderPaid($data);
               break;
           case 'InvoiceCreated':
               handleInvoiceCreated($data);
               break;
           case 'ClientCreated':
               handleClientCreated($data);
               break;
           default:
               logWebhook($data);
       }
   }
   ```

2. Event payload structure
   ```php
   // Example: OrderPaid event
   $payload = [
       'event_type' => 'OrderPaid',
       'event_id' => 'abc123',
       'timestamp' => '2024-01-15T10:30:00Z',
       'data' => [
           'order_id' => 12345,
           'client_id' => 100,
           'invoice_id' => 67890,
           'amount' => '99.99',
           'currency' => 'USD',
       ]
   ];
   ```

### Phase 4: Webhook Authentication

1. Signature verification
   ```php
   // HMAC signature verification
   function verifyWebhookSignature($payload, $signature, $secret) {
       $expectedSignature = hash_hmac('sha256', $payload, $secret);
       return hash_equals($expectedSignature, $signature);
   }
   
   // In webhook handler
   $payload = file_get_contents('php://input');
   $signature = $_SERVER['HTTP_X_WHMCS_WEBHOOK_SIGNATURE'] ?? '';
   
   if (!verifyWebhookSignature($payload, $signature, WEBHOOK_SECRET)) {
       http_response_code(403);
       die('Forbidden');
   }
   ```

2. Basic auth for webhooks
   ```php
   // Basic authentication
   $user = $_SERVER['PHP_AUTH_USER'] ?? '';
   $pass = $_SERVER['PHP_AUTH_PW'] ?? '';
   
   if (!validateCredentials($user, $pass)) {
       header('WWW-Authenticate: Basic realm="Webhook"');
       http_response_code(401);
       exit;
   }
   
   function validateCredentials($user, $pass) {
       $validUsers = [
           'webhook_user' => password_hash('secure_password', PASSWORD_DEFAULT),
       ];
       
       return isset($validUsers[$user]) && password_verify($pass, $validUsers[$user]);
   }
   ```

3. Token authentication
   ```php
   // Bearer token authentication
   $authHeader = $_SERVER['HTTP_AUTHORIZATION'] ?? '';
   
   if (!preg_match('/^Bearer\s+(.+)$/', $authHeader, $matches)) {
       http_response_code(401);
       exit('Missing token');
   }
   
   $token = $matches[1];
   
   if (!validateToken($token)) {
       http_response_code(401);
       exit('Invalid token');
   }
   
   function validateToken($token) {
       // Validate against stored tokens
       $validTokens = ['token1_hash', 'token2_hash'];
       return in_array(hash('sha256', $token), $validTokens);
   }
   ```

### Phase 5: Webhook Event Handlers

1. Order event handlers
   ```php
   function handleOrderPaid($data) {
       $orderId = $data['data']['order_id'];
       $clientId = $data['data']['client_id'];
       $amount = $data['data']['amount'];
       
       // Process order payment
       // Send notification
       // Update external systems
       // Create provisioning task
       
       logActivity("Order #$orderId paid: $" . $amount);
   }
   
   function handleOrderCancelled($data) {
       $orderId = $data['data']['order_id'];
       
       // Handle cancellation
       // Cancel provisioning
       // Process refund if needed
       
       logActivity("Order #$orderId cancelled");
   }
   ```

2. Invoice event handlers
   ```php
   function handleInvoiceCreated($data) {
       $invoiceId = $data['data']['invoice_id'];
       $clientId = $data['data']['client_id'];
       $total = $data['data']['total'];
       
       // Create invoice record in external system
       // Send notification
       // Add to accounting system
       
       logActivity("Invoice #$invoiceId created: $" . $total);
   }
   
   function handleInvoicePaid($data) {
       $invoiceId = $data['data']['invoice_id'];
       $paymentMethod = $data['data']['payment_method'];
       
       // Record payment
       // Update accounting
       // Trigger fulfillment
       
       logActivity("Invoice #$invoiceId paid via " . $paymentMethod);
   }
   ```

3. Client event handlers
   ```php
   function handleClientCreated($data) {
       $clientId = $data['data']['client_id'];
       $email = $data['data']['email'];
       $firstName = $data['data']['first_name'];
       
       // Create user in external system
       // Send welcome email
       // Add to CRM
       
       logActivity("New client: #$clientId - $firstName ($email)");
   }
   
   function handleClientUpdated($data) {
       $clientId = $data['data']['client_id'];
       $changes = $data['data']['changes'] ?? [];
       
       // Update external system
       // Sync changes
       
       logActivity("Client #$clientId updated: " . json_encode($changes));
   }
   ```

### Phase 6: Webhook Security

1. Security best practices
   ```
   Webhook Security:
   ├── Always verify signatures
   ├── Use HTTPS for endpoints
   ├── Validate all input data
   ├── Implement rate limiting
   ├── Log all webhook activity
   └── Set up alerts for failures
   ```

2. IP whitelisting
   ```php
   // Verify webhook comes from WHMCS
   $allowedIps = [
       '127.0.0.1',  // Local for testing
       // Add WHMCS server IPs
   ];
   
   $clientIp = $_SERVER['REMOTE_ADDR'];
   
   if (!in_array($clientIp, $allowedIps)) {
       // Could be legitimate but verify
       logWebhookSecurity("Unexpected IP: $clientIp");
   }
   ```

3. Data validation
   ```php
   function validateWebhookData($data) {
       $requiredFields = ['event_type', 'event_id', 'timestamp', 'data'];
       
       foreach ($requiredFields as $field) {
           if (!isset($data[$field])) {
               throw new Exception("Missing required field: $field");
           }
       }
       
       // Validate data structure based on event type
       return true;
   }
   ```

### Phase 7: Webhook Logging and Debugging

1. Logging implementation
   ```php
   function logWebhook($data, $status = 'received') {
       $logEntry = [
           'timestamp' => date('Y-m-d H:i:s'),
           'event_type' => $data['event_type'] ?? 'unknown',
           'event_id' => $data['event_id'] ?? '',
           'status' => $status,
           'data' => $data,
       ];
       
       $logFile = '/var/log/whmcs-webhooks.log';
       file_put_contents($logFile, json_encode($logEntry) . "\n", FILE_APPEND);
   }
   ```

2. Testing webhooks
   ```
   Testing Methods:
   - Use WHMCS webhook test utility
   - Manually trigger events
   - Use ngrok for local testing
   - Check webhook logs
   ```

### Phase 8: Retry and Error Handling

1. Retry configuration
   ```
   Webhook Retry Settings:
   - Max attempts: 3-5
   - Retry delays: 1min, 5min, 30min
   - Timeout: 30 seconds
   - Dead letter queue for failures
   ```

2. Error response handling
   ```php
   // Return proper HTTP status codes
   function handleWebhookError($message, $code = 500) {
       http_response_code($code);
       echo json_encode([
           'status' => 'error',
           'message' => $message,
       ]);
   }
   
   // Success response
   function handleWebhookSuccess($data = []) {
       http_response_code(200);
       echo json_encode(array_merge([
           'status' => 'success',
           'processed' => date('c'),
       ], $data));
   }
   ```

## Related Workflows
- whmcs-api-key-generation
- whmcs-api-authentication
- whmcs-api-integration
- whmcs-api-error-handling