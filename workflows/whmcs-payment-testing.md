# WHMCS Payment Testing Workflow

## Description
Test payment gateways in sandbox/test environment before production.

## Prerequisites
- Payment gateway test/sandbox account
- Test credit cards
- WHMCS installation

## Steps

### Step 1: Configure Test Environment
```php
<?php
// In gateway configuration
function test_gateway_config()
{
    return [
        'FriendlyName' => ['value' => 'Test Gateway'],
        'environment' => [
            'Type' => 'dropdown',
            'Options' => 'sandbox,live',
            'Default' => 'sandbox',
        ],
        'sandboxApiKey' => ['Type' => 'password', 'Label' => 'Sandbox API Key'],
        'liveApiKey' => ['Type' => 'password', 'Label' => 'Production API Key'],
    ];
}
```

### Step 2: Test Cards

**Stripe Test Cards:**
```markdown
| Card Number | Scenario |
|-------------|----------|
| 4242424242424242 | Success |
| 4000000000000002 | Always decline |
| 4000000000009995 | Insufficient funds |
| 4000002500003155 | Test 3D Secure |
| 4000000000003220 | 3D Secure 2 - success |
| 4000000000003228 | 3D Secure 2 - failure |
```

**PayPal Test Accounts:**
```markdown
- Business account: business@test.com
- Personal account: buyer@test.com
Access: https://developer.paypal.com
```

**General Test Cards:**
```markdown
Visa: 4111111111111111
Mastercard: 5555555555554444
American Express: 378282246310005
Discover: 6011111111111117

Expiry: Any future date
CVC: Any 3 digits (4 for Amex)
```

### Step 3: Create Test Invoice
```bash
# Create test invoice for payment testing
# Use admin area or API
```

### Step 4: Test Scenarios

**Test Case 1: Successful Payment**
```
1. Create invoice for $10.00
2. Select test gateway
3. Enter card 4242424242424242
4. Submit payment
5. Verify invoice marked as Paid
6. Check transaction logged
Expected: Payment succeeds, invoice updated
```

**Test Case 2: Failed Payment**
```
1. Create invoice for $10.00
2. Select test gateway
3. Enter card 4000000000000002
4. Submit payment
5. Verify error message shown
Expected: Error displayed, invoice unchanged
```

**Test Case 3: 3D Secure Flow**
```
1. Enable 3D Secure in gateway
2. Create invoice for $50.00
3. Enter card 4000002500003155
4. Complete 3DS verification
5. Verify payment succeeds
Expected: Authentication flow, then payment
```

**Test Case 4: Refund Processing**
```
1. Find paid invoice from test
2. Process refund via admin
3. Verify amount deducted
4. Check refund in gateway dashboard
Expected: Refund processed successfully
```

### Step 5: Automated Testing
```php
<?php
/**
 * Payment Gateway Test Suite
 */

class PaymentGatewayTest
{
    public function runAllTests()
    {
        $results = [];
        
        $results['test_success'] = $this->testSuccessfulPayment();
        $results['test_decline'] = $this->testDeclinedCard();
        $results['test_insufficient'] = $this->testInsufficientFunds();
        $results['test_refund'] = $this->testRefund();
        $results['test_webhook'] = $this->testWebhook();
        
        return $results;
    }
    
    private function testSuccessfulPayment()
    {
        $params = [
            'amount' => 10.00,
            'currency' => 'USD',
            'card' => [
                'number' => '4242424242424242',
                'exp_month' => 12,
                'exp_year' => 2025,
                'cvc' => 123,
            ],
        ];
        
        $result = $this->processPayment($params);
        
        return [
            'passed' => $result['success'],
            'message' => $result['success'] ? 'Payment succeeded' : $result['error'],
        ];
    }
    
    private function testDeclinedCard()
    {
        $params = [
            'amount' => 10.00,
            'card' => ['number' => '4000000000000002'],
        ];
        
        $result = $this->processPayment($params);
        
        return [
            'passed' => !$result['success'] && $result['error_code'] === 'card_declined',
            'message' => $result['error'] ?? 'Wrong error',
        ];
    }
    
    private function processPayment($params)
    {
        // Implement payment processing
    }
}
```

### Step 6: Production Checklist
```markdown
Before Going Live:
[ ] All test transactions passed
[ ] Sandbox credentials replaced with production
[ ] Webhook URLs updated to production
[ ] Error handling tested
[ ] Refund flow tested
[ ] Email notifications tested
[ ] Logging enabled
[ ] Monitoring configured
[ ] Support contacts configured with gateway
```

## Test Account Setup
```bash
# Stripe: https://dashboard.stripe.com/test/apikeys
# PayPal: https://developer.paypal.com/demo
# Braintree: https://sandbox.braintreegateway.com
# Authorize.Net: https://sandbox.authorize.net
```

## Tags
- testing
- payment
- sandbox
- quality-assurance