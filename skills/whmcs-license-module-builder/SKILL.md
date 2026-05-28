# WHMCS License Module Builder Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Guide for building software license provisioning modules.

## When to Use

- Creating license key management systems
- Building software provisioning modules
- Implementing license validation

## License Module Pattern

```php
<?php
// modules/servers/{licensemodule}/{licensemodule}.php
if (!defined("WHMCS")) { die("Direct access denied"); }

function {module}_MetaData(): array {
    return [
        'DisplayName' => 'License Provider',
        'APIVersion' => '1.0',
        'RequiresServer' => true,
    ];
}

function {module}_CreateAccount(array $params): string {
    try {
        $api = new \License\ApiClient($params);

        // Generate or retrieve license
        $license = $api->createLicense([
            'product' => $params['configoption1'],
            'domain' => $params['customfields']['domain'] ?? $params['domain'],
            'quantity' => $params['configoption2'] ?? 1,
        ]);

        saveCustomFieldValue($params['serviceid'], 'license_key', $license['key']);
        saveCustomFieldValue($params['serviceid'], 'license_id', $license['id']);

        return 'success';
    } catch (\Exception $e) {
        return 'Error: ' . $e->getMessage();
    }
}

function {module}_SuspendAccount(array $params): string {
    $licenseId = getCustomFieldValue($params['serviceid'], 'license_id');

    try {
        $api = new \License\ApiClient($params);
        $api->suspendLicense($licenseId);

        return 'success';
    } catch (\Exception $e) {
        return 'Error: ' . $e->getMessage();
    }
}

function {module}_TerminateAccount(array $params): string {
    $licenseId = getCustomFieldValue($params['serviceid'], 'license_id');

    try {
        $api = new \License\ApiClient($params);
        $api->revokeLicense($licenseId);

        return 'success';
    } catch (\Exception $e) {
        return 'Error: ' . $e->getMessage();
    }
}

function {module}_ClientArea(array $params): array {
    $licenseKey = getCustomFieldValue($params['serviceid'], 'license_key');

    return [
        'pagetitle' => 'License Details',
        'templatefile' => 'license',
        'vars' => [
            'license_key' => $licenseKey,
            'copy_code' => 'Copy License Key',
        ],
    ];
}
```

---

**Related Skills:**
- whmcs-server-builder
- whmcs-clientarea-builder
- whmcs-license-validation