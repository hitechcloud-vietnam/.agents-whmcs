# WHMCS Consent Management Workflow

## Overview
This workflow implements consent management for privacy compliance in WHMCS.

## Prerequisites
- WHMCS with client data collection
- Admin access for consent configuration
- Privacy requirements defined

## Step-by-Step Process

### Step 1: Create Consent Manager
```php
<?php
// /includes/privacy/ConsentManager.php

class ConsentManager {
    private $consentTypes = [
        'terms_of_service' => [
            'name' => 'Terms of Service',
            'required' => true,
            'retention_days' => 3650 // 10 years
        ],
        'privacy_policy' => [
            'name' => 'Privacy Policy',
            'required' => true,
            'retention_days' => 3650
        ],
        'marketing_email' => [
            'name' => 'Marketing Emails',
            'required' => false,
            'retention_days' => 365
        ],
        'analytics' => [
            'name' => 'Analytics',
            'required' => false,
            'retention_days' => 365
        ],
        'third_party_sharing' => [
            'name' => 'Third Party Data Sharing',
            'required' => false,
            'retention_days' => 365
        ]
    ];

    /**
     * Record consent
     */
    public function recordConsent(int $clientId, string $consentType, bool $granted, array $context = []): void
    {
        Capsule::table('mod_consent_records')->insert([
            'client_id' => $clientId,
            'consent_type' => $consentType,
            'granted' => $granted ? 1 : 0,
            'ip_address' => $_SERVER['REMOTE_ADDR'] ?? null,
            'user_agent' => $_SERVER['HTTP_USER_AGENT'] ?? null,
            'context' => json_encode($context),
            'recorded_at' => date('Y-m-d H:i:s')
        ]);

        // Update client preferences
        if ($consentType === 'marketing_email') {
            Capsule::table('tblclients')
                ->where('id', $clientId)
                ->update(['newsletter' => $granted ? 1 : 0]);
        }
    }

    /**
     * Check if client has given consent
     */
    public function hasConsent(int $clientId, string $consentType): bool
    {
        $consent = Capsule::table('mod_consent_records')
            ->where('client_id', $clientId)
            ->where('consent_type', $consentType)
            ->where('granted', 1)
            ->orderBy('recorded_at', 'DESC')
            ->first();

        if (!$consent) {
            return false;
        }

        // Check if consent is still valid
        $policy = $this->consentTypes[$consentType] ?? null;

        if (!$policy) {
            return false;
        }

        $expiryDays = $policy['retention_days'];
        $consentExpiry = date('Y-m-d H:i:s', strtotime("+{$expiryDays} days", strtotime($consent->recorded_at)));

        return $consentExpiry > date('Y-m-d H:i:s');
    }

    /**
     * Withdraw consent
     */
    public function withdrawConsent(int $clientId, string $consentType): void
    {
        $this->recordConsent($clientId, $consentType, false, ['action' => 'withdrawal']);
    }
}
```

### Step 2: Consent Hooks
```php
<?php
// /includes/hooks/consent_hooks.php

$consentManager = new ConsentManager();

// Record consent at registration
add_hook('ClientAdd', 1, function($vars) use ($consentManager) {
    // Terms acceptance
    if (!empty($_POST['accept_terms'])) {
        $consentManager->recordConsent($vars['userid'], 'terms_of_service', true, [
            'version' => getCurrentTermsVersion()
        ]);
    }

    // Marketing consent
    if (!empty($_POST['marketing_consent'])) {
        $consentManager->recordConsent($vars['userid'], 'marketing_email', true);
    }
});

// Check consent before marketing
add_hook('PreEmailSend', 1, function($vars) use ($consentManager) {
    if ($vars['type'] === 'marketing') {
        $clientId = getClientIdByEmail($vars['to']);

        if (!$consentManager->hasConsent($clientId, 'marketing_email')) {
            throw new Exception('Client has not consented to marketing emails');
        }
    }
});
```

## Consent Types

| Type | Required | Default | Retention |
|------|----------|---------|-----------|
| Terms of Service | Yes | Unchecked | 10 years |
| Privacy Policy | Yes | Unchecked | 10 years |
| Marketing | No | Unchecked | 1 year |
| Analytics | No | Per policy | 1 year |

## Related Workflows
- [WHMCS GDPR Compliance](./whmcs-gdpr-compliance.md)
- [WHMCS Privacy Policy](./whmcs-privacy-policy.md)