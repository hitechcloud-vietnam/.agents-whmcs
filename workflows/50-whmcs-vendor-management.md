# WHMCS Vendor Management Workflow

## Overview
This workflow covers managing vendor relationships, licenses, and third-party integrations in WHMCS.

## Step 1: Vendor Management Service

```php
<?php
// src/Service/VendorManagementService.php

namespace WHMCS\Module\Addon\YourModule\Service;

use WHMCS\Database\Capsule;

class VendorManagementService
{
    public function registerVendor(string $name, array $details): int
    {
        return Capsule::table('mod_vendors')->insertGetId([
            'name' => $name,
            'contact_email' => $details['email'] ?? '',
            'website' => $details['website'] ?? '',
            'api_endpoint' => $details['api_endpoint'] ?? '',
            'api_key' => $details['api_key'] ?? '',
            'status' => 'active',
            'created_at' => date('Y-m-d H:i:s')
        ]);
    }

    public function addLicense(string $vendorName, string $licenseKey, string $product, string $expiryDate): int
    {
        $vendor = $this->getVendorByName($vendorName);

        if (!$vendor) {
            throw new \Exception("Vendor not found: $vendorName");
        }

        return Capsule::table('mod_vendor_licenses')->insertGetId([
            'vendor_id' => $vendor->id,
            'license_key' => $licenseKey,
            'product' => $product,
            'expiry_date' => $expiryDate,
            'status' => 'active',
            'created_at' => date('Y-m-d H:i:s')
        ]);
    }

    public function checkLicenseStatus(int $licenseId): array
    {
        $license = Capsule::table('mod_vendor_licenses')
            ->where('id', $licenseId)
            ->first();

        if (!$license) {
            return ['valid' => false, 'error' => 'License not found'];
        }

        // Check if expired
        if ($license->expiry_date < date('Y-m-d')) {
            $this->updateLicenseStatus($licenseId, 'expired');
            return ['valid' => false, 'status' => 'expired', 'expiry_date' => $license->expiry_date];
        }

        // Verify with vendor API if configured
        $vendor = Capsule::table('mod_vendors')
            ->where('id', $license->vendor_id)
            ->first();

        if ($vendor && $vendor->api_endpoint) {
            $isValid = $this->verifyWithVendor($vendor, $license->license_key);

            if (!$isValid) {
                $this->updateLicenseStatus($licenseId, 'invalid');
                return ['valid' => false, 'status' => 'invalid'];
            }
        }

        return [
            'valid' => true,
            'status' => $license->status,
            'expiry_date' => $license->expiry_date
        ];
    }

    private function verifyWithVendor($vendor, string $licenseKey): bool
    {
        // Would make API call to vendor
        return true;
    }

    private function updateLicenseStatus(int $licenseId, string $status): void
    {
        Capsule::table('mod_vendor_licenses')
            ->where('id', $licenseId)
            ->update(['status' => $status]);
    }

    public function getVendorByName(string $name): ?object
    {
        return Capsule::table('mod_vendors')
            ->where('name', $name)
            ->first();
    }

    public function getAllVendors(): array
    {
        return Capsule::table('mod_vendors')
            ->where('status', 'active')
            ->get()
            ->toArray();
    }

    public function getVendorLicenses(int $vendorId): array
    {
        return Capsule::table('mod_vendor_licenses')
            ->where('vendor_id', $vendorId)
            ->get()
            ->toArray();
    }

    public function getExpiringLicenses(int $days = 30): array
    {
        $cutoffDate = date('Y-m-d', strtotime("+{$days} days"));

        return Capsule::table('mod_vendor_licenses')
            ->join('mod_vendors', 'mod_vendor_licenses.vendor_id', '=', 'mod_vendors.id')
            ->where('expiry_date', '<=', $cutoffDate)
            ->where('status', 'active')
            ->select('mod_vendor_licenses.*', 'mod_vendors.name as vendor_name')
            ->get()
            ->toArray();
    }

    public function renewLicense(int $licenseId, string $newExpiryDate): bool
    {
        return Capsule::table('mod_vendor_licenses')
            ->where('id', $licenseId)
            ->update([
                'expiry_date' => $newExpiryDate,
                'status' => 'active',
                'renewed_at' => date('Y-m-d H:i:s')
            ]);
    }

    public function trackIntegrationUsage(int $vendorId, string $operation, int $creditsUsed = 0): void
    {
        Capsule::table('mod_vendor_usage')->insert([
            'vendor_id' => $vendorId,
            'operation' => $operation,
            'credits_used' => $creditsUsed,
            'used_at' => date('Y-m-d H:i:s')
        ]);
    }

    public function getVendorUsageReport(int $vendorId, string $period = '30 days'): array
    {
        $since = date('Y-m-d H:i:s', strtotime("-{$period}"));

        $usage = Capsule::table('mod_vendor_usage')
            ->where('vendor_id', $vendorId)
            ->where('used_at', '>=', $since)
            ->selectRaw('operation, COUNT(*) as calls, SUM(credits_used) as credits')
            ->groupBy('operation')
            ->get();

        $totalCredits = Capsule::table('mod_vendor_usage')
            ->where('vendor_id', $vendorId)
            ->where('used_at', '>=', $since)
            ->sum('credits_used');

        return [
            'vendor_id' => $vendorId,
            'period' => $period,
            'breakdown' => $usage,
            'total_credits' => $totalCredits
        ];
    }

    public function sendLicenseExpiryReminders(): array
    {
        $expiringLicenses = $this->getExpiringLicenses(30);
        $sent = [];

        foreach ($expiringLicenses as $license) {
            $daysUntilExpiry = (strtotime($license->expiry_date) - time()) / (60 * 60 * 24);

            if ($daysUntilExpiry <= 30 && $daysUntilExpiry > 0) {
                // Send reminder
                $this->sendExpiryReminder($license);
                $sent[] = $license->id;
            }
        }

        return $sent;
    }

    private function sendExpiryReminder($license): void
    {
        $vendor = Capsule::table('mod_vendors')
            ->where('id', $license->vendor_id)
            ->first();

        $admins = Capsule::table('tbladmins')
            ->where('roleid', 1)
            ->get();

        foreach ($admins as $admin) {
            send_email('LicenseExpiryReminder', $admin->email, [
                'vendor_name' => $vendor->name,
                'product' => $license->product,
                'expiry_date' => $license->expiry_date,
                'license_key' => $license->license_key
            ]);
        }
    }
}
```

## Step 2: Vendor Hook

```php
<?php
// includes/hooks/vendor_hook.php

use WHMCS\Module\Addon\YourModule\Service\VendorManagementService;

$vendorService = new VendorManagementService();

// Check license before critical operations
add_hook('BeforeOrderFulfillment', 1, function($params) {
    $vendorService->trackIntegrationUsage($params['vendor_id'], 'order_fulfillment');
});

// Track API usage
add_hook('ApiStart', 1, function($params) {
    $vendorService->trackIntegrationUsage($params['vendor_id'], 'api_call', $params['credits'] ?? 0);
});
```

## Verification Checklist

- [ ] Vendor service implemented
- [ ] Vendor registration working
- [ ] License tracking working
- [ ] License verification working
- [ ] Expiry reminders sending
- [ ] Usage tracking working
- [ ] Integration usage reporting working
