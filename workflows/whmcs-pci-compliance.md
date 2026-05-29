# WHMCS PCI DSS Compliance Workflow

## Overview
This workflow implements PCI DSS (Payment Card Industry Data Security Standard) compliance for WHMCS.

## Prerequisites
- WHMCS with payment processing
- Compliance officer access
- Security assessment resources

## Step-by-Step Process

### Step 1: PCI DSS Compliance Checklist
```
REQUIREMENTS:
□ Build and Maintain Secure Network
  - Firewall configuration
  - Default credentials changed

□ Protect Cardholder Data
  - Data encryption in transit/rest
  - No card data storage

□ Maintain Vulnerability Management Program
  - Anti-virus software
  - Secure systems and applications

□ Implement Strong Access Controls
  - Unique user IDs
  - Restrict access by role
  - Physical security

□ Monitor and Test Networks
  - Track access to network
  - Regular testing

□ Maintain Information Security Policy
  - Security policy documentation
```

### Step 2: Card Data Handling
```php
<?php
// NEVER store full card numbers - WHMCS handles this via tokenization

// For custom payment integrations:
class SecurePaymentHandler {
    /**
     * Process payment without storing card data
     */
    public function processPayment(array $paymentData): array
    {
        // Use tokenization - never handle raw card data
        $token = $this->tokenizeCard($paymentData['token']);

        // Process with token
        return $this->chargeToken($token, $paymentData['amount']);
    }

    /**
     * Never log or store these fields:
     */
    private $forbiddenFields = [
        'cardnumber', 'cardnum', 'card_number',
        'cvv', 'cvc', 'cvv2',
        'exp_month', 'exp_year', 'expiry'
    ];
}
```

### Step 3: Network Security Configuration
```php
// Firewall rules for WHMCS
# Allow only necessary traffic
-A INPUT -p tcp -s 10.0.0.0/8 --dport 443 -j ACCEPT  # HTTPS
-A INPUT -p tcp --dport 22 -s YOUR_IP/32 -j ACCEPT   # SSH (restrict to known IPs)
-A INPUT -p tcp --dport 3306 -j DROP                   # Block MySQL external

# Rate limiting
-A INPUT -p tcp --dport 443 -m state --state NEW -m recent --set
-A INPUT -p tcp --dport 443 -m state --state NEW -m recent --update --seconds 60 --hitcount 10 -j DROP
```

### Step 4: Access Control Implementation
```php
<?php
// Implement principle of least privilege

// Admin permission check
function requirePCIPermission(string $action): void
{
    $adminId = adminId();

    $hasPermission = Capsule::table('tbladminpermissions')
        ->join('tbladminperms', 'tbladminpermissions.id', '=', 'tbladminperms.permid')
        ->where('tbladminperms.adminid', $adminId)
        ->where('tbladminpermissions.name', $action)
        ->exists();

    if (!$hasPermission) {
        throw new Exception('Insufficient permissions for PCI-required action');
    }
}

// Activity logging for PCI audit
function logPCIAccess(string $action, array $details): void
{
    Capsule::table('mod_pci_audit_log')->insert([
        'admin_id' => adminId(),
        'action' => $action,
        'details' => json_encode($details),
        'ip_address' => $_SERVER['REMOTE_ADDR'],
        'timestamp' => date('Y-m-d H:i:s')
    ]);
}
```

### Step 5: Regular Security Testing
```php
// Schedule quarterly PCI compliance checks
add_hook('QuarterlyCronJob', 1, function($vars) {
    $checks = [
        'network_scan' => runNetworkVulnerabilityScan(),
        'file_integrity' => verifyFileIntegrity(),
        'access_review' => reviewAdminAccess(),
        'log_review' => reviewPCIAuditLogs()
    ];

    $passed = array_filter($checks, fn($r) => $r['passed']);

    if (count($passed) < count($checks)) {
        sendComplianceAlert('PCI Compliance Check Failed', $checks);
    }

    return $checks;
});
```

## PCI Compliance Levels

| Level | Annual Transactions | Requirements |
|-------|-------------------|--------------|
| 1 | 6M+ | Most stringent |
| 2 | 1M - 6M | Moderate |
| 3 | 20K - 1M | Standard |
| 4 | < 20K | Basic |

## Related Workflows
- [WHMCS GDPR Compliance](./whmcs-gdpr-compliance.md)
- [WHMCS Security Scan](./whmcs-security-scan.md)