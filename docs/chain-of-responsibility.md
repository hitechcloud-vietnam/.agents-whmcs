# Chain of Responsibility Pattern in WHMCS

The Chain of Responsibility Pattern passes requests along a chain of handlers. Each handler decides either to process the request or to pass it to the next handler. In WHMCS, this pattern is excellent for building request processing pipelines, validation chains, and event handling systems.

## Overview

Chain of Responsibility pattern decouples senders and receivers:
- Passes request along a chain of handlers
- Each handler decides to handle or pass on
- Allows dynamic chain configuration
- Simplifies connections between objects
- Single handler can stop processing

## Core Structure

### Base Handler

```php
<?php
// includes/ChainOfResponsibility/Handler.php

namespace CustomModule\ChainOfResponsibility;

abstract class Handler
{
    protected ?Handler $nextHandler = null;

    public function setNext(Handler $handler): Handler
    {
        $this->nextHandler = $handler;
        return $handler;
    }

    public function next($request): mixed
    {
        if ($this->nextHandler !== null) {
            return $this->nextHandler->handle($request);
        }
        return null;
    }

    abstract public function handle($request): mixed;
}
```

## Real-World WHMCS Examples

### Request Validation Chain

```php
<?php
// includes/ChainOfResponsibility/ValidationChain.php

namespace CustomModule\ChainOfResponsibility;

class ValidationChain
{
    protected ?Handler $firstHandler = null;
    protected ?Handler $lastHandler = null;

    public function add(Handler $handler): self
    {
        if ($this->firstHandler === null) {
            $this->firstHandler = $handler;
            $this->lastHandler = $handler;
        } else {
            $this->lastHandler->setNext($handler);
            $this->lastHandler = $handler;
        }

        return $this;
    }

    public function handle($request): ValidationResult
    {
        if ($this->firstHandler === null) {
            return new ValidationResult(true);
        }

        return $this->firstHandler->handle($request);
    }
}
```

```php
<?php
// includes/ChainOfResponsibility/Validators/RequiredFieldValidator.php

namespace CustomModule\ChainOfResponsibility\Validators;

use CustomModule\ChainOfResponsibility\Handler;

class RequiredFieldValidator extends Handler
{
    protected array $requiredFields;

    public function __construct(array $requiredFields)
    {
        $this->requiredFields = $requiredFields;
    }

    public function handle($request): ValidationResult
    {
        $missingFields = [];

        foreach ($this->requiredFields as $field) {
            if (!isset($request[$field]) || $request[$field] === '') {
                $missingFields[] = $field;
            }
        }

        if (!empty($missingFields)) {
            return new ValidationResult(false, 'Missing required fields: ' . implode(', ', $missingFields));
        }

        return $this->next($request);
    }
}
```

```php
<?php
// includes/ChainOfResponsibility/Validators/EmailValidator.php

namespace CustomModule\ChainOfResponsibility\Validators;

use CustomModule\ChainOfResponsibility\Handler;

class EmailValidator extends Handler
{
    public function handle($request): ValidationResult
    {
        $email = $request['email'] ?? '';

        if (empty($email)) {
            // Email is optional or validated by required field validator
            return $this->next($request);
        }

        if (!filter_var($email, FILTER_VALIDATE_EMAIL)) {
            return new ValidationResult(false, 'Invalid email format');
        }

        // Check for disposable email domains
        if ($this->isDisposableEmail($email)) {
            return new ValidationResult(false, 'Disposable email addresses are not allowed');
        }

        return $this->next($request);
    }

    protected function isDisposableEmail(string $email): bool
    {
        $domain = substr(strrchr($email, '@'), 1);
        $disposableDomains = ['tempmail.com', 'throwaway.com', 'mailinator.com'];

        return in_array(strtolower($domain), $disposableDomains);
    }
}
```

```php
<?php
// includes/ChainOfResponsibility/Validators/PhoneValidator.php

namespace CustomModule\ChainOfResponsibility\Validators;

use CustomModule\ChainOfResponsibility\Handler;

class PhoneValidator extends Handler
{
    protected array $validFormats = [
        '/^\+?1?\d{10}$/',      // US format
        '/^\+?44\d{10}$/',      // UK format
        '/^\+?61\d{9}$/'        // AU format
    ];

    public function handle($request): ValidationResult
    {
        $phone = $request['phone'] ?? '';

        if (empty($phone)) {
            return $this->next($request);
        }

        // Remove formatting characters
        $cleanPhone = preg_replace('/[^0-9+]/', '', $phone);

        $isValid = false;
        foreach ($this->validFormats as $format) {
            if (preg_match($format, $cleanPhone)) {
                $isValid = true;
                break;
            }
        }

        if (!$isValid) {
            return new ValidationResult(false, 'Invalid phone number format');
        }

        return $this->next($request);
    }
}
```

```php
<?php
// includes/ChainOfResponsibility/Validators/BusinessRuleValidator.php

namespace CustomModule\ChainOfResponsibility\Validators;

use CustomModule\ChainOfResponsibility\Handler;

class BusinessRuleValidator extends Handler
{
    protected array $rules;

    public function __construct(array $rules = [])
    {
        $this->rules = $rules;
    }

    public function addRule(callable $rule, string $errorMessage): self
    {
        $this->rules[] = ['rule' => $rule, 'message' => $errorMessage];
        return $this;
    }

    public function handle($request): ValidationResult
    {
        foreach ($this->rules as $ruleConfig) {
            $result = call_user_func($ruleConfig['rule'], $request);

            if (!$result) {
                return new ValidationResult(false, $ruleConfig['message']);
            }
        }

        return $this->next($request);
    }
}
```

```php
<?php
// includes/ChainOfResponsibility/Validators/BlacklistValidator.php

namespace CustomModule\ChainOfResponsibility\Validators;

use CustomModule\ChainOfResponsibility\Handler;
use WHMCS\Database\Capsule;

class BlacklistValidator extends Handler
{
    protected string $type;

    public function __construct(string $type = 'email')
    {
        $this->type = $type;
    }

    public function handle($request): ValidationResult
    {
        $value = $request[$this->type] ?? '';

        if (empty($value)) {
            return $this->next($request);
        }

        $isBlacklisted = Capsule::table('mod_blacklist')
            ->where('type', $this->type)
            ->where('value', $value)
            ->exists();

        if ($isBlacklisted) {
            return new ValidationResult(false, "This {$this->type} is blacklisted");
        }

        return $this->next($request);
    }
}
```

### Usage Example

```php
<?php
// Building validation chain for client registration

$validationChain = new ValidationChain();

$validationChain
    ->add(new RequiredFieldValidator(['firstname', 'lastname', 'email', 'password']))
    ->add(new EmailValidator())
    ->add(new PhoneValidator())
    ->add(new BlacklistValidator('email'))
    ->add(new BlacklistValidator('ip'))
    ->add(new BusinessRuleValidator([
        ['rule' => fn($r) => strlen($r['password'] ?? '') >= 8, 'message' => 'Password must be at least 8 characters'],
        ['rule' => fn($r) => ($r['country'] ?? '') !== 'XX', 'message' => 'Invalid country code']
    ]));

$result = $validationChain->handle([
    'firstname' => 'John',
    'lastname' => 'Doe',
    'email' => 'john@example.com',
    'phone' => '+1234567890',
    'password' => 'securepassword123',
    'country' => 'US'
]);

if (!$result->isValid()) {
    echo "Validation failed: " . $result->getError();
}
```

### Order Processing Chain

```php
<?php
// includes/ChainOfResponsibility/OrderProcessors/InventoryCheckHandler.php

namespace CustomModule\ChainOfResponsibility\OrderProcessors;

use CustomModule\ChainOfResponsibility\Handler;
use WHMCS\Database\Capsule;

class InventoryCheckHandler extends Handler
{
    public function handle($request): OrderResult
    {
        $items = $request['items'] ?? [];

        foreach ($items as $item) {
            $product = Capsule::table('tblproducts')
                ->where('id', $item['product_id'])
                ->first();

            if (!$product) {
                return new OrderResult(false, "Product not found: {$item['product_id']}");
            }

            // Check stock if managed
            if ($product->stock_control && $product->stock_level < ($item['qty'] ?? 1)) {
                return new OrderResult(false, "Insufficient stock for: {$product->name}");
            }
        }

        return $this->next($request);
    }
}
```

```php
<?php
// includes/ChainOfResponsibility/OrderProcessors/PricingHandler.php

namespace CustomModule\ChainOfResponsibility\OrderProcessors;

use CustomModule\ChainOfResponsibility\Handler;

class PricingHandler extends Handler
{
    public function handle($request): OrderResult
    {
        $items = $request['items'] ?? [];
        $calculatedTotal = 0;

        foreach ($items as $item) {
            $price = $this->getProductPrice($item['product_id'], $item['billingcycle'] ?? 'monthly');
            $qty = $item['qty'] ?? 1;
            $calculatedTotal += $price * $qty;
        }

        // Apply discounts
        $discount = $this->calculateDiscount($request['client_id'] ?? 0, $calculatedTotal);
        $calculatedTotal -= $discount;

        // Verify against submitted total
        $submittedTotal = $request['total'] ?? 0;
        $tolerance = 0.01; // 1 cent tolerance

        if (abs($calculatedTotal - $submittedTotal) > $tolerance) {
            return new OrderResult(false, "Price mismatch. Expected: {$calculatedTotal}, Received: {$submittedTotal}");
        }

        return $this->next($request);
    }

    protected function getProductPrice(int $productId, string $billingCycle): float
    {
        $price = Capsule::table('tblpricing')
            ->where('relid', $productId)
            ->where('type', 'product')
            ->first();

        return $price->{$billingCycle} ?? 0;
    }

    protected function calculateDiscount(int $clientId, float $total): float
    {
        $client = Capsule::table('tblclients')->find($clientId);

        // Loyalty discount
        $discount = 0;
        if ($client && $this->isLoyalCustomer($clientId)) {
            $discount += $total * 0.05; // 5% loyalty discount
        }

        return $discount;
    }

    protected function isLoyalCustomer(int $clientId): bool
    {
        $orderCount = Capsule::table('tblorders')
            ->where('userid', $clientId)
            ->where('status', 'Completed')
            ->count();

        return $orderCount >= 5;
    }
}
```

```php
<?php
// includes/ChainOfResponsibility/OrderProcessors/FraudCheckHandler.php

namespace CustomModule\ChainOfResponsibility\OrderProcessors;

use CustomModule\ChainOfResponsibility\Handler;

class FraudCheckHandler extends Handler
{
    public function handle($request): OrderResult
    {
        // Check for fraud indicators
        if ($this->isHighRiskOrder($request)) {
            return new OrderResult(false, 'Order flagged for manual review', [
                'requires_review' => true,
                'risk_score' => $this->calculateRiskScore($request)
            ]);
        }

        // Check against known fraud patterns
        if ($this->isFraudulentPattern($request)) {
            return new OrderResult(false, 'Order blocked due to fraud detection');
        }

        return $this->next($request);
    }

    protected function isHighRiskOrder(array $request): bool
    {
        $amount = $request['total'] ?? 0;

        // Orders over $1000 require review
        return $amount > 1000;
    }

    protected function calculateRiskScore(array $request): int
    {
        $score = 0;

        // Check IP reputation
        $ip = $request['ip_address'] ?? '';
        if ($this->isKnownProxy($ip)) {
            $score += 30;
        }

        // Check email domain
        $email = $request['email'] ?? '';
        if ($this->isFreeEmailDomain($email)) {
            $score += 10;
        }

        // Check address consistency
        if (!$this->isAddressValid($request)) {
            $score += 25;
        }

        return min(100, $score);
    }

    protected function isKnownProxy(string $ip): bool
    {
        // Check against known proxy/VPN list
        return false;
    }

    protected function isFreeEmailDomain(string $email): bool
    {
        $freeDomains = ['gmail.com', 'yahoo.com', 'hotmail.com', 'outlook.com'];
        $domain = substr(strrchr($email, '@'), 1);
        return in_array(strtolower($domain), $freeDomains);
    }

    protected function isAddressValid(array $request): bool
    {
        // Validate address against postal database
        return true;
    }

    protected function isFraudulentPattern(array $request): bool
    {
        // Check against known fraud patterns
        return false;
    }
}
```

```php
<?php
// includes/ChainOfResponsibility/OrderProcessors/PaymentAuthorizationHandler.php

namespace CustomModule\ChainOfResponsibility\OrderProcessors;

use CustomModule\ChainOfResponsibility\Handler;

class PaymentAuthorizationHandler extends Handler
{
    public function handle($request): OrderResult
    {
        $paymentMethod = $request['payment_method'] ?? '';
        $total = $request['total'] ?? 0;

        $result = match($paymentMethod) {
            'creditcard' => $this->authorizeCreditCard($request),
            'paypal' => $this->authorizePayPal($request),
            'bank_transfer' => $this->authorizeBankTransfer($request),
            'invoice' => $this->authorizeInvoice($request),
            default => new PaymentResult(false, 'Unknown payment method')
        };

        if (!$result->isSuccess()) {
            return new OrderResult(false, $result->getError());
        }

        return $this->next($request);
    }

    protected function authorizeCreditCard(array $request): PaymentResult
    {
        // Simulate credit card authorization
        return new PaymentResult(true, 'Credit card authorized');
    }

    protected function authorizePayPal(array $request): PaymentResult
    {
        // Simulate PayPal authorization
        return new PaymentResult(true, 'PayPal payment authorized');
    }

    protected function authorizeBankTransfer(array $request): PaymentResult
    {
        // Bank transfers are always valid
        return new PaymentResult(true, 'Bank transfer pending');
    }

    protected function authorizeInvoice(array $request): PaymentResult
    {
        // Check client credit limit
        return new PaymentResult(true, 'Invoice created');
    }
}
```

```php
<?php
// includes/ChainOfResponsibility/OrderProcessors/FulfillmentHandler.php

namespace CustomModule\ChainOfResponsibility\OrderProcessors;

use CustomModule\ChainOfResponsibility\Handler;

class FulfillmentHandler extends Handler
{
    public function handle($request): OrderResult
    {
        $items = $request['items'] ?? [];
        $orderId = $request['order_id'] ?? 0;

        foreach ($items as $item) {
            $this->fulfillItem($orderId, $item);
        }

        // If we got here, all checks passed
        return new OrderResult(true, 'Order processed successfully', [
            'order_id' => $orderId,
            'fulfillment_status' => 'pending'
        ]);
    }

    protected function fulfillItem(int $orderId, array $item): void
    {
        // Create service/provision product
        Capsule::table('tblhosting')->insert([
            'orderid' => $orderId,
            'userid' => $item['client_id'] ?? 0,
            'packageid' => $item['product_id'],
            'domain' => $item['domain'] ?? '',
            'regdate' => date('Y-m-d H:i:s'),
            'domainstatus' => 'Pending'
        ]);
    }
}
```

### Usage Example

```php
<?php
// Building order processing chain

$orderChain = new OrderChain();

$orderChain
    ->add(new InventoryCheckHandler())
    ->add(new PricingHandler())
    ->add(new FraudCheckHandler())
    ->add(new PaymentAuthorizationHandler())
    ->add(new FulfillmentHandler());

$result = $orderChain->handle([
    'items' => [
        ['product_id' => 1, 'qty' => 1, 'billingcycle' => 'monthly']
    ],
    'client_id' => 123,
    'total' => 9.99,
    'payment_method' => 'creditcard',
    'ip_address' => $_SERVER['REMOTE_ADDR'] ?? '',
    'email' => 'john@example.com'
]);

if ($result->isSuccess()) {
    echo "Order created: " . $result->getData()['order_id'];
} else {
    echo "Order failed: " . $result->getError();
}
```

### Webhook Processing Chain

```php
<?php
// includes/ChainOfResponsibility/WebhookHandlers/BaseWebhookHandler.php

namespace CustomModule\ChainOfResponsibility\WebhookHandlers;

use CustomModule\ChainOfResponsibility\Handler;

abstract class BaseWebhookHandler extends Handler
{
    protected string $webhookType;

    public function handle($request): WebhookResult
    {
        if ($this->canHandle($request)) {
            return $this->process($request);
        }

        return $this->next($request);
    }

    abstract protected function canHandle($request): bool;
    abstract protected function process($request): WebhookResult;
}
```

```php
<?php
// includes/ChainOfResponsibility/WebhookHandlers/SignatureVerificationHandler.php

namespace CustomModule\ChainOfResponsibility\WebhookHandlers;

use CustomModule\ChainOfResponsibility\Handler;

class SignatureVerificationHandler extends Handler
{
    protected string $secret;

    public function __construct(string $secret)
    {
        $this->secret = $secret;
    }

    public function handle($request): WebhookResult
    {
        $signature = $request['signature'] ?? '';

        if (!$this->verifySignature($request, $signature)) {
            return new WebhookResult(false, 'Invalid webhook signature');
        }

        return $this->next($request);
    }

    protected function verifySignature(array $request, string $signature): bool
    {
        $payload = $request['payload'] ?? [];
        $expectedSignature = hash_hmac('sha256', json_encode($payload), $this->secret);

        return hash_equals($expectedSignature, $signature);
    }
}
```

```php
<?php
// includes/ChainOfResponsibility/WebhookHandlers/IPWhitelistHandler.php

namespace CustomModule\ChainOfResponsibility\WebhookHandlers;

use CustomModule\ChainOfResponsibility\Handler;
use WHMCS\Database\Capsule;

class IPWhitelistHandler extends Handler
{
    public function handle($request): WebhookResult
    {
        $ip = $request['ip_address'] ?? '';

        if (empty($ip)) {
            return new WebhookResult(false, 'Missing IP address');
        }

        $isWhitelisted = Capsule::table('mod_webhook_ips')
            ->where('ip_address', $ip)
            ->where('is_active', 1)
            ->exists();

        if (!$isWhitelisted) {
            return new WebhookResult(false, 'IP not whitelisted: ' . $ip);
        }

        return $this->next($request);
    }
}
```

```php
<?php
// includes/ChainOfResponsibility/WebhookHandlers/RateLimitHandler.php

namespace CustomModule\ChainOfResponsibility\WebhookHandlers;

use CustomModule\ChainOfResponsibility\Handler;
use WHMCS\Database\Capsule;

class RateLimitHandler extends Handler
{
    protected int $maxRequests;
    protected int $windowSeconds;

    public function __construct(int $maxRequests = 100, int $windowSeconds = 60)
    {
        $this->maxRequests = $maxRequests;
        $this->windowSeconds = $windowSeconds;
    }

    public function handle($request): WebhookResult
    {
        $ip = $request['ip_address'] ?? '';
        $key = "webhook_rate:{$ip}";

        $currentCount = Capsule::table('mod_cache')
            ->where('key', $key)
            ->where('expires_at', '>', date('Y-m-d H:i:s'))
            ->first();

        if ($currentCount && $currentCount->value >= $this->maxRequests) {
            return new WebhookResult(false, 'Rate limit exceeded', [
                'retry_after' => $this->windowSeconds
            ]);
        }

        // Increment counter
        Capsule::table('mod_cache')->updateOrInsert(
            ['key' => $key],
            [
                'value' => Capsule::raw('COALESCE(value, 0) + 1'),
                'expires_at' => date('Y-m-d H:i:s', strtotime("+{$this->windowSeconds} seconds"))
            ],
            ['value' => 1]
        );

        return $this->next($request);
    }
}
```

### Chain Configuration

```php
<?php
// includes/ChainOfResponsibility/ChainBuilder.php

namespace CustomModule\ChainOfResponsibility;

class ChainBuilder
{
    protected ?Handler $firstHandler = null;
    protected ?Handler $lastHandler = null;

    public function add(Handler $handler): self
    {
        if ($this->firstHandler === null) {
            $this->firstHandler = $handler;
            $this->lastHandler = $handler;
        } else {
            $this->lastHandler->setNext($handler);
            $this->lastHandler = $handler;
        }

        return $this;
    }

    public function build(): Handler
    {
        if ($this->firstHandler === null) {
            throw new \RuntimeException('Chain must have at least one handler');
        }

        return $this->firstHandler;
    }

    public static function create(): self
    {
        return new self();
    }
}
```

## Pros

- **Decoupling**: Senders and receivers are decoupled
- **Flexibility**: Reorder handlers dynamically
- **Single Responsibility**: Each handler does one thing
- **Open/Closed**: Add new handlers without changing existing code

## Cons

- **Debugging**: Hard to trace which handler handled the request
- **Performance**: Every request traverses the chain
- **Risk**: If no handler handles, request is silently dropped

## Best Practices

1. Use a builder to construct chains
2. Implement default handling at end of chain
3. Log each handler's decision for debugging
4. Keep handlers small and focused
5. Consider using early returns in handlers