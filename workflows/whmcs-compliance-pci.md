# WHMCS PCI Compliance Workflow

## Description
Achieve and maintain PCI DSS compliance for WHMCS payment processing.

## Prerequisites
- Payment gateway integration
- SSL certificate
- Security awareness

## Steps

### Step 1: Understand PCI DSS Requirements
```markdown
PCI DSS has 12 requirements organized into 6 goals:

Build and Maintain Secure Network:
1. Install and maintain firewall
2. Do not use vendor-supplied defaults

Protect Cardholder Data:
3. Protect stored cardholder data
4. Encrypt transmission of cardholder data

Maintain Vulnerability Management Program:
5. Use and update anti-virus software
6. Develop and maintain secure systems

Implement Strong Access Control Measures:
7. Restrict access by business need-to-know
8. Assign unique ID to each person
9. Restrict physical access

Regular Monitoring and Testing:
10. Track and monitor all access
11. Regularly test systems

Maintain Information Security Policy:
12. Maintain information security policy
```

### Step 2: Self-Assessment Questionnaire (SAQ)
```markdown
SAQ Types:
- SAQ A: Card-not-present merchants (no card data storage)
- SAQ A-EP: E-commerce merchants
- SAQ B: Dial-in terminals
- SAQ C-VT: Virtual terminal
- SAQ C: Web-based applications
- SAQ D: All other merchants

For WHMCS:
Most users qualify for SAQ A or SAQ A-EP
```

### Step 3: Compliance Checklist
```markdown
Technical Requirements:
[ ] SSL/TLS 1.2+ for all payment pages
[ ] No card data stored in WHMCS database
[ ] Payment gateway handles card processing
[ ] Strong passwords for admin accounts
[ ] Two-factor authentication enabled

Network Security:
[ ] Firewall configured
[ ] No direct database access from internet
[ ] Regular security updates
[ ] Intrusion detection in place

Access Control:
[ ] Unique admin accounts
[ ] Role-based permissions
[ ] Regular access reviews
[ ] Session timeouts configured

Monitoring:
[ ] Access logging enabled
[ ] Log retention (90+ days)
[ ] Regular log reviews
[ ] Incident response plan
```

### Step 4: Configure WHMCS for PCI Compliance
```php
<?php
// configuration.php security settings

// Disable card storage in WHMCS
$disable_local_card_storage = true;

// Use gateway tokenization
$use_tokenization = true;

// Enable admin security
$admin_2fa_required = true;

// Session security
$session_secure_cookie = true;
$session_httponly = true;
```

### Step 5: Security Hardening
```bash
# File permissions
chmod 644 /var/www/whmcs/configuration.php
chmod 755 /var/www/whmcs/modules
chmod 644 /var/www/whmcs/modules/gateways/*/*.php

# Directory permissions
chmod 755 /var/www/whmcs/templates_c
chmod 755 /var/www/whmcs/attachments
chmod 755 /var/www/whmcs/downloads

# Disable unused PHP functions
sed -i 's/disable_functions = /disable_functions = exec,passthru,shell_exec,system,/' /etc/php/*/fpm/php.ini
```

### Step 6: Regular Compliance Tasks
```php
<?php
// Quarterly compliance tasks

// 1. Review admin accounts
function reviewAdminAccounts()
{
    $admins = Capsule::table('tbladmins')->get();
    foreach ($admins as $admin) {
        if (!$admin->twofa_enabled) {
            // Flag for 2FA enforcement
        }
    }
}

// 2. Review access logs
function reviewAccessLogs($days = 90)
{
    $cutoff = date('Y-m-d', strtotime("-{$days} days"));
    
    // Look for suspicious patterns
    $suspicious = Capsule::table('tblactivitylog')
        ->where('date', '>=', $cutoff)
        ->where(function($q) {
            $q->where('description', 'LIKE', '%failed login%')
              ->orWhere('description', 'LIKE', '%invalid password%');
        })
        ->get();
    
    return $suspicious;
}

// 3. Review system changes
function reviewSystemChanges()
{
    // Check for unauthorized configuration changes
    // Review module installations
    // Check file integrity
}
```

### Step 7: Compliance Documentation
```markdown
Documentation Required:
1. Network diagram showing card data flow
2. Data flow diagram for cardholder information
3. Security policies and procedures
4. Vendor list with PCI compliance status
5. Incident response procedures
6. Employee security awareness training records
7. Quarterly security scan results
8. Annual risk assessment
```

## Annual Compliance Schedule
| Month | Task |
|-------|------|
| Jan | Annual risk assessment |
| Mar | Q1 security scan |
| Jun | Q2 security scan |
| Jul | Policy review and update |
| Sep | Q3 security scan |
| Oct | Prepare for annual assessment |
| Dec | Annual assessment completion |

## Tags
- pci-compliance
- security
- payment
- compliance