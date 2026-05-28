# WHMCS Marketplace Submission Guide

**Version:** 8.x | **Updated:** 2026-05-29
**Related Skills:** `module-packaging-guide`, `module-release-checklist`, `module-versioning-guide`, `whmcs-testing-qa`

---

## Overview

This guide covers the process of submitting modules to the WHMCS Marketplace, including preparation, requirements, review process, and post-submission management.

---

## Pre-Submission Requirements

### Developer Account Setup

```php
<?php
/**
 * WHMCS Marketplace Developer Account Prerequisites
 */

// 1. Create WHMCS account at marketplace.whmcs.com
// 2. Verify email address
// 3. Complete developer profile:
//    - Company name or personal name
//    - Contact email
//    - Website URL
//    - Support email/URL
//    - Tax information (for paid modules)
```

### Module Requirements

| Requirement | Description |
|-------------|-------------|
| WHMCS Version | Must support WHMCS 8.0+ |
| PHP Version | PHP 7.4 or higher |
| License | Must include license file |
| Documentation | Installation and usage guide |
| Compatibility | Test with latest WHMCS version |
| Security | No security vulnerabilities |
| Code Quality | Follows WHMCS coding standards |

---

## Submission Process

### Step 1: Prepare Your Package

```
your_module_v1.0.0/
├── module.xml              # Module manifest
├── CHANGELOG.md            # Version history
├── README.md               # Installation guide
├── LICENSE                 # License file
│
└── modules/
    ├── addons/
    │   └── your_module/
    │       ├── your_module.php
    │       └── ...
    ├── servers/
    │   └── your_server/
    │       └── your_server.php
    └── gateways/
        └── your_gateway/
            └── your_gateway.php
```

### Step 2: Create Module Manifest

```xml
<?xml version="1.0" encoding="UTF-8"?>
<module>
    <name>Your Module Name</name>
    <version>1.0.0</version>
    <description>
        Brief description of your module's functionality.
        Can span multiple lines.
    </description>
    <author>Your Name or Company</author>
    <authorUrl>https://example.com</authorUrl>
    <authorEmail>support@example.com</authorEmail>

    <license>proprietary</license>
    <!-- or: <license>mit</license> -->
    <!-- or: <license>gpl-3.0</license> -->

    <requires>
        <php version="7.4"/>
        <whmcs version="8.0" min="8.0.0" max="8.9.9"/>
    </requires>

    <types>
        <type>addon</type>
    </types>

    <files>
        <file>modules/addons/your_module/</file>
    </files>

    <hooks>
        <hook>ClientAdd</hook>
        <hook>InvoiceCreation</hook>
    </hooks>
</module>
```

### Step 3: Write Documentation

```markdown
# Your Module Name

Brief description of what your module does.

## Features

- Feature 1
- Feature 2
- Feature 3

## Requirements

- WHMCS 8.0 or higher
- PHP 7.4 or higher
- [External dependency if any]

## Installation

1. Download the module package
2. Extract the archive contents
3. Upload the `modules` folder to your WHMCS root
4. Navigate to Setup > Addon Modules
5. Find and activate "Your Module"
6. Click Configure to enter your settings

## Configuration

| Setting | Description | Required |
|---------|-------------|----------|
| API Key | Your API key | Yes |
| Webhook URL | Webhook endpoint | Yes |
| Debug Mode | Enable debug logging | No |

## Usage

Explain how to use the module after installation.

### Screenshots

Include screenshots showing:
- Admin configuration page
- Module in action
- Client area (if applicable)

## Troubleshooting

### Common Issues

**Q: Module doesn't activate**
A: Check that all files were uploaded correctly and PHP version meets requirements.

**Q: API errors**
A: Verify your API key is correct and the endpoint is accessible.

## Support

- Email: support@example.com
- Website: https://example.com/support
- Documentation: https://docs.example.com

## License

Proprietary license - see LICENSE file
```

---

## Marketplace Listing Details

### Module Information Form

```php
<?php
/**
 * Marketplace submission data structure
 */

$submissionData = [
    // Basic Information
    'name' => 'Your Module Name',
    'tagline' => 'One-line description',
    'description' => <<<HTML
Full HTML description of your module.
Include:
- What it does
- Key features
- Target audience
- Benefits
HTML,

    // Category Selection
    'category' => 'billing', // billing, domains, provisioning, support, other

    // Pricing
    'price' => 0,  // 0 for free
    'currency' => 'USD',
    'license_type' => 'single', // single, unlimited, OEM

    // Media
    'logo' => '/path/to/logo.png',  // 300x150px, PNG
    'screenshots' => [
        '/path/to/screenshot1.png',
        '/path/to/screenshot2.png',
    ],
    'video_url' => 'https://youtube.com/watch?v=...', // Optional

    // Tags
    'tags' => ['invoice', 'billing', 'automation'],

    // Support
    'support_email' => 'support@example.com',
    'support_url' => 'https://example.com/support',
    'documentation_url' => 'https://docs.example.com',
];
```

### Listing Categories

| Category | Description |
|----------|-------------|
| `billing` | Payment processing, invoicing |
| `domains` | Domain management, registrars |
| `provisioning` | Server automation, hosting |
| `support` | Helpdesk, ticketing |
| `marketing` | CRM, affiliate tools |
| `utilities` | Admin tools, utilities |
| `other` | Miscellaneous |

---

## Review Process

### What Reviewers Check

```php
<?php
/**
 * Review checklist used by WHMCS marketplace team
 */

$reviewChecklist = [
    // Installation
    'install_files_present' => 'All required files included',
    'install_instructions_clear' => 'README contains clear instructions',
    'no_missing_dependencies' => 'All dependencies documented',

    // Functionality
    'module_activates' => 'Module activates without errors',
    'features_work' => 'All advertised features functional',
    'settings_save' => 'Configuration persists correctly',
    'no_errors_in_log' => 'No PHP errors in activity log',

    // Security
    'no_sql_injection' => 'No SQL injection vulnerabilities',
    'no_xss' => 'No cross-site scripting issues',
    'csrf_protection' => 'Forms use CSRF tokens',
    'no_hardcoded_secrets' => 'No API keys in code',
    'proper_input_sanitization' => 'All input sanitized',

    // Code Quality
    'psr_compliance' => 'Follows PHP coding standards',
    'no_deprecated_functions' => 'No deprecated PHP functions',
    'proper_error_handling' => 'Errors handled gracefully',
    'logging_implemented' => 'Operations logged appropriately',

    // Compatibility
    'whmcs_8_compatible' => 'Works with WHMCS 8.x',
    'php_7_4_plus' => 'Supports PHP 7.4+',
    'database_migrations' => 'Proper migration handling',

    // Documentation
    'changelog_present' => 'CHANGELOG.md included',
    'license_present' => 'LICENSE file included',
    'screenshots_provided' => 'Screenshots demonstrate features',
];
```

### Common Rejection Reasons

```php
<?php
/**
 * Common reasons for marketplace rejection
 */

// 1. Missing or incomplete documentation
$issues[] = 'README.md missing or incomplete';
$issues[] = 'Installation steps unclear';
$issues[] = 'No changelog provided';

// 2. Security vulnerabilities
$issues[] = 'SQL injection in user input';
$issues[] = 'XSS vulnerability in output';
$issues[] = 'Missing CSRF tokens';
$issues[] = 'Hardcoded passwords/keys';

// 3. Functionality issues
$issues[] = 'Module fails to activate';
$issues[] = 'Settings not saving';
$issues[] = 'Required features not implemented';

// 4. Code quality
$issues[] = 'Deprecated PHP functions used';
$issues[] = 'Code not following standards';
$issues[] = 'No error handling';

// 5. Compatibility
$issues[] = 'Not compatible with WHMCS 8.0';
$issues[] = 'Requires unsupported PHP version';
```

---

## Submission API

### Using the Marketplace API

```php
<?php
/**
 * WHMCS Marketplace API integration
 */

use WHMCS\Marketplace\Submission;

class MarketplaceSubmission
{
    private string $apiKey;
    private string $apiUrl = 'https://marketplace.whmcs.com/api/v1';

    public function __construct(string $apiKey)
    {
        $this->apiKey = $apiKey;
    }

    /**
     * Submit module for review
     */
    public function submit(array $moduleData): array
    {
        return $this->request('POST', '/modules', [
            'module' => $moduleData,
        ]);
    }

    /**
     * Update existing submission
     */
    public function update(int $moduleId, array $data): array
    {
        return $this->request('PUT', "/modules/{$moduleId}", [
            'module' => $data,
        ]);
    }

    /**
     * Upload module package
     */
    public function uploadPackage(int $moduleId, string $filePath): array
    {
        $boundary = md5(microtime());
        $filename = basename($filePath);

        $body = "--{$boundary}\r\n";
        $body .= "Content-Disposition: form-data; name=\"file\"; filename=\"{$filename}\"\r\n";
        $body .= "Content-Type: application/zip\r\n\r\n";
        $body .= file_get_contents($filePath) . "\r\n";
        $body .= "--{$boundary}--\r\n";

        return $this->request('POST', "/modules/{$moduleId}/upload", [], $body);
    }

    /**
     * Get submission status
     */
    public function getStatus(int $moduleId): array
    {
        return $this->request('GET', "/modules/{$moduleId}/status");
    }

    /**
     * Make API request
     */
    private function request(
        string $method,
        string $endpoint,
        array $data = [],
        string $body = null
    ): array {

        $ch = curl_init($this->apiUrl . $endpoint);

        $headers = [
            'Authorization: Bearer ' . $this->apiKey,
            'Content-Type: application/json',
        ];

        curl_setopt_array($ch, [
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_CUSTOMREQUEST => $method,
            CURLOPT_HTTPHEADER => $headers,
        ]);

        if ($body) {
            curl_setopt($ch, CURLOPT_POSTFIELDS, $body);
        } elseif (!empty($data)) {
            curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($data));
        }

        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);

        return [
            'status' => $httpCode,
            'data' => json_decode($response, true),
        ];
    }
}
```

---

## Post-Approval Management

### Version Updates

```php
<?php
/**
 * Updating an approved module
 */

// 1. Create new version locally
// 2. Update version number in module.xml
// 3. Update CHANGELOG.md
// 4. Test thoroughly
// 5. Submit via API or marketplace dashboard

$submission = new MarketplaceSubmission($apiKey);

// Submit update
$submission->submit([
    'version' => '1.1.0',
    'changelog' => <<<TEXT
## Version 1.1.0

### Added
- New feature X
- Support for Y

### Fixed
- Issue with Z
TEXT,
    'minimum_whmcs_version' => '8.0',
    'minimum_php_version' => '7.4',
]);
```

### Managing Listings

```php
<?php
/**
 * Listing management operations
 */

// Update pricing
$submission->update($moduleId, [
    'price' => 29.99,
    'discount' => [
        'type' => 'percentage',
        'value' => 20,
        'expires_at' => '2026-06-30',
    ],
]);

// Update screenshots
$submission->update($moduleId, [
    'screenshots' => [
        ['url' => 'https://cdn.example.com/screenshot1.png', 'order' => 1],
        ['url' => 'https://cdn.example.com/screenshot2.png', 'order' => 2],
    ],
]);

// Enable/disable listing
$submission->update($moduleId, [
    'status' => 'inactive', // or 'active'
]);
```

---

## Marketing Your Module

### Module Listing Optimization

```php
<?php
/**
 * Tips for better marketplace visibility
 */

// 1. Clear, descriptive title
// "Stripe Payment Gateway for WHMCS"
// Not: "Stripe Gateway Module"

// 2. Compelling tagline
$tagline = "Accept payments worldwide with zero setup fees";

// 3. Detailed feature list
$features = [
    'Accept Visa, Mastercard, American Express',
    'Support for 135+ currencies',
    '3D Secure 2.0 compliant',
    'Automatic payment reconciliation',
    'Instant settlement to your bank',
];

// 4. Regular updates (monthly at minimum)
// 5. Responsive support (within 48 hours)
// 6. Positive reviews (encourage satisfied customers)
```

---

## Security Requirements

### Required Security Measures

```php
<?php
/**
 * Security requirements for marketplace modules
 */

// 1. CSRF Protection
if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    check_token('WHMCS.default');
}

// 2. Input Validation
$input = filter_input(INPUT_POST, 'id', FILTER_VALIDATE_INT);
$email = filter_var($email, FILTER_VALIDATE_EMAIL);

// 3. Output Escaping
echo htmlspecialchars($userInput, ENT_QUOTES, 'UTF-8');

// 4. SQL Injection Prevention
$stmt = Capsule::connection()->getPdo()
    ->prepare('SELECT * FROM table WHERE id = ?');
$stmt->execute([$id]);

// 5. Secure Password Handling
$hash = password_hash($password, PASSWORD_ARGON2ID);

// 6. File Upload Security
$allowedTypes = ['image/jpeg', 'image/png', 'image/gif'];
if (!in_array($_FILES['upload']['type'], $allowedTypes)) {
    throw new \Exception('Invalid file type');
}
```

---

## Related Documentation

- [Module Packaging Guide](module-packaging-guide.md)
- [Module Release Checklist](module-release-checklist.md)
- [Module Versioning Guide](module-versioning-guide.md)
- [Module Testing Strategies](module-testing-strategies.md)
- [Security Best Practices](security-best-practices.md)
