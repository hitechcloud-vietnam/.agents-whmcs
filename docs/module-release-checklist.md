# WHMCS Module Release Checklist

**Version:** 8.x
**Updated:** 2026-05-28
**Related Skills:** `hooks-reference`, `database-schema-design`, `security-best-practices`

## Overview

This checklist ensures your WHMCS module meets quality, security, and compatibility standards before release. Use this guide when developing and releasing third-party modules.

## Pre-Development Phase

### Requirements Gathering

- [ ] Define module purpose and scope
- [ ] Identify supported WHMCS versions (8.x, 7.x)
- [ ] List required WHMCS API functions
- [ ] Document dependencies and requirements
- [ ] Create feature specification document

### Version Compatibility Matrix

```php
/**
 * Module Compatibility
 *
 * WHMCS Version Support:
 * - 8.0.x (minimum)
 * - 8.1.x
 * - 8.2.x (latest tested)
 *
 * PHP Version Support:
 * - 7.4 (minimum)
 * - 8.0
 * - 8.1
 * - 8.2
 *
 * MySQL Version Support:
 * - 5.7+
 * - 8.0+
 */
```

## Development Phase

### Core Module Structure

```php
<?php
/**
 * Module Name
 *
 * @package WHMCS\Module\Server\ModuleName
 * @copyright Copyright (c) 2026 Your Company
 * @license https://whmcs.com/license.html
 */

if (!defined('WHMCS')) {
    die('Direct access not permitted');
}

/**
 * Module Metadata
 */
function ModuleName_MetaData(): array
{
    return [
        'DisplayName' => 'Module Name',
        'APIVersion' => '1.1',
        'RequiresServer' => true,
        'ServiceSingleSignOn' => true,
        'DefaultNonSSLPort' => 80,
        'DefaultSSLPort' => 443,
    ];
}

/**
 * Module Configuration
 */
function ModuleName_ConfigOptions(array $params): array
{
    return [
        'api_key' => [
            'Type' => 'text',
            'Size' => '64',
            'Default' => '',
            'Description' => 'API Key from your provider',
        ],
        'api_secret' => [
            'Type' => 'password',
            'Size' => '64',
            'Default' => '',
            'Description' => 'API Secret',
        ],
        'environment' => [
            'Type' => 'dropdown',
            'Options' => 'Production,Staging,Development',
            'Default' => 'Production',
        ],
    ];
}

/**
 * Account Creation
 */
function ModuleName_CreateAccount(array $params): array
{
    try {
        // Implementation
        logModuleCall('ModuleName', 'CreateAccount', $params, $result);

        return [
            'success' => true,
            'accountid' => $result['serveraccountid'],
        ];
    } catch (\Exception $e) {
        logModuleCall('ModuleName', 'CreateAccount', $params, $e->getMessage(), [], true);
        return [
            'success' => false,
            'error' => $e->getMessage(),
        ];
    }
}

// ... Implement other required functions
```

### Required Functions by Module Type

#### Server Module (Provisioning)

| Function | Required | Description |
|----------|----------|-------------|
| MetaData | Yes | Module information |
| ConfigOptions | Yes | Admin configuration |
| CreateAccount | Yes | Provision new service |
| SuspendAccount | Yes | Suspend service |
| UnsuspendAccount | Yes | Reactivate service |
| TerminateAccount | Yes | Cancel/terminate service |
| RenewAccount | No | Handle renewal |
| ChangePackage | No | Change service plan |
| ChangePassword | No | Change account password |
| ChangeUsername | No | Change username |
| GetRemoteRenewalPrice | No | Get renewal pricing |
| GetUsage | No | Get usage statistics |
| TestConnection | Yes | Test API connectivity |

#### Registrar Module

| Function | Required | Description |
|----------|----------|-------------|
| MetaData | Yes | Module information |
| GetRegistrarParameters | Yes | Config parameters |
| RegisterDomain | Yes | Register new domain |
| TransferDomain | Yes | Initiate transfer |
| RenewDomain | Yes | Renew domain |
| GetDomainNameservers | Yes | Get current NS |
| SaveDomainNameservers | Yes | Update NS |
| GetDomainContact | Yes | Get contact info |
| SaveDomainContact | Yes | Update contact |
| GetEPPCode | Yes | Get auth code |
| RegisterNameserver | Yes | Create NS record |
| ModifyNameserver | Yes | Update NS record |
| DeleteNameserver | Yes | Remove NS record |
| TransferSync | No | Sync transfer status |

#### Gateway Module

| Function | Required | Description |
|----------|----------|-------------|
| MetaData | Yes | Module information |
| GetPaymentParameters | Yes | Config parameters |
| Link | Yes | Generate payment link |
| refund | No | Process refunds |

### Logging Implementation

```php
/**
 * Proper module logging
 */
function ModuleName_CreateAccount(array $params): array
{
    $action = 'CreateAccount';
    $successData = null;
    $errorMessage = null;

    try {
        // Validate input
        if (empty($params['username'])) {
            throw new \Exception('Username is required');
        }

        // Prepare sensitive data masking
        $sensitiveParams = $params;
        $sensitiveParams['password'] = '***';
        $sensitiveParams['api_key'] = '***';

        logModuleCall(
            'ModuleName',
            $action,
            $sensitiveParams,
            null, // Will be set on success
            null, // Will be set on error
            true  // Replace variables in log
        );

        // API call
        $result = $this->api->createAccount([
            'username' => $params['username'],
            'password' => $params['password'],
            'package' => $params['configoption1'],
        ]);

        // Log success
        logModuleCall(
            'ModuleName',
            $action,
            $sensitiveParams,
            $result
        );

        return [
            'success' => true,
            'accountid' => $result['account_id'],
        ];

    } catch (\Exception $e) {
        // Log error
        logModuleCall(
            'ModuleName',
            $action,
            $params,
            null,
            $e->getMessage(),
            true
        );

        return [
            'success' => false,
            'error' => $e->getMessage(),
        ];
    }
}
```

## Testing Phase

### Unit Testing

```php
<?php
// tests/ModuleNameTest.php

namespace WHMCS\Module\Server\ModuleName\Tests;

use PHPUnit\Framework\TestCase;

class ModuleNameTest extends TestCase
{
    protected $module;

    protected function setUp(): void
    {
        parent::setUp();
        require_once __DIR__ . '/../ModuleName.php';
        $this->module = new ModuleName();
    }

    public function testMetaDataReturnsArray(): void
    {
        $metaData = ModuleName_MetaData();

        $this->assertIsArray($metaData);
        $this->assertArrayHasKey('DisplayName', $metaData);
        $this->assertArrayHasKey('APIVersion', $metaData);
    }

    public function testConfigOptionsReturnsArray(): void
    {
        $config = ModuleName_ConfigOptions([]);

        $this->assertIsArray($config);
        $this->assertArrayHasKey('api_key', $config);
        $this->assertArrayHasKey('api_secret', $config);
    }

    public function testCreateAccountValidatesParams(): void
    {
        $result = ModuleName_CreateAccount([
            'username' => '',
        ]);

        $this->assertFalse($result['success']);
        $this->assertArrayHasKey('error', $result);
    }

    public function testCreateAccountSucceeds(): void
    {
        $params = [
            'server' => 'server1',
            'serverusername' => 'api_user',
            'serverpassword' => 'api_pass',
            'username' => 'testuser_' . time(),
            'password' => 'TestPass123!',
            'configoption1' => 'basic',
        ];

        $result = ModuleName_CreateAccount($params);

        // May fail if no API connection - that's OK for unit tests
        $this->assertArrayHasKey('success', $result);
    }
}
```

### Integration Testing Checklist

- [ ] Test on WHMCS 8.0.x
- [ ] Test on WHMCS 8.1.x
- [ ] Test on WHMCS 8.2.x (latest)
- [ ] Test with PHP 7.4
- [ ] Test with PHP 8.0
- [ ] Test with PHP 8.1
- [ ] Test with PHP 8.2
- [ ] Test database migrations
- [ ] Test upgrade from previous version
- [ ] Test fresh installation
- [ ] Test uninstallation (cleanup)

### Manual Testing Scenarios

```markdown
## Test Scenarios

### Server Module Testing

1. **New Order Provisioning**
   - [ ] Order placed with new client
   - [ ] Module CreateAccount called
   - [ ] Service created in remote system
   - [ ] Service activated in WHMCS
   - [ ] Welcome email sent

2. **Service Suspension**
   - [ ] Admin clicks Suspend
   - [ ] Module SuspendAccount called
   - [ ] Service suspended remotely
   - [ ] Status updated in WHMCS
   - [ ] Suspension email sent

3. **Service Termination**
   - [ ] Admin clicks Terminate
   - [ ] Confirmation dialog shown
   - [ ] Module TerminateAccount called
   - [ ] Service removed remotely
   - [ ] Service deleted in WHMCS

### Error Handling Testing

1. **API Timeout**
   - [ ] Simulate API timeout
   - [ ] Verify error logged
   - [ ] Verify user-friendly message
   - [ ] Verify no partial state

2. **Invalid Credentials**
   - [ ] Use wrong API key
   - [ ] Verify connection test fails
   - [ ] Verify clear error message
```

## Security Phase

### Security Checklist

- [ ] No hardcoded credentials in code
- [ ] All passwords masked in logs
- [ ] API keys stored securely
- [ ] Input validation on all parameters
- [ ] SQL injection prevention
- [ ] XSS prevention in output
- [ ] CSRF protection implemented
- [ ] Rate limiting on API calls
- [ ] Secure API calls (HTTPS)
- [ ] No sensitive data in error messages

### Code Security Review

```php
/**
 * SECURE: Using prepared statements
 */
public function getAccount(int $accountId): array
{
    $stmt = Capsule::connection()->getPdo()->prepare(
        'SELECT * FROM tblcustom WHERE id = :id'
    );
    $stmt->execute(['id' => $accountId]);
    return $stmt->fetch(PDO::FETCH_ASSOC);
}

/**
 * SECURE: Validating input
 */
public function createAccount(array $params): array
{
    // Validate required fields
    if (empty($params['username'])) {
        throw new \InvalidArgumentException('Username is required');
    }

    // Validate username format
    if (!preg_match('/^[a-zA-Z0-9_-]{3,32}$/', $params['username'])) {
        throw new \InvalidArgumentException('Invalid username format');
    }

    // Sanitize before use
    $username = htmlspecialchars($params['username'], ENT_QUOTES, 'UTF-8');

    // Continue with creation...
}

/**
 * INSECURE: SQL Injection vulnerability - DO NOT USE
 */
public function insecureQuery(string $userId): array
{
    // NEVER do this!
    $result = Capsule::select("SELECT * FROM tblcustom WHERE id = $userId");
    return $result;
}
```

## Documentation Phase

### Required Documentation

- [ ] README.md with installation instructions
- [ ] CHANGELOG.md with version history
- [ ] LICENSE file
- [ ] Configuration guide
- [ ] API documentation (if applicable)
- [ ] Troubleshooting section
- [ ] Support contact information

### README.md Template

```markdown
# Module Name

Brief description of what the module does.

## Requirements

- WHMCS 8.0 or higher
- PHP 7.4 or higher
- [External service] account with API access

## Installation

1. Upload the module to `/modules/servers/modulename/`
2. Navigate to WHMCS Admin > System > Addon Modules
3. Activate the module
4. Configure server credentials

## Configuration

| Setting | Description |
|---------|-------------|
| API Key | Your API key from provider |
| Environment | Production/Staging/Development |

## Changelog

### Version 1.0.0 (2026-05-28)
- Initial release
- Support for WHMCS 8.x
- Basic provisioning features

## Support

For support requests, contact: support@example.com

## License

Commercial license - see LICENSE file
```

## Pre-Release Checklist

### Final Verification

- [ ] All code follows coding standards
- [ ] No debug or test code left
- [ ] No console.log or print_r left
- [ ] Version number updated
- [ ] Changelog updated
- [ ] README updated
- [ ] License file included
- [ ] Module tested on clean WHMCS install
- [ ] Module tested on existing WHMCS upgrade
- [ ] All paths use proper directory separators
- [ ] All links verified
- [ ] Module directory name is correct

### Package Contents

```
ModuleName-v1.0.0.zip/
├── README.md
├── CHANGELOG.md
├── LICENSE
├── modules/
│   └── servers/
│       └── modulename/
│           ├── modulename.php
│           ├── hooks.php (if needed)
│           └── views/
│               └── templates/
│                   └── ...
├── cli/
│   └── scripts/
│       └── ...
├── assets/
│   ├── css/
│   └── images/
└── tests/
    └── ...
```

### File Permissions

```bash
# Set correct permissions for release
find modulename -type f -exec chmod 644 {} \;
find modulename -type d -exec chmod 755 {} \;

# Verify no files have execute permission
find modulename -type f -perm /111
```

## Post-Release

### Distribution Checklist

- [ ] Create GitHub release
- [ ] Update WHMCS marketplace listing
- [ ] Announce in WHMCS community
- [ ] Notify existing customers
- [ ] Update documentation site
- [ ] Set up support channels
- [ ] Monitor for issues
- [ ] Create bug report template

## Related Documentation

- [Hooks Reference](hooks-reference.md)
- [Database Schema Design](database-schema-design.md)
- [Security Best Practices](security-best-practices.md)
