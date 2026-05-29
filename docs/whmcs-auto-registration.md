# WHMCS Auto Registration Documentation

## Overview

Auto registration in WHMCS enables automatic domain registration when customers complete checkout. This feature eliminates manual intervention and ensures seamless domain provisioning.

## Configuration

### Prerequisites

- WHMCS version 8.0 or higher
- Registrar module with auto-registration support
- Valid API credentials configured
- Sufficient account balance with registrar (if applicable)

### Enabling Auto Registration

1. Navigate to **Setup > Products/Services > Domain Pricing**
2. Select the TLD (Top-Level Domain) to configure
3. Enable "Auto Registration" checkbox
4. Configure registration period (1-10 years)
5. Save changes

### Registrar Module Requirements

Your registrar module must implement:

```php
/**
 * Check if registrar supports auto-registration
 * @return bool
 */
public function supportsAutoRegistration()
{
    return true;
}
```

## Workflow

### Registration Process Flow

1. Customer selects domain during checkout
2. WHMCS validates domain availability via registrar API
3. If available, registration order is queued
4. Background cron job processes registration
5. Registrar API receives registration request
6. Domain status updated to "Active"
7. Customer notified via email

### Cron Job Configuration

Configure auto-registration cron in **Configuration > System Settings > Automation Settings**:

| Setting | Value | Description |
|---------|-------|-------------|
| Auto Domain Registration Frequency | Every 5 minutes | How often to check queue |
| Maximum Registrations per Run | 50 | Rate limiting |
| Retry Failed Registrations | Yes | Auto-retry on failure |
| Retry Interval | 1 hour | Time between retries |

## API Integration

### Manual Registration via API

```php
use WHMCS\Session;
use WHMCS\Domain\Registrar\Factory;

// Register domain manually
$registrar = Factory::getRegistrar('enom');
$result = $registrar->register([
    'domain' => 'example.com',
    'registration_period' => 2,
    'registrant' => [
        'firstname' => 'John',
        'lastname' => 'Doe',
        'email' => 'john@example.com',
        'address1' => '123 Main St',
        'city' => 'Anytown',
        'state' => 'CA',
        'country' => 'US',
        'postcode' => '12345',
        'phone' => '+1.5551234567'
    ]
]);

if ($result['success']) {
    // Domain registered successfully
    $domainId = $result['domain_id'];
} else {
    // Handle error
    $error = $result['error'];
}
```

### API Response Codes

| Code | Description | Action |
|------|-------------|--------|
| 100 | Registration successful | Update domain status |
| 200 | Domain unavailable | Notify customer |
| 300 | Invalid registrant data | Log and retry |
| 400 | Authentication failed | Alert admin |
| 500 | Registrar error | Retry with backoff |

## Error Handling

### Common Errors

| Error Code | Message | Solution |
|------------|---------|----------|
| EPP-001 | Authentication failed | Check registrar credentials |
| EPP-002 | Domain already exists | Verify availability before registration |
| EPP-003 | Invalid contact info | Validate registrant data |
| EPP-004 | Nameserver invalid | Use valid nameservers |
| EPP-005 | Premium domain pricing | Handle premium fee notification |

### Retry Logic

WHMCS implements exponential backoff for failed registrations:

1. First retry: 1 hour
2. Second retry: 4 hours
3. Third retry: 12 hours
4. Final retry: 24 hours
5. Mark as failed after 5 attempts

## Best Practices

### Security

- Store registrar credentials securely using encrypted storage
- Use API keys instead of passwords when available
- Implement IP whitelisting for registrar API access
- Log all registration attempts for audit

### Performance

- Queue registrations during peak hours
- Implement batch registration for multiple domains
- Use async processing for non-urgent registrations
- Monitor API rate limits

### Customer Experience

- Provide real-time registration status updates
- Send confirmation emails upon successful registration
- Display estimated completion time at checkout
- Offer alternative suggestions if registration fails

## Troubleshooting

### Debug Mode

Enable debug logging in `configuration.php`:

```php
$debug_mode = true;
$log_provider = 'file';
```

View logs at: `/whmcs/logs/registration.log`

### Verification Commands

```bash
# Check pending registrations
whmcscli domain pending-list

# Force registration retry
whmcscli domain retry-registration --domain=example.com

# Verify registrar connection
whmcscli registrar test-connection --module=enom
```

## Webhooks

Configure webhooks for registration events:

```json
{
  "event": "DomainRegistrationCompleted",
  "url": "https://your-app.com/webhook/domain",
  "method": "POST",
  "headers": {
    "Authorization": "Bearer your-webhook-secret"
  }
}
```

## See Also

- [Domain Pricing Configuration](../products-services/domain-pricing.md)
- [Registrar Module Development](../developer/registrar-modules.md)
- [Domain Transfer Tool](./whmcs-transfer-tool.md)
