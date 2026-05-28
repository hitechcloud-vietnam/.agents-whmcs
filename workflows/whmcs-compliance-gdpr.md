# WHMCS GDPR Compliance Workflow

## Purpose

Comprehensive guide to ensuring WHMCS installations comply with GDPR requirements, including data subject rights, consent management, privacy policies, and data processing documentation.

## Prerequisites

- WHMCS installation
- Understanding of GDPR requirements
- Legal review of policies
- Technical implementation access

## Workflow Steps

### Step 1: GDPR Compliance Assessment

Assess current compliance status:

```php
// modules/addons/gdpr_compliance/assessment.php

class GDPRComplianceAssessment
{
    /**
     * Perform full GDPR compliance assessment
     */
    public function assess(): array
    {
        return [
            'data_collection' => $this->assessDataCollection(),
            'consent_management' => $this->assessConsentManagement(),
            'data_subject_rights' => $this->assessDataSubjectRights(),
            'data_retention' => $this->assessDataRetention(),
            'third_party_sharing' => $this->assessThirdPartySharing(),
            'security_measures' => $this->assessSecurityMeasures(),
            'documentation' => $this->assessDocumentation(),
        ];
    }
    
    /**
     * Assess what personal data is collected
     */
    private function assessDataCollection(): array
    {
        $dataCategories = [
            [
                'category' => 'Client Account Data',
                'fields' => ['firstname', 'lastname', 'email', 'phonenumber', 'companyname', 'address'],
                'legal_basis' => 'Contract',
                'required' => true,
            ],
            [
                'category' => 'Payment Data',
                'fields' => ['payment_method', 'card_last4', 'billing_address'],
                'legal_basis' => 'Contract',
                'required' => true,
            ],
            [
                'category' => 'Service Usage Data',
                'fields' => ['login_history', 'activity_logs', 'support_tickets'],
                'legal_basis' => 'Legitimate Interest',
                'required' => false,
            ],
            [
                'category' => 'Marketing Preferences',
                'fields' => ['email_marketing', 'sms_marketing'],
                'legal_basis' => 'Consent',
                'required' => false,
            ],
        ];
        
        return [
            'compliant' => true,
            'categories' => $dataCategories,
            'action_items' => [],
        ];
    }
    
    /**
     * Assess consent management
     */
    private function assessConsentManagement(): array
    {
        $issues = [];
        
        // Check if consent checkboxes exist on signup
        $signupForm = Capsule::table('tblproduct_groups')->count();
        
        if ($signupForm > 0) {
            // Verify consent fields are present
            $consentFields = Capsule::table('tblcustomfields')
                ->where('type', 'client')
                ->where('fieldname', 'LIKE', '%consent%')
                ->count();
            
            if ($consentFields === 0) {
                $issues[] = 'No consent fields configured for marketing';
            }
        }
        
        return [
            'compliant' => empty($issues),
            'issues' => $issues,
            'action_items' => empty($issues) ? [] : [
                'Add explicit consent checkboxes to signup forms',
                'Record consent timestamps',
            ],
        ];
    }
}
```

### Step 2: Consent Management Implementation

Implement proper consent collection and tracking:

```php
// modules/addons/gdpr_compliance/consent.php

/**
 * Consent tracking class
 */
class ConsentManager
{
    /**
     * Record user consent
     */
    public static function recordConsent(int $userId, string $consentType, string $consentGiven): bool
    {
        $ipAddress = $_SERVER['REMOTE_ADDR'] ?? 'unknown';
        
        return Capsule::table('mod_consent_log')->insert([
            'user_id' => $userId,
            'consent_type' => $consentType,
            'consent_given' => $consentGiven ? 1 : 0,
            'ip_address' => $ipAddress,
            'user_agent' => $_SERVER['HTTP_USER_AGENT'] ?? '',
            'recorded_at' => date('Y-m-d H:i:s'),
        ]) !== false;
    }
    
    /**
     * Get all consent records for a user
     */
    public static function getUserConsents(int $userId): array
    {
        return Capsule::table('mod_consent_log')
            ->where('user_id', $userId)
            ->orderBy('recorded_at', 'desc')
            ->get()
            ->toArray();
    }
    
    /**
     * Check if user has given specific consent
     */
    public static function hasConsent(int $userId, string $consentType): bool
    {
        $latestConsent = Capsule::table('mod_consent_log')
            ->where('user_id', $userId)
            ->where('consent_type', $consentType)
            ->orderBy('recorded_at', 'desc')
            ->first();
        
        return $latestConsent && $latestConsent->consent_given;
    }
    
    /**
     * Revoke consent
     */
    public static function revokeConsent(int $userId, string $consentType): bool
    {
        return self::recordConsent($userId, $consentType, false);
    }
}

/**
 * Hook: Record consent on signup
 */
add_hook('ClientAdd', 1, function($vars) {
    $userId = $vars['user_id'];
    
    // Record marketing consent
    $marketingConsent = $_POST['marketing_consent'] ?? false;
    ConsentManager::recordConsent($userId, 'marketing', $marketingConsent);
    
    // Record terms acceptance
    $termsAccepted = $_POST['accept_terms'] ?? false;
    ConsentManager::recordConsent($userId, 'terms_of_service', $termsAccepted);
    
    // Record privacy policy acceptance
    $privacyAccepted = $_POST['accept_privacy'] ?? false;
    ConsentManager::recordConsent($userId, 'privacy_policy', $privacyAccepted);
});

/**
 * Hook: Record consent on preference change
 */
add_hook('ClientAreaPrimaryNavbar', 1, function($vars) {
    if ($_SERVER['REQUEST_METHOD'] === 'POST' && isset($_POST['update_consent'])) {
        $userId = $_SESSION['uid'];
        
        if (isset($_POST['marketing_email'])) {
            ConsentManager::recordConsent($userId, 'marketing_email', true);
        } else {
            ConsentManager::recordConsent($userId, 'marketing_email', false);
        }
    }
});

/**
 * Create consent tracking table
 */
function gdpr_activate(): array
{
    Capsule::schema()->create('mod_consent_log', function($table) {
        $table->increments('id');
        $table->integer('user_id');
        $table->string('consent_type', 100);
        $table->boolean('consent_given');
        $table->string('ip_address', 45);
        $table->string('user_agent', 255);
        $table->timestamp('recorded_at')->useCurrent();
        $table->index(['user_id', 'consent_type']);
    });
    
    return ['status' => 'success'];
}
```

### Step 3: Data Subject Rights Implementation

Implement GDPR data subject rights:

```php
// modules/addons/gdpr_compliance/data_subject_rights.php

class DataSubjectRights
{
    /**
     * Export all personal data for a user (Right to Access)
     */
    public static function exportUserData(int $userId): array
    {
        $export = [
            'export_date' => date('Y-m-d H:i:s'),
            'user_id' => $userId,
            'data' => [],
        ];
        
        // Account information
        $export['data']['account'] = Capsule::table('tblclients')
            ->where('id', $userId)
            ->first();
        
        // Contacts
        $export['data']['contacts'] = Capsule::table('tblcontacts')
            ->where('userid', $userId)
            ->get()
            ->toArray();
        
        // Services
        $export['data']['services'] = Capsule::table('tblhosting')
            ->where('userid', $userId)
            ->get()
            ->toArray();
        
        // Domains
        $export['data']['domains'] = Capsule::table('tbldomains')
            ->where('userid', $userId)
            ->get()
            ->toArray();
        
        // Invoices
        $export['data']['invoices'] = Capsule::table('tblinvoices')
            ->where('userid', $userId)
            ->get()
            ->toArray();
        
        // Payment records
        $export['data']['payments'] = Capsule::table('tblaccounts')
            ->where('userid', $userId)
            ->get()
            ->toArray();
        
        // Support tickets
        $export['data']['tickets'] = Capsule::table('tbltickets')
            ->where('userid', $userId)
            ->get()
            ->toArray();
        
        // Activity logs
        $export['data']['activity'] = Capsule::table('tblactivitylog')
            ->where('userid', $userId)
            ->get()
            ->toArray();
        
        // Consent records
        $export['data']['consents'] = ConsentManager::getUserConsents($userId);
        
        // Custom fields
        $export['data']['custom_fields'] = Capsule::table('tblcustomfieldsvalues')
            ->where('relid', $userId)
            ->where('type', 'client')
            ->get()
            ->toArray();
        
        return $export;
    }
    
    /**
     * Export data as JSON file
     */
    public static function exportToJson(int $userId): string
    {
        $data = self::exportUserData($userId);
        
        return json_encode($data, JSON_PRETTY_PRINT);
    }
    
    /**
     * Delete user data (Right to Erasure / Right to be Forgotten)
     * 
     * Note: Some data must be retained for legal/compliance reasons
     */
    public static function anonymizeUser(int $userId): array
    {
        $results = [
            'success' => true,
            'deleted' => [],
            'retained' => [],
            'errors' => [],
        ];
        
        // Anonymize account data
        try {
            Capsule::table('tblclients')
                ->where('id', $userId)
                ->update([
                    'firstname' => 'REDACTED',
                    'lastname' => 'REDACTED',
                    'companyname' => '',
                    'email' => "deleted_{$userId}@anonymized.local",
                    'phonenumber' => '',
                    'address1' => '',
                    'address2' => '',
                    'city' => '',
                    'state' => '',
                    'postcode' => '',
                    'country' => '',
                    'password' => hash('sha256', uniqid()),
                    'notes' => 'Account anonymized per GDPR request',
                    'emailverifystatus' => 0,
                ]);
            
            $results['deleted'][] = 'Account information';
        } catch (Exception $e) {
            $results['errors'][] = 'Account: ' . $e->getMessage();
        }
        
        // Anonymize contacts
        try {
            Capsule::table('tblcontacts')
                ->where('userid', $userId)
                ->update([
                    'firstname' => 'REDACTED',
                    'lastname' => 'REDACTED',
                    'email' => "anonymized",
                ]);
            
            $results['deleted'][] = 'Contact information';
        } catch (Exception $e) {
            $results['errors'][] = 'Contacts: ' . $e->getMessage();
        }
        
        // Keep financial records (required by law)
        $results['retained'][] = 'Financial/Invoice records (legal requirement)';
        
        // Log the anonymization request
        Capsule::table('mod_gdpr_requests')->insert([
            'user_id' => $userId,
            'request_type' => 'erasure',
            'processed_at' => date('Y-m-d H:i:s'),
            'anonymized' => true,
        ]);
        
        return $results;
    }
    
    /**
     * Process data portability request (Right to Portability)
     */
    public static function portData(int $userId, string $format = 'json'): string
    {
        $data = self::exportUserData($userId);
        
        if ($format === 'json') {
            return json_encode($data);
        }
        
        // CSV export would require transformation logic
        return json_encode($data);
    }
}
```

### Step 4: Privacy Policy and Notices

Create GDPR-compliant privacy notices:

```php
// modules/addons/gdpr_compliance/notices.php

/**
 * Generate privacy policy content
 */
function getGDPRPrivacyPolicy(): string
{
    return <<<'HTML'
    <h1>Privacy Policy</h1>
    <p>Last updated: [DATE]</p>
    
    <h2>1. Data Controller</h2>
    <p>[Company Name] ("we", "us", or "our") operates this website and is the 
    data controller for the personal data collected.</p>
    
    <h2>2. Personal Data We Collect</h2>
    <ul>
        <li>Contact information (name, email, phone, address)</li>
        <li>Account credentials</li>
        <li>Payment information</li>
        <li>Service usage data</li>
        <li>Communication preferences</li>
    </ul>
    
    <h2>3. Legal Basis for Processing</h2>
    <p>We process your personal data based on:</p>
    <ul>
        <li><strong>Contract:</strong> To provide services you've requested</li>
        <li><strong>Consent:</strong> For marketing communications</li>
        <li><strong>Legitimate Interest:</strong> For security and fraud prevention</li>
        <li><strong>Legal Obligation:</strong> For tax and financial records</li>
    </ul>
    
    <h2>4. Your Rights Under GDPR</h2>
    <p>You have the right to:</p>
    <ul>
        <li>Access your personal data</li>
        <li>Rectify inaccurate data</li>
        <li>Erase your data ("right to be forgotten")</li>
        <li>Restrict processing</li>
        <li>Data portability</li>
        <li>Object to processing</li>
        <li>Withdraw consent</li>
        <li>Lodge a complaint with a supervisory authority</li>
    </ul>
    
    <h2>5. Data Retention</h2>
    <p>We retain your data for as long as your account is active and as required 
    by law. Financial records are retained for 7 years.</p>
    
    <h2>6. Data Sharing</h2>
    <p>We share data with:</p>
    <ul>
        <li>Payment processors</li>
        <li>Service providers</li>
        <li>Legal authorities when required</li>
    </ul>
    
    <h2>7. Contact</h2>
    <p>For GDPR inquiries: privacy@example.com</p>
    HTML;
}

/**
 * Cookie consent notice
 */
function getCookieConsentNotice(): string
{
    return <<<'HTML'
    <div id="cookie-consent" style="position: fixed; bottom: 0; left: 0; right: 0; background: #333; color: white; padding: 20px; z-index: 9999;">
        <p style="margin: 0 0 15px;">
            We use cookies to improve your experience and analyze site traffic. 
            By clicking "Accept", you consent to our use of cookies. 
            <a href="/privacy-policy.php" style="color: #87CEEB;">Learn more</a>
        </p>
        <button onclick="acceptCookies()" style="background: #4CAF50; color: white; border: none; padding: 10px 20px; cursor: pointer;">
            Accept
        </button>
        <button onclick="rejectCookies()" style="background: #666; color: white; border: none; padding: 10px 20px; cursor: pointer;">
            Reject
        </button>
    </div>
    
    <script>
    function acceptCookies() {
        document.cookie = "cookie_consent=accepted; expires=Fri, 31 Dec 9999 23:59:59 GMT; path=/";
        document.getElementById('cookie-consent').style.display = 'none';
    }
    
    function rejectCookies() {
        document.cookie = "cookie_consent=rejected; expires=Fri, 31 Dec 9999 23:59:59 GMT; path=/";
        document.getElementById('cookie-consent').style.display = 'none';
    }
    </script>
    HTML;
}
```

### Step 5: Data Processing Agreement

Create DPA template and management:

```php
// modules/addons/gdpr_compliance/dpa.php

class DataProcessingAgreement
{
    /**
     * Generate DPA for vendor
     */
    public static function generateDPA(string $vendorName, array $processingDetails): string
    {
        $template = <<<'HTML'
DATA PROCESSING AGREEMENT

Between:
[Your Company Name] ("Data Controller")
And:
[VENDOR_NAME] ("Data Processor")

Effective Date: [DATE]

1. SCOPE AND PURPOSE
The Processor agrees to process personal data only on behalf of and in accordance 
with documented instructions from the Controller for the following purposes:
[PROCESSING_DETAILS]

2. NATURE AND PURPOSE OF PROCESSING
[PURPOSE_DESCRIPTION]

3. TYPES OF PERSONAL DATA
- Contact information
- Account data
- [ADDITIONAL_TYPES]

4. CATEGORIES OF DATA SUBJECTS
[CATEGORIES]

5. PROCESSOR OBLIGATIONS
The Processor shall:
a) Process personal data only on documented instructions
b) Ensure confidentiality of personnel
c) Implement appropriate security measures
d) Not engage sub-processors without consent
e) Assist Controller with data subject requests
f) Delete or return data at end of contract
g) Make information available for audits

6. SECURITY MEASURES
[SECURITY_MEASURES]

7. SUB-PROCESSORS
Prior written consent required for sub-processor engagement.

8. DATA BREACH NOTIFICATION
Notify Controller within 48 hours of becoming aware of a breach.

9. LIABILITY
[LIABILITY_TERMS]

Signed:
____________________          ____________________
[Your Company]               [VENDOR_NAME]
Date:                        Date:
HTML;
        
        return str_replace(
            ['[VENDOR_NAME]', '[PROCESSING_DETAILS]', '[DATE]'],
            [$vendorName, implode("\n", $processingDetails), date('Y-m-d')],
            $template
        );
    }
}
```

## Best Practices

1. **Privacy by design** - Build compliance into systems from start
2. **Clear consent** - Unambiguous, affirmative consent
3. **Easy withdrawal** - As easy to withdraw as to give consent
4. **Data minimization** - Only collect what's necessary
5. **Retention policies** - Define how long data is kept
6. **Regular audits** - Review compliance periodically
7. **Document everything** - Record all processing activities
8. **Staff training** - Ensure everyone understands requirements

## Common Pitfalls to Avoid

1. **Pre-ticked boxes** - Consent must be explicit
2. **Hidden processing** - Must inform users clearly
3. **No retention policy** - Keep data indefinitely
4. **Ignoring requests** - Must respond within 30 days
5. **Poor security** - Inadequate protection measures
6. **Missing documentation** - Can't prove compliance
7. **No breach plan** - Not prepared for data breaches
8. **Third-party risks** - Unvetted vendors and processors
