# WHMCS DomainGetInfo Hook Reference

## Overview

The `DomainGetInfo` hook fires when domain information is retrieved in WHMCS. This hook allows you to modify or enrich domain data displayed to clients and admins.

## Hook Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `domainid` | int | The domain ID |
| `domain` | string | The domain name |
| `userid` | int | The client ID |
| `registrationdate` | string | Registration date |
| `expirydate` | string | Expiry date |
| `status` | string | Domain status |
| `dns` | array | DNS settings |
| `nameservers` | array | Nameservers |
| `model` | object | The Domain model instance |

## Example Implementation

```php
<?php
add_hook('DomainGetInfo', 1, function(array $params) {
    // Add custom data to domain info
    $params['custom_data'] = getDomainCustomData($params['domainid']);
    
    // Add WHOIS info
    $params['whois_info'] = getWHOISData($params['domain']);
    
    return $params;
});
```

## Enriching Domain Data

```php
<?php
add_hook('DomainGetInfo', 1, function(array $params) {
    /** @var \WHMCS\Domain\Domain $model */
    $model = $params['model'];
    
    // 1. Get DNS propagation status
    $dnsStatus = checkDNSPropagation($params['domain']);
    $params['dns_propagated'] = $dnsStatus['complete'];
    $params['dns_propagation_percent'] = $dnsStatus['percent'];
    
    // 2. Check SSL certificate status
    $sslStatus = checkDomainSSL($params['domain']);
    $params['ssl_enabled'] = $sslStatus['valid'];
    $params['ssl_expires'] = $sslStatus['expires'];
    
    // 3. Get website status
    $websiteStatus = checkWebsiteStatus($params['domain']);
    $params['website_online'] = $websiteStatus['online'];
    $params['website_response_time'] = $websiteStatus['response_time'];
    
    // 4. Add registrar lock status
    $params['registrar_lock'] = checkRegistrarLock($params['domainid']);
    
    // 5. Get nameserver health
    $params['nameservers_healthy'] = checkNameserverHealth($params['nameservers']);
    
    return $params;
});
```

## Custom Domain Information

```php
<?php
add_hook('DomainGetInfo', 1, function(array $params) {
    // 1. Add renewal price comparison
    $currentPrice = getCurrentRenewalPrice($params['domainid']);
    $marketPrice = getMarketRenewalPrice($params['domain']);
    $params['renewal_savings'] = $marketPrice - $currentPrice;
    
    // 2. Add transfer availability
    $params['transfer_available'] = isTransferAvailable($params['domain']);
    $params['transfer_price'] = getTransferPrice($params['domain']);
    
    // 3. Add privacy protection status
    $params['privacy_protected'] = isPrivacyEnabled($params['domainid']);
    
    // 4. Add email forwarding status
    $params['email_forwards'] = getEmailForwardCount($params['domainid']);
    
    // 5. Add auto-renewal status
    $params['auto_renew'] = isAutoRenewEnabled($params['domainid']);
    
    return $params;
});
```

## Status Indicators

```php
<?php
add_hook('DomainGetInfo', 1, function(array $params) {
    $domainId = (int)$params['domainid'];
    
    // 1. Calculate days until expiry for status display
    $expiryDate = strtotime($params['expirydate']);
    $daysUntilExpiry = ceil(($expiryDate - time()) / 86400);
    
    if ($daysUntilExpiry < 0) {
        $params['expiry_status'] = 'expired';
        $params['days_expired'] = abs($daysUntilExpiry);
    } elseif ($daysUntilExpiry <= 30) {
        $params['expiry_status'] = 'critical';
    } elseif ($daysUntilExpiry <= 60) {
        $params['expiry_status'] = 'warning';
    } else {
        $params['expiry_status'] = 'healthy';
    }
    $params['days_until_expiry'] = $daysUntilExpiry;
    
    // 2. Add verification status
    $params['email_verified'] = isDomainEmailVerified($params['domainid']);
    $params['dns_verified'] = isDNSVerified($params['domainid']);
    
    // 3. Check for pending operations
    $pendingOps = getPendingDomainOperations($domainId);
    $params['pending_operation'] = !empty($pendingOps);
    $params['pending_operation_type'] = $pendingOps['type'] ?? null;
    
    return $params;
});
```

## Use Cases

- **DNS Status**: Show propagation status
- **SSL Status**: Display certificate info
- **Health Checks**: Nameserver and website status
- **Pricing Info**: Renewal and transfer prices
- **Status Indicators**: Expiry warnings, verification status

## Notes

- Fires when domain data is retrieved
- Can modify data before display
- Useful for enriching client area information
- Consider caching external lookups

## Related Hooks

- `DomainRegister` - Domain registration
- `DomainTransfer` - Domain transfer
- `DomainRenew` - Domain renewal

## See Also

- [WHMCS Hooks Documentation](https://docs.whmcs.com/Hooks)
- [Domain Management](../whmcs-domain-management.md)