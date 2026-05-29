# WHMCS Terms of Service Workflow

## Overview
This workflow implements Terms of Service management and enforcement for WHMCS.

## Prerequisites
- WHMCS with client acceptance tracking
- Legal access for terms management
- Terms version control

## Step-by-Step Process

### Step 1: Terms of Service Structure
```
TERMS SECTIONS:
□ Acceptance of Terms
□ Description of Services
□ Account Registration
□ Acceptable Use Policy
□ Service Levels (SLA)
□ Payment Terms
□ Cancellation and Termination
□ Limitation of Liability
□ Intellectual Property
□ User Content
□ Indemnification
□ Governing Law
□ Dispute Resolution
□ Changes to Terms
□ Contact Information
```

### Step 2: Terms Management
```php
<?php
// /includes/legal/TermsManager.php

class TermsManager {
    private $versionHistory = [];

    /**
     * Create new terms version
     */
    public function createVersion(string $content, string $locale = 'en'): int
    {
        $version = $this->generateVersionId();

        Capsule::table('mod_terms_versions')->insert([
            'version' => $version,
            'content' => $content,
            'locale' => $locale,
            'effective_date' => date('Y-m-d', strtotime('+30 days')), // 30 day grace period
            'created_at' => date('Y-m-d H:i:s'),
            'created_by' => adminId()
        ]);

        return $version;
    }

    /**
     * Get current active terms
     */
    public function getCurrentTerms(string $locale = 'en'): array
    {
        return Capsule::table('mod_terms_versions')
            ->where('locale', $locale)
            ->where('effective_date', '<=', date('Y-m-d'))
            ->orderBy('effective_date', 'DESC')
            ->first();
    }

    /**
     * Require agreement to terms
     */
    public function requireAgreement(int $clientId, int $termsVersion): bool
    {
        $hasAgreed = Capsule::table('mod_terms_agreements')
            ->where('client_id', $clientId)
            ->where('terms_version', $termsVersion)
            ->exists();

        if (!$hasAgreed) {
            // Redirect to terms acceptance
            redirectToTermsAcceptance($termsVersion);
        }

        return true;
    }

    /**
     * Record client agreement
     */
    public function recordAgreement(int $clientId, int $termsVersion): void
    {
        Capsule::table('mod_terms_agreements')->insert([
            'client_id' => $clientId,
            'terms_version' => $termsVersion,
            'agreed_at' => date('Y-m-d H:i:s'),
            'ip_address' => $_SERVER['REMOTE_ADDR'],
            'user_agent' => $_SERVER['HTTP_USER_AGENT']
        ]);
    }
}
```

### Step 3: Terms Integration Hooks
```php
<?php
// /includes/hooks/terms_hooks.php

add_hook('ClientLogin', 1, function($vars) {
    $termsManager = new TermsManager();
    $currentTerms = $termsManager->getCurrentTerms();

    if ($currentTerms) {
        $needsAgreement = $termsManager->requireAgreement($vars['userid'], $currentTerms->id);
    }
});

add_hook('OrderFormSubmit', 1, function($vars) {
    $termsManager = new TermsManager();
    $currentTerms = $termsManager->getCurrentTerms();

    if ($currentTerms) {
        // Verify terms checkbox was checked
        if (empty($_POST['accept_terms'])) {
            throw new Exception('You must accept the Terms of Service');
        }

        // Record agreement for new client
        if (isset($vars['clientid'])) {
            $termsManager->recordAgreement($vars['clientid'], $currentTerms->id);
        }
    }
});
```

### Step 4: Terms Version Management
```php
<?php
// Track and manage terms versions

class TermsVersionManager {
    /**
     * Get version history
     */
    public function getVersionHistory(): array
    {
        return Capsule::select("
            SELECT
                v.*,
                a.username as created_by,
                COUNT(aa.id) as agreements
            FROM mod_terms_versions v
            LEFT JOIN tbladmins a ON v.created_by = a.id
            LEFT JOIN mod_terms_agreements aa ON v.version = aa.terms_version
            GROUP BY v.id
            ORDER BY v.effective_date DESC
        ");
    }

    /**
     * Get clients pending agreement
     */
    public function getPendingAgreements(int $termsId): array
    {
        $currentVersion = Capsule::table('mod_terms_versions')
            ->where('id', $termsId)
            ->first();

        return Capsule::select("
            SELECT
                c.id,
                c.email,
                c.firstname,
                c.lastname,
                aa.agreed_at
            FROM tblclients c
            LEFT JOIN mod_terms_agreements aa ON c.id = aa.client_id
                AND aa.terms_version = ?
            WHERE aa.id IS NULL
        ", [$currentVersion->version]);
    }
}
```

## Terms of Service Best Practices

1. Clear, plain language
2. Regular updates and version control
3. Grace period for new terms
4. Easy acceptance mechanism
5. Proof of acceptance (IP, timestamp, user agent)
6. Archive of previous versions
7. Notification of changes

## Related Workflows
- [WHMCS GDPR Compliance](./whmcs-gdpr-compliance.md)
- [WHMCS Privacy Policy](./whmcs-privacy-policy.md)