# WHMCS Privacy Policy Workflow

## Overview
This workflow implements privacy policy management and compliance for WHMCS.

## Prerequisites
- WHMCS with client data collection
- Legal/compliance access
- Privacy policy template

## Step-by-Step Process

### Step 1: Privacy Policy Components
```
PRIVACY POLICY SECTIONS:
□ Information We Collect
□ How We Use Information
□ Information Sharing
□ Data Retention
□ Security Measures
□ Your Rights
□ Cookies and Tracking
□ International Transfers
□ Children's Privacy
□ Changes to Policy
□ Contact Information
```

### Step 2: Dynamic Privacy Policy
```php
<?php
// /includes/privacy/PrivacyPolicyManager.php

class PrivacyPolicyManager {
    private $supportedLanguages = ['en', 'es', 'fr', 'de'];

    /**
     * Generate privacy policy based on jurisdiction
     */
    public function generatePolicy(string $locale, array $jurisdiction = []): string
    {
        $language = $this->getSupportedLanguage($locale);

        $policy = $this->loadTemplate($language);

        // Add jurisdiction-specific clauses
        if (in_array('EU', $jurisdiction)) {
            $policy .= $this->getEUClauses();
        }

        if (in_array('CA', $jurisdiction)) {
            $policy .= $this->getCanadaClauses();
        }

        // Insert dynamic content
        $policy = str_replace(
            ['{{company_name}}', '{{effective_date}}', '{{contact_email}}'],
            [getConfig('company_name'), date('F j, Y'), getConfig('support_email')],
            $policy
        );

        return $policy;
    }

    /**
     * Get data processing purposes
     */
    public function getDataProcessingPurposes(): array
    {
        return [
            'service_delivery' => [
                'name' => 'Service Delivery',
                'description' => 'Providing hosting, domain, and related services'
            ],
            'billing' => [
                'name' => 'Billing and Payments',
                'description' => 'Processing payments and generating invoices'
            ],
            'support' => [
                'name' => 'Customer Support',
                'description' => 'Responding to support requests'
            ],
            'marketing' => [
                'name' => 'Marketing Communications',
                'description' => 'Sending promotional communications (with consent)',
                'requires_consent' => true
            ],
            'legal' => [
                'name' => 'Legal Compliance',
                'description' => 'Fulfilling legal obligations'
            ]
        ];
    }
}
```

### Step 3: Privacy Policy Hook
```php
<?php
// /includes/hooks/privacy_policy_hooks.php

add_hook('ClientAreaFooterOutput', 1, function($vars) {
    $locale = $_SESSION['Locale'] ?? 'en';

    $policyManager = new PrivacyPolicyManager();
    $policy = $policyManager->generatePolicy($locale);

    // Show in footer
    return '<div class="privacy-policy">' . $policy . '</div>';
});
```

### Step 4: Privacy Policy Updates
```php
<?php
// Track policy version and consent

class PrivacyPolicyVersion {
    public function trackVersion(int $policyId, string $version): void
    {
        Capsule::table('mod_privacy_policy_versions')->insert([
            'policy_id' => $policyId,
            'version' => $version,
            'effective_date' => date('Y-m-d'),
            'created_at' => date('Y-m-d H:i:s')
        ]);
    }

    public function requireReConsent(int $clientId, int $policyId): bool
    {
        $currentVersion = Capsule::table('mod_privacy_policy_versions')
            ->where('policy_id', $policyId)
            ->orderBy('created_at', 'DESC')
            ->first();

        $consent = Capsule::table('mod_client_consent')
            ->where('client_id', $clientId)
            ->where('consent_type', 'privacy_policy')
            ->orderBy('created_at', 'DESC')
            ->first();

        return !$consent || $consent->version !== $currentVersion->version;
    }
}
```

## Privacy Policy Best Practices

1. Clear and plain language
2. Specific data categories listed
3. Third-party sharing disclosed
4. Retention periods specified
5. User rights clearly explained
6. Contact information provided
7. Regular updates and version tracking

## Related Workflows
- [WHMCS GDPR Compliance](./whmcs-gdpr-compliance.md)
- [WHMCS Consent Management](./whmcs-consent-management.md)