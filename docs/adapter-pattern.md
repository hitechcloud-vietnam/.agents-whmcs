# Adapter Pattern in WHMCS

The Adapter Pattern allows incompatible interfaces to work together. In WHMCS, this is particularly useful when integrating with third-party APIs, payment gateways, or external services that have different interface requirements than your module expects.

## Overview

An adapter wraps an existing class with a new interface, translating calls from your code into calls the third-party service understands. This allows you to:
- Integrate external APIs without modifying their code
- Create consistent interfaces across multiple providers
- Swap providers without changing consuming code

## Core Structure

### Target Interface

```php
<?php
// includes/Adapters/NotificationChannelInterface.php

namespace CustomModule\Adapters;

interface NotificationChannelInterface
{
    public function send(NotificationMessage $message): NotificationResult;
    public function supportsBatch(): bool;
    public function getMaxRecipients(): int;
}
```

### Adaptee (Third-Party Service)

```php
<?php
// includes/Adapters/External/SMSGlobalClient.php
// This is the external library with incompatible interface

namespace CustomModule\Adapters\External;

class SMSGlobalClient
{
    protected string $apiKey;
    protected string $apiSecret;

    public function __construct(string $apiKey, string $apiSecret)
    {
        $this->apiKey = $apiKey;
        $this->apiSecret = $apiSecret;
    }

    // Incompatible interface - different method name and parameters
    public function dispatchSMS(string $recipient, string $text, array $options = []): array
    {
        // Implementation...
        return [
            'messageId' => uniqid('sms_'),
            'status' => 'queued',
            'timestamp' => date('c')
        ];
    }

    public function batchDispatch(array $messages): array
    {
        // Implementation...
        return array_map(fn($m) => $this->dispatchSMS($m['to'], $m['text']), $messages);
    }
}
```

### Adapter Implementation

```php
<?php
// includes/Adapters/SMSGlobalAdapter.php

namespace CustomModule\Adapters;

use CustomModule\Adapters\External\SMSGlobalClient;
use CustomModule\ValueObjects\NotificationMessage;
use CustomModule\ValueObjects\NotificationResult;

class SMSGlobalAdapter implements NotificationChannelInterface
{
    protected SMSGlobalClient $client;

    public function __construct(SMSGlobalClient $client)
    {
        $this->client = $client;
    }

    public function send(NotificationMessage $message): NotificationResult
    {
        try {
            $result = $this->client->dispatchSMS(
                $message->getRecipient(),
                $message->getContent(),
                [
                    'from' => $message->getSender(),
                    'schedule' => $message->getScheduleTime()
                ]
            );

            return new NotificationResult(
                success: true,
                externalId: $result['messageId'],
                status: $result['status'],
                timestamp: $result['timestamp']
            );
        } catch (\Exception $e) {
            return new NotificationResult(
                success: false,
                error: $e->getMessage()
            );
        }
    }

    public function supportsBatch(): bool
    {
        return true;
    }

    public function getMaxRecipients(): int
    {
        return 100;
    }
}
```

## Real-World WHMCS Examples

### Payment Gateway Adapter

```php
<?php
// includes/Adapters/Gateways/PaymentGatewayInterface.php

namespace CustomModule\Adapters\Gateways;

interface PaymentGatewayInterface
{
    public function charge(ChargeRequest $request): ChargeResponse;
    public function refund(RefundRequest $request): RefundResponse;
    public function getWebhookHandler(): WebhookHandlerInterface;
}
```

```php
<?php
// includes/Adapters/Gateways/StripeAdapter.php

namespace CustomModule\Adapters\Gateways;

use Stripe\StripeClient;

class StripeAdapter implements PaymentGatewayInterface
{
    protected StripeClient $stripe;

    public function __construct(string $apiKey)
    {
        $this->stripe = new StripeClient($apiKey);
    }

    public function charge(ChargeRequest $request): ChargeResponse
    {
        try {
            $paymentIntent = $this->stripe->paymentIntents->create([
                'amount' => $request->getAmountCents(),
                'currency' => strtolower($request->getCurrency()),
                'customer' => $request->getCustomerId(),
                'metadata' => [
                    'invoice_id' => $request->getReference(),
                    'order_id' => $request->getOrderId()
                ]
            ]);

            return new ChargeResponse(
                success: true,
                transactionId: $paymentIntent->id,
                status: $this->mapStatus($paymentIntent->status),
                clientSecret: $paymentIntent->client_secret
            );
        } catch (\Stripe\Exception\CardException $e) {
            return new ChargeResponse(
                success: false,
                error: $e->getMessage(),
                errorCode: $e->getStripeCode()
            );
        }
    }

    public function refund(RefundRequest $request): RefundResponse
    {
        $refund = $this->stripe->refunds->create([
            'payment_intent' => $request->getTransactionId(),
            'amount' => $request->getAmountCents()
        ]);

        return new RefundResponse(
            success: true,
            refundId: $refund->id,
            status: $refund->status
        );
    }

    public function getWebhookHandler(): WebhookHandlerInterface
    {
        return new StripeWebhookHandler($this->stripe);
    }

    protected function mapStatus(string $stripeStatus): string
    {
        return match($stripeStatus) {
            'succeeded' => 'completed',
            'requires_payment_method' => 'pending',
            'canceled' => 'failed',
            default => 'pending'
        };
    }
}
```

### Registrar Adapter for Multiple DNS Providers

```php
<?php
// includes/Adapters/Registrar/DomainRegistrarInterface.php

namespace CustomModule\Adapters\Registrar;

interface DomainRegistrarInterface
{
    public function registerDomain(RegisterDomainRequest $request): RegisterDomainResponse;
    public function transferDomain(TransferDomainRequest $request): TransferDomainResponse;
    public function renewDomain(RenewDomainRequest $request): RenewDomainResponse;
    public function getNameservers(string $domain): NameserverResponse;
    public function setNameservers(string $domain, array $nameservers): void;
    public function getDomainInfo(string $domain): DomainInfoResponse;
}
```

```php
<?php
// includes/Adapters/Registrar/CloudflareRegistrarAdapter.php

namespace CustomModule\Adapters\Registrar;

use Cloudflare\API\Adapter\GuzzleAdapter;
use Cloudflare\API\Authenticator;

class CloudflareRegistrarAdapter implements DomainRegistrarInterface
{
    protected GuzzleAdapter $adapter;

    public function __construct(string $email, string $apiKey)
    {
        $auth = new Authenticator($email, $apiKey);
        $this->adapter = new GuzzleAdapter($auth);
    }

    public function registerDomain(RegisterDomainRequest $request): RegisterDomainResponse
    {
        // Cloudflare Registrar API implementation
        $response = $this->adapter->post('registrar/domains', [
            'name' => $request->getDomain(),
            'years' => $request->getRegistrationYears()
        ]);

        return new RegisterDomainResponse(
            success: true,
            domainId: $response['id'],
            expirationDate: $response['expires_at']
        );
    }

    public function transferDomain(TransferDomainRequest $request): TransferDomainResponse
    {
        $response = $this->adapter->post('registrar/transfer', [
            'domain' => $request->getDomain(),
            'auth_code' => $request->getAuthCode()
        ]);

        return new TransferDomainResponse(
            success: true,
            transferId: $response['id'],
            status: $response['status']
        );
    }

    public function renewDomain(RenewDomainRequest $request): RenewDomainResponse
    {
        $response = $this->adapter->put("registrar/domains/{$request->getDomain()}", [
            'years' => $request->getRenewalYears()
        ]);

        return new RenewDomainResponse(
            success: true,
            newExpiration: $response['expires_at']
        );
    }

    public function getNameservers(string $domain): NameserverResponse
    {
        $zones = $this->adapter->get("zones?domain={$domain}");
        $zoneId = $zones['result'][0]['id'] ?? null;

        if (!$zoneId) {
            return new NameserverResponse(nameservers: []);
        }

        $dns = $this->adapter->get("zones/{$zoneId}/dns_records?type=NS");
        $nameservers = array_map(fn($r) => $r['name'], $dns['result']);

        return new NameserverResponse(nameservers: $nameservers);
    }

    public function setNameservers(string $domain, array $nameservers): void
    {
        // Implementation...
    }

    public function getDomainInfo(string $domain): DomainInfoResponse
    {
        // Implementation...
        return new DomainInfoResponse(/*...*/);
    }
}
```

### Email Provider Adapter

```php
<?php
// includes/Adapters/Email/EmailProviderInterface.php

namespace CustomModule\Adapters\Email;

interface EmailProviderInterface
{
    public function send(SendEmailRequest $request): SendEmailResponse;
    public function sendTemplate(SendTemplateRequest $request): SendEmailResponse;
    public function getTrackingStats(string $emailId): EmailTrackingStats;
}
```

```php
<?php
// includes/Adapters/Email/SendGridAdapter.php

namespace CustomModule\Adapters\Email;

use SendGrid;
use SendGrid\Mail\Mail;

class SendGridAdapter implements EmailProviderInterface
{
    protected SendGrid $sendgrid;

    public function __construct(string $apiKey)
    {
        $this->sendgrid = new SendGrid($apiKey);
    }

    public function send(SendEmailRequest $request): SendEmailResponse
    {
        $email = new Mail();
        $email->setFrom($request->getFromEmail(), $request->getFromName());
        $email->addTo($request->getToEmail(), $request->getToName());
        $email->setSubject($request->getSubject());
        $email->addContent('text/html', $request->getHtmlBody());

        try {
            $response = $this->sendgrid->send($email);

            return new SendEmailResponse(
                success: true,
                messageId: $response->headers()->get('X-Message-Id'),
                status: $this->parseStatusCode($response->statusCode())
            );
        } catch (\Exception $e) {
            return new SendEmailResponse(success: false, error: $e->getMessage());
        }
    }

    public function sendTemplate(SendTemplateRequest $request): SendEmailResponse
    {
        $email = new Mail();
        $email->setFrom($request->getFromEmail());
        $email->addTo($request->getToEmail());
        $email->setTemplateId($request->getTemplateId());
        $email->addDynamicTemplateData($request->getTemplateData());

        try {
            $response = $this->sendgrid->send($email);
            return new SendEmailResponse(
                success: true,
                messageId: $response->headers()->get('X-Message-Id')
            );
        } catch (\Exception $e) {
            return new SendEmailResponse(success: false, error: $e->getMessage());
        }
    }

    public function getTrackingStats(string $emailId): EmailTrackingStats
    {
        // Implementation with tracking API
        return new EmailTrackingStats(
            delivered: true,
            opens: 5,
            clicks: 2,
            bounces: 0
        );
    }

    protected function parseStatusCode(int $code): string
    {
        return match(true) {
            $code >= 200 && $code < 300 => 'delivered',
            $code >= 400 && $code < 500 => 'bounced',
            $code >= 500 => 'failed',
            default => 'unknown'
        };
    }
}
```

### Factory for Creating Adapters

```php
<?php
// includes/Adapters/AdapterFactory.php

namespace CustomModule\Adapters;

class AdapterFactory
{
    protected static array $adapters = [];

    public static function register(string $name, string $adapterClass): void
    {
        self::$adapters[$name] = $adapterClass;
    }

    public static function make(string $name, array $config = []): NotificationChannelInterface
    {
        if (!isset(self::$adapters[$name])) {
            throw new \InvalidArgumentException("Adapter '{$name}' not registered");
        }

        $adapterClass = self::$adapters[$name];

        // Handle constructor dependencies
        $adapter = new $adapterClass(...$config);

        return $adapter;
    }

    public static function has(string $name): bool
    {
        return isset(self::$adapters[$name]);
    }
}

// Usage
AdapterFactory::register('smsglobal', SMSGlobalAdapter::class);
AdapterFactory::register('twilio', TwilioAdapter::class);
AdapterFactory::register('sendgrid', SendGridAdapter::class);
```

## Pros

- **Decoupling**: Consumer code doesn't depend on external service interfaces
- **Testability**: Easy to mock adapters for unit testing
- **Flexibility**: Swap providers without changing consuming code
- **Single Responsibility**: Adapters handle only translation logic
- **Consistency**: Create uniform interfaces across different providers

## Cons

- **Boilerplate**: Requires writing adapter classes for each service
- **Complexity**: Additional layer can be confusing for simple integrations
- **Maintenance**: Must keep adapters in sync with external API changes
- **Performance**: Extra method call overhead for translation

## Best Practices

1. Define clear target interfaces before implementing adapters
2. Keep adapters thin - only handle translation, not business logic
3. Use factory pattern to manage adapter creation
4. Create integration tests that verify adapter behavior
5. Document the mapping between your interface and external API
6. Use composition over inheritance for better flexibility