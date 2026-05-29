# WHMCS User Consent Workflow

## Overview
This workflow implements user consent collection and management for WHMCS.

## Prerequisites
- WHMCS with consent tracking
- Admin access for consent settings
- Privacy requirements

## Step-by-Step Process

### Step 1: Create User Consent Manager
```php
<?php
// /includes/privacy/UserConsentManager.php

class UserConsentManager {
    private $consentTypes = [
        'marketing' => 'Marketing Communications',
        'analytics' => 'Analytics Cookies',
        'third_party' => 'Third Party Sharing'
    ];

    /**
     * Record user consent
     */
    public function recordConsent(int $clientId, string $type, bool $granted): void
    {
        Capsule::table('mod_user_consent')->insert([
            'client_id' => $clientId,
            'consent_type' => $type,
            'granted' => $granted,
            'ip_address' => $_SERVER['REMOTE_ADDR'] ?? '',
            'recorded_at' => date('Y-m-d H:i:s')
        ]);
    }

    /**
     * Check if user has given consent
     */
    public function hasConsent(int $clientId, string $type): bool
    {
        return Capsule::table('mod_user_consent')
            ->where('client_id', $clientId)
            ->where('consent_type', $type)
            ->where('granted', 1)
            ->orderBy('recorded_at', 'DESC')
            ->first() !== null;
    }

    /**
     * Withdraw consent
     */
    public function withdrawConsent(int $clientId, string $type): void
    {
        $this->recordConsent($clientId, $type, false);
    }
}
```

### Step 2: Consent Hooks
```php
<?php
// /includes/hooks/consent_hooks.php

add_hook('ClientAreaPage', 1, function($vars) {
    $consentManager = new UserConsentManager();

    // Check marketing consent before sending emails
    if ($consentManager->hasConsent($_SESSION['uid'], 'marketing')) {
        // Allow marketing
    } else {
        // Block marketing
    }
});
```

## Related Workflows
- [WHMCS Consent Management](./whmcs-consent-management.md)
- [WHMCS GDPR Compliance](./whmcs-gdpr-compliance.md)