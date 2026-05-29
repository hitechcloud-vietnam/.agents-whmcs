# WHMCS Transfer Tool Documentation

## Overview

The WHMCS Transfer Tool provides comprehensive domain transfer management, including incoming transfers, outgoing transfers, and transfer status tracking.

## Transfer Types

### Incoming Transfers

Domains transferred TO your registrar/reseller from another registrar.

**Key Steps:**
1. Customer initiates transfer with auth code (EPP key)
2. WHMCS validates transfer request
3. Send transfer request to registry
4. Approve/reject at current registrar
5. Complete transfer at new registrar

### Outgoing Transfers

Domains transferred FROM your registrar to another registrar.

**Key Steps:**
1. Customer obtains auth code from current registrar
2. Customer submits to new registrar
3. Current registrar receives transfer request
4. Approve/reject transfer
5. Transfer completes automatically

### Internal Transfers

Domain ownership transfer between accounts within WHMCS.

## Configuration

### Transfer Settings

Navigate to **Setup > General Settings > Domains Tab**

```php
// Transfer Configuration
$domainTransferSettings = [
    // Transfer Lock
    'autoLockOnTransfer' => true,      // Lock domain during transfer
    'requireAuthCode' => true,          // Require EPP code verification
    'authCodeLength' => 12,             // Minimum auth code length

    // Transfer Notifications
    'notifyOnIncoming' => true,         // Email admin on incoming
    'notifyOnOutgoing' => true,         // Email admin on outgoing
    'notifyOnComplete' => true,         // Email on transfer completion

    // Transfer Windows
    'renewOnTransfer' => true,          // Add year on transfer
    'renewalPeriod' => 1,               // Years to add (1-10)
    'transferGracePeriod' => 5,         // Days before transfer allowed

    // Approval Settings
    'autoApproveIncoming' => false,      // Auto-approve incoming
    'autoApproveInternal' => true,      // Auto-approve internal
    'approvalTimeout' => 5,             // Days before auto-approve
];
```

### Registrar Transfer Policies

Configure per-registrar transfer policies:

```php
// Registrar-specific transfer settings
$registrarPolicies = [
    'enom' => [
        'requireFoA' => false,           // Require Fax of Authorization
        'quickTransfer' => true,         // Enable expedited transfers
        'regenerateAuthCode' => true,     // Allow auth code regeneration
        'transferLockDays' => 0           // Days to keep domain locked
    ],
    'godaddy' => [
        'requireFoA' => true,
        'urgentTransfer' => true,
        'approvalsRequired' => true
    ]
];
```

## Transfer Workflow

### Incoming Transfer Process

```
Customer Request → Validation → Registry Request → Approval → Completion
     ↓                ↓              ↓                ↓            ↓
  Auth Code      Domain Status    Transfer Sent    5-Day Window   Domain Active
  Verified       Checked          to Registry      or Approved    in WHMCS
```

### Step-by-Step Flow

1. **Customer submits transfer request**
   - Enter domain name
   - Provide auth code
   - Select registration period

2. **WHMCS validates request**
   - Verify domain exists
   - Check domain is not locked
   - Validate auth code format
   - Confirm domain not in transfer window

3. **Send transfer to registry**
   - Format EPP transfer command
   - Send via registrar API
   - Log transaction

4. **Registry processes transfer**
   - Notify losing registrar
   - Start 5-day transfer window
   - Accept/reject via whois

5. **Transfer completion**
   - Update domain status
   - Change nameservers if needed
   - Send confirmation emails
   - Update billing

### Outgoing Transfer Process

1. **Customer requests auth code**
2. **Admin verifies eligibility**
3. **Generate/send auth code**
4. **Unlock domain if locked**
5. **Log outgoing transfer request**
6. **Track completion status**

## API Integration

### Initiate Transfer

```php
use WHMCS\Domain\Transfer;

// Create new transfer request
$transfer = new Transfer();
$result = $transfer->initiate([
    'domain' => 'example.com',
    'type' => 'incoming',
    'auth_code' => 'domainAuthCode123',
    'registration_period' => 2,
    'registrant' => [
        'firstname' => 'John',
        'lastname' => 'Doe',
        'email' => 'john@example.com'
    ],
    'nameservers' => [
        'ns1.yourdomain.com',
        'ns2.yourdomain.com'
    ]
]);

if ($result['success']) {
    $transferId = $result['transfer_id'];
    $status = $result['status'];
    $estimatedCompletion = $result['eta'];
}
```

### Check Transfer Status

```php
use WHMCS\Domain\Transfer;

// Get transfer status
$status = Transfer::checkStatus($transferId);

// Response
[
    'transfer_id' => 'TRF-12345',
    'domain' => 'example.com',
    'status' => 'pending_approval',
    'initiated_at' => '2024-01-15T10:30:00Z',
    'approval_deadline' => '2024-01-20T10:30:00Z',
    'current_registrar' => 'Losing Registrar Inc.',
    'losing_registrar_status' => 'pending',
    'messages' => [
        'Registry has sent approval request to current registrar',
        'Waiting for response (4 days remaining)'
    ]
]
```

### Cancel Transfer

```php
// Cancel pending transfer
$result = Transfer::cancel($transferId, [
    'reason' => 'Customer requested cancellation',
    'notify_registrar' => true
]);
```

### Transfer Response Codes

| Code | Status | Description |
|------|--------|-------------|
| 100 | SUCCESS | Transfer completed successfully |
| 101 | PENDING | Transfer in progress |
| 102 | APPROVED | Transfer approved by current registrar |
| 200 | INVALID_CODE | Auth code invalid or expired |
| 201 | DOMAIN_LOCKED | Domain is locked |
| 202 | IN_TRANSFER_WINDOW | Domain recently transferred |
| 203 | NOT_ELIGIBLE | TLD not eligible for transfer |
| 204 | PENDING_RENEWAL | Domain has pending renewal |
| 300 | REGISTRY_ERROR | Registry rejected transfer |
| 400 | TIMEOUT | Transfer window expired |

## Customer Portal

### Transfer Request Form

**Client Area > Orders > Transfer Domain**

Form fields:
- Domain name (auto-complete)
- Auth code (hidden input with reveal)
- Registration period dropdown
- Nameserver selection
- Add-ons (privacy, email forwarding)
- Terms acceptance checkbox

### Transfer Status Tracking

**Client Area > My Domains > Transfer Status**

Display:
- Domain name
- Current status with icon
- Progress indicator
- Estimated completion date
- Action buttons (cancel, resend auth code)
- Timeline of events

### Transfer Notifications

Email templates for customers:

| Template | Trigger |
|----------|---------|
| Domain Transfer Initiated | Transfer submitted |
| Domain Transfer Approved | Current registrar approved |
| Domain Transfer Rejected | Transfer denied |
| Domain Transfer Completed | Successfully transferred |
| Domain Transfer Cancelled | Customer cancelled |

## Transfer Pricing

### Pricing Configuration

Navigate to **Setup > Products/Services > Domain Pricing**

| TLD | Transfer Price | Renew Price | Notes |
|-----|---------------|-------------|-------|
| .com | $9.99 | $12.99 | Includes 1 year renewal |
| .net | $10.99 | $13.99 | Includes 1 year renewal |
| .org | $10.99 | $13.99 | Includes 1 year renewal |
| .io | $35.99 | $39.99 | Premium TLD |

### Transfer vs. Re-Registration

| Aspect | Transfer | Re-Registration |
|--------|----------|-----------------|
| Price | Transfer fee + renewal | Registration fee |
| Renewal Period | Typically 1 year added | 1-10 years |
| Downtime | Minimal (hours) | None (register fresh) |
| DNS | Preserved | Must configure |
| History | Preserved | Fresh registration |

## Troubleshooting

### Common Transfer Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| Transfer pending forever | Registrar not responding | Cancel and retry |
| Invalid auth code | Code expired/wrong | Request new code |
| Domain locked | Registrar lock enabled | Unlock domain first |
| Transfer rejected | Registrar policy | Contact current registrar |
| 60-day rule triggered | Recent transfer/registration | Wait for 60-day window |

### Debug Commands

```bash
# List pending transfers
whmcscli transfer list --status=pending

# Check transfer status
whmcscli transfer status --domain=example.com

# Retry failed transfer
whmcscli transfer retry --transfer-id=TRF-12345

# Cancel transfer
whmcscli transfer cancel --transfer-id=TRF-12345

# Force complete (admin)
whmcscli transfer complete --transfer-id=TRF-12345
```

### Transfer Logs

View transfer activity:

```
/whmcs/logs/transfers.log
```

Sample log entry:
```
[2024-01-15 10:30:00] [INFO] Transfer initiated: example.com
[2024-01-15 10:30:05] [INFO] Registry response: 100 (Success)
[2024-01-15 10:30:05] [INFO] Transfer pending approval
[2024-01-20 10:30:00] [WARN] Approval deadline reached
[2024-01-20 10:35:00] [INFO] Auto-approved by registry
[2024-01-20 10:35:02] [INFO] Transfer completed
```

## Bulk Transfers

### Bulk Transfer Import

```php
// Process bulk transfer file
$bulkTransfer = WHMCS\Domain\Transfer\BulkImport::factory('csv');

// CSV format:
// domain,auth_code,period,nameserver1,nameserver2
// example.com,code123,1,ns1.com,ns2.com

$result = $bulkTransfer->process([
    'skip_existing' => true,
    'notify_on_complete' => true,
    'parallel_transfers' => 5
]);
```

### Bulk Transfer Report

```php
$report = $bulkTransfer->generateReport();

// Output
[
    'total' => 100,
    'success' => 85,
    'failed' => 15,
    'pending' => 0,
    'results' => [
        ['domain' => 'example.com', 'status' => 'completed'],
        ['domain' => 'test.net', 'status' => 'failed', 'error' => 'Invalid auth code'],
        // ...
    ]
];
```

## ICANN Transfer Policy

### Required Elements

Per ICANN policy:

1. **AuthInfo Code** - Mandatory verification
2. **FOA (Fax of Authorization)** - Some registrars require
3. **Email to Registrant** - Confirmation required
4. **5-Day Window** - Minimum transfer time
5. **60-Day Rule** - No transfer within 60 days of registration or previous transfer

### Compliance Check

```php
// Verify ICANN compliance
$compliance = WHMCS\Domain\Transfer\Compliance::check([
    'domain' => 'example.com',
    'requested_at' => date('Y-m-d H:i:s')
]);

// Response
[
    'compliant' => true,
    'checks' => [
        'auth_code_present' => true,
        'registrant_verified' => true,
        'not_within_60_days' => true,
        'not_pending_restore' => true,
        'not_in_registrar_lock' => true
    ]
];
```

## See Also

- [Auto Registration](./whmcs-auto-registration.md)
- [Sync Daemon](./whmcs-sync-daemon.md)
- [Registrar Commands](./whmcs-registrar-commands.md)
