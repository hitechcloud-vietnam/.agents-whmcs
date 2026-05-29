# WHMCS Domain Hooks

## Overview

Domain hooks allow customization of domain registration, transfer, renewal, and management operations.

## Available Domain Hooks

### Domain Registered Hook

```php
<?php
// Triggered when a new domain is registered
add_hook('DomainRegistered', 1, function(array $vars) {
    $domainId = $vars['domain_id'];
    $domain = $vars['domain'];
    $registrationPeriod = $vars['registration_period'];
    $userId = $vars['user_id'];
    
    // Setup DNS records
    setupDomainDns($domain);
    
    // Configure email forwarding
    setupEmailForwarding($domain, $userId);
    
    // Enable domain privacy
    if ($vars['id_protection']) {
        enableIdProtection($domain);
    }
    
    // Sync to external systems
    syncDomainToCrm($domain, $vars);
    
    // Send welcome email
    sendDomainWelcomeEmail($domainId);
    
    // Schedule renewal reminder
    scheduleRenewalReminders($domainId, $vars['expiry_date']);
    
    return ['success' => true];
});
```

### Domain Transferred Hook

```php
<?php
// Triggered when a domain transfer completes
add_hook('DomainTransferred', 1, function(array $vars) {
    $domainId = $vars['domain_id'];
    $domain = $vars['domain'];
    $userId = $vars['user_id'];
    
    // Update DNS to default nameservers
    setupDefaultDns($domain);
    
    // Import existing DNS records
    importDnsRecords($domain);
    
    // Verify ownership
    verifyDomainOwnership($domain);
    
    // Sync to external systems
    syncDomainToCrm($domain, $vars);
    
    return ['success' => true];
});
```

### Domain Renewed Hook

```php
<?php
// Triggered when a domain is renewed
add_hook('DomainRenewed', 1, function(array $vars) {
    $domainId = $vars['domain_id'];
    $domain = $vars['domain'];
    $newExpiryDate = $vars['expiry_date'];
    $years = $vars['renewal_years'];
    
    // Update SSL certificates
    renewSSLCertificates($domain);
    
    // Extend DNS hosting
    extendDnsService($domain, $years);
    
    // Update monitoring
    extendMonitoringExpiry($domainId, $newExpiryDate);
    
    // Send renewal confirmation
    sendRenewalConfirmation($domainId, $newExpiryDate);
    
    // Log renewal
    logRenewal($domainId, $years);
    
    return ['success' => true];
});
```

### Domain Expired Hook

```php
<?php
// Triggered when a domain expires
add_hook('DomainExpired', 1, function(array $vars) {
    $domainId = $vars['domain_id'];
    $domain = $vars['domain'];
    $daysSinceExpiry = $vars['days_since_expiry'];
    
    // Take appropriate action based on days expired
    if ($daysSinceExpiry <= 30) {
        // Grace period - attempt auto-renewal
        attemptAutoRenewal($domainId);
    } elseif ($daysSinceExpiry <= 60) {
        // Redemption period - notify owner
        notifyRedemptionPeriod($domainId);
    } else {
        // Release domain
        releaseDomain($domainId);
    }
    
    // Update DNS
    pointToExpiredPage($domain);
    
    return ['success' => true];
});
```

### Domain Transferred Away Hook

```php
<?php
// Triggered when domain transfers away
add_hook('DomainTransferCompleted', 1, function(array $vars) {
    $domainId = $vars['domain_id'];
    $domain = $vars['domain'];
    
    // Clean up local records
    archiveDomainData($domainId);
    
    // Disable services
    disableDomainServices($domain);
    
    // Remove from DNS
    removeDnsRecords($domain);
    
    // Revoke SSL
    revokeCertificates($domain);
    
    // Log transfer
    logTransferOut($domainId, $vars);
    
    return ['success' => true];
});
```

### Domain DNS Updated Hook

```php
<?php
// Triggered when DNS records are updated
add_hook('DomainDnsUpdated', 1, function(array $vars) {
    $domainId = $vars['domain_id'];
    $domain = $vars['domain'];
    $changes = $vars['changes'];
    
    // Log DNS changes
    logDnsChanges($domainId, $changes);
    
    // Propagate to CDN
    propagateToCdn($domain, $changes);
    
    // Validate records
    validateDnsRecords($domain);
    
    // Send notification
    notifyDnsUpdate($domainId, $changes);
    
    return ['success' => true];
});
```

### Domain Contact Updated Hook

```php
<?php
// Triggered when domain contact details are updated
add_hook('DomainContactUpdated', 1, function(array $vars) {
    $domainId = $vars['domain_id'];
    $changes = $vars['changes'];
    
    // Update WHOIS
    updateWhoisContact($domainId, $vars['contacts']);
    
    // Sync to registrar
    syncContactsToRegistrar($domainId);
    
    // Log changes
    logContactChanges($domainId, $changes);
    
    return ['success' => true];
});
```

## Comprehensive Domain Handler

```php
<?php
class DomainHookHandler {
    
    public function register(): void
    {
        add_hook('DomainRegistered', 1, [$this, 'handleRegistered']);
        add_hook('DomainTransferred', 1, [$this, 'handleTransferred']);
        add_hook('DomainRenewed', 1, [$this, 'handleRenewed']);
        add_hook('DomainExpired', 1, [$this, 'handleExpired']);
        add_hook('DomainTransferCompleted', 1, [$this, 'handleTransferOut']);
        add_hook('DomainDnsUpdated', 1, [$this, 'handleDnsUpdate']);
        add_hook('DomainContactUpdated', 1, [$this, 'handleContactUpdate']);
    }
    
    public function handleRegistered(array $vars): array
    {
        $this->setupDns($vars['domain']);
        $this->enablePrivacy($vars);
        $this->syncCrm($vars);
        $this->scheduleReminders($vars);
        return ['success' => true];
    }
    
    public function handleTransferred(array $vars): array
    {
        $this->importDns($vars['domain']);
        $this->verifyOwnership($vars['domain']);
        return ['success' => true];
    }
    
    public function handleRenewed(array $vars): array
    {
        $this->renewSsl($vars['domain']);
        $this->extendServices($vars);
        return ['success' => true];
    }
    
    public function handleExpired(array $vars): array
    {
        $this->handleExpiry($vars);
        return ['success' => true];
    }
    
    public function handleTransferOut(array $vars): array
    {
        $this->cleanup($vars['domain_id']);
        return ['success' => true];
    }
    
    public function handleDnsUpdate(array $vars): array
    {
        $this->propagateDns($vars['domain']);
        $this->validateRecords($vars['domain']);
        return ['success' => true];
    }
    
    public function handleContactUpdate(array $vars): array
    {
        $this->updateWhois($vars['domain_id']);
        return ['success' => true];
    }
    
    private function setupDns(string $domain): void
    {
        // Setup default DNS
    }
    
    private function enablePrivacy(array $vars): void
    {
        if ($vars['id_protection'] ?? false) {
            // Enable ID protection
        }
    }
    
    private function syncCrm(array $vars): void
    {
        // Sync to CRM
    }
    
    private function scheduleReminders(array $vars): void
    {
        // Schedule renewal reminders
    }
    
    private function importDns(string $domain): void
    {
        // Import existing DNS
    }
    
    private function verifyOwnership(string $domain): void
    {
        // Verify ownership
    }
    
    private function renewSsl(string $domain): void
    {
        // Renew SSL certificates
    }
    
    private function extendServices(array $vars): void
    {
        // Extend related services
    }
    
    private function handleExpiry(array $vars): void
    {
        $days = $vars['days_since_expiry'];
        if ($days <= 30) {
            $this->attemptAutoRenewal($vars['domain_id']);
        }
    }
    
    private function attemptAutoRenewal(int $domainId): void
    {
        // Attempt auto-renewal
    }
    
    private function cleanup(int $domainId): void
    {
        // Cleanup after transfer out
    }
    
    private function propagateDns(string $domain): void
    {
        // Propagate DNS changes
    }
    
    private function validateRecords(string $domain): void
    {
        // Validate DNS records
    }
    
    private function updateWhois(int $domainId): void
    {
        // Update WHOIS contact
    }
}

$handler = new DomainHookHandler();
$handler->register();
```

## Related Documentation

- [WHMCS Order Hooks](/docs/whmcs-order-hooks.md)
- [WHMCS Registrar Module Development](/docs/whmcs-registrar-dev.md)