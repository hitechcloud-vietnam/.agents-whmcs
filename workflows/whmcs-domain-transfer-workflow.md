# WHMCS Domain Transfer Workflow

## Purpose

Automated and manual procedures for managing domain transfers in WHMCS, including incoming transfers, outgoing transfers, and automation of the transfer process.

## Prerequisites

- WHMCS registrar module configured
- Registrar API credentials
- Domain transfer authorization codes
- Understanding of ICANN transfer policies

## Workflow Steps

### Step 1: Configure Registrar Module

Set up domain registrar for transfers:

```php
<?php
// modules/registrars/your_registrar/your_registrar.php

class Your_Registrar_Module
{
    public function transferDomain($params)
    {
        // Required parameters
        $domain = $params['sld'] . '.' . $params['tld'];
        $transferKey = $params['transfersecret']; // Auth code
        
        // API call to initiate transfer
        $response = $this->apiCall('TransferDomain', [
            'domain' => $domain,
            'authCode' => $transferKey,
            'registrant' => [
                'firstName' => $params['firstname'],
                'lastName' => $params['lastname'],
                'email' => $params['email'],
                'address1' => $params['address1'],
                'city' => $params['city'],
                'country' => $params['country'],
                'postcode' => $params['postcode'],
                'phone' => $params['phonenumber'],
            ],
        ]);
        
        if ($response['success']) {
            return [
                'success' => true,
                'transferid' => $response['transferId'],
            ];
        }
        
        return [
            'success' => false,
            'error' => $response['error'] ?? 'Transfer failed',
        ];
    }
    
    public function getTransferStatus($params)
    {
        $domain = $params['sld'] . '.' . $params['tld'];
        
        $response = $this->apiCall('GetTransferStatus', [
            'domain' => $domain,
        ]);
        
        return [
            'status' => $this->mapTransferStatus($response['transferStatus']),
        ];
    }
    
    private function mapTransferStatus($apiStatus)
    {
        $statusMap = [
            'pending' => 'pending',
            'approved' => 'completed',
            'rejected' => 'cancelled',
            'cancelled' => 'cancelled',
            'expired' => 'failed',
        ];
        
        return $statusMap[$apiStatus] ?? 'pending';
    }
    
    private function apiCall($action, $data)
    {
        // Implement your registrar's API call
    }
}
```

### Step 2: Create Transfer Automation Hook

Automate transfer processes:

```php
<?php
// modules/custom/hooks/domain_transfer_automation.php

use WHMCS\Carbon;
use WHMCS\Domain\Transfer;

// Hook: When transfer is initiated
add_hook('DomainTransferInitiated', 1, function($params) {
    $domainId = $params['domainId'];
    $domain = $params['domain'];
    
    logActivity("Domain transfer initiated: {$domain}");
    
    // Send notification to client
    sendEmail(
        $params['clientId'],
        'Domain Transfer Initiated',
        [
            'domain' => $domain,
            'transfer_id' => $params['transferId'],
        ]
    );
    
    // Update tracking
    Capsule::table('mod_transfer_tracking')->insert([
        'domain_id' => $domainId,
        'transfer_id' => $params['transferId'],
        'initiated_at' => Carbon::now()->toDateTimeString(),
        'status' => 'initiated',
    ]);
});

// Hook: When transfer status changes
add_hook('DomainTransferStatusChanged', 1, function($params) {
    $oldStatus = $params['oldStatus'];
    $newStatus = $params['newStatus'];
    $domain = $params['domain'];
    
    logActivity("Domain transfer status changed for {$domain}: {$oldStatus} -> {$newStatus}");
    
    // Handle specific status changes
    switch ($newStatus) {
        case 'completed':
            handleTransferCompletion($params);
            break;
        case 'cancelled':
            handleTransferCancellation($params);
            break;
        case 'rejected':
            handleTransferRejection($params);
            break;
    }
});

function handleTransferCompletion($params)
{
    // Update domain record
    Capsule::table('tbldomain')
        ->where('id', $params['domainId'])
        ->update([
            'status' => 'Active',
            'expirydate' => $params['newExpiryDate'],
        ]);
    
    // Update tracking
    Capsule::table('mod_transfer_tracking')
        ->where('domain_id', $params['domainId'])
        ->update([
            'completed_at' => Carbon::now()->toDateTimeString(),
            'status' => 'completed',
        ]);
    
    // Send completion notification
    sendEmail(
        $params['clientId'],
        'Domain Transfer Completed',
        [
            'domain' => $params['domain'],
            'new_expiry_date' => $params['newExpiryDate'],
        ]
    );
}
```

### Step 3: Set Up Transfer Cron Jobs

Automate transfer monitoring:

```php
<?php
// modules/custom/transfer_monitor.php
// Run via cron every hour

require_once __DIR__ . '/init.php';

class TransferMonitor
{
    public function checkPendingTransfers()
    {
        // Get transfers stuck in pending for too long
        $stuckTransfers = Capsule::table('tbldomain')
            ->where('status', 'Pending Transfer')
            ->where('transfertime', '<', Carbon::now()->subDays(7)->toDateTimeString())
            ->get();
        
        foreach ($stuckTransfers as $domain) {
            $this->checkTransferStatus($domain);
        }
        
        // Get pending transfers to check
        $pendingTransfers = Capsule::table('tbldomain')
            ->where('status', 'Pending Transfer')
            ->where('transfertime', '>=', Carbon::now()->subDays(7)->toDateTimeString())
            ->get();
        
        foreach ($pendingTransfers as $domain) {
            $this->checkTransferStatus($domain);
        }
    }
    
    private function checkTransferStatus($domain)
    {
        $server = Capsule::table('tblservers')
            ->where('id', $domain->server)
            ->where('type', $domain->registrar)
            ->first();
        
        if (!$server) {
            return;
        }
        
        $params = [
            'domain' => $domain->domain,
            'registrar' => $domain->registrar,
            'server' => $server,
        ];
        
        $result = Registrar\Module::call($domain->registrar, 'GetTransferStatus', $params);
        
        if ($result && isset($result['status'])) {
            $this->updateDomainStatus($domain->id, $result['status'], $result);
        }
    }
    
    private function updateDomainStatus($domainId, $newStatus, $result)
    {
        $whmcsStatus = $this->mapRegistrarToWhmcsStatus($newStatus);
        
        Capsule::table('tbldomain')
            ->where('id', $domainId)
            ->update([
                'status' => $whmcsStatus,
            ]);
        
        // Log the status change
        logActivity("Domain transfer status updated via cron: ID {$domainId} -> {$whmcsStatus}");
    }
    
    private function mapRegistrarToWhmcsStatus($registrarStatus)
    {
        $mapping = [
            'pending' => 'Pending Transfer',
            'approved' => 'Active',
            'completed' => 'Active',
            'rejected' => 'Cancelled',
            'cancelled' => 'Cancelled',
            'expired' => 'Expired',
        ];
        
        return $mapping[$registrarStatus] ?? 'Pending Transfer';
    }
}

// Run the monitor
$monitor = new TransferMonitor();
$monitor->checkPendingTransfers();
```

### Step 4: Handle Outgoing Transfers

Manage domain transfers away:

```php
<?php
// modules/custom/hooks/outgoing_transfer_hooks.php

// Hook: Before domain transfer away
add_hook('DomainTransferAwayPre', 1, function($params) {
    $domain = $params['domain'];
    $domainId = $params['domainId'];
    
    // Check for outstanding issues
    $issues = [];
    
    // Check if domain is locked
    if ($params['lockstatus'] === 'locked') {
        $issues[] = 'Domain is locked. Please unlock before transfer.';
    }
    
    // Check for transfer lock (61-day rule)
    $registrationDate = Carbon::parse($params['registrationdate']);
    if ($registrationDate->diffInDays(Carbon::now()) < 61) {
        $daysRemaining = 61 - $registrationDate->diffInDays(Carbon::now());
        $issues[] = "Domain cannot be transferred for {$daysRemaining} more days.";
    }
    
    // Check for outstanding invoices
    $unpaidInvoices = Capsule::table('tblinvoiceitems')
        ->join('tblinvoices', 'tblinvoiceitems.invoiceid', '=', 'tblinvoices.id')
        ->where('tblinvoiceitems.relid', $domainId)
        ->where('tblinvoiceitems.type', 'domain')
        ->where('tblinvoices.status', '!=', 'Paid')
        ->count();
    
    if ($unpaidInvoices > 0) {
        $issues[] = "Outstanding invoices must be paid before transfer.";
    }
    
    if (!empty($issues)) {
        return [
            'abort' => true,
            'error' => implode(' ', $issues),
        ];
    }
    
    return ['abort' => false];
});

// Hook: Initiate domain transfer away
add_hook('DomainTransferAway', 1, function($params) {
    $domain = $params['domain'];
    $authCode = $params['authCode'];
    
    logActivity("Outgoing domain transfer initiated: {$domain}");
    
    // Notify admin
    sendAdminNotification(
        'Domain Transfer Away Initiated',
        [
            'domain' => $domain,
            'auth_code_provided' => !empty($authCode),
            'client_id' => $params['clientId'],
        ]
    );
});
```

### Step 5: Create Transfer API Endpoints

Expose transfer status via API:

```php
<?php
// modules/custom/api/TransferController.php

namespace WHMCS\Custom\Api;

class TransferController extends ApiHandler
{
    public function initiateTransfer()
    {
        try {
            $input = json_decode(file_get_contents('php://input'), true);
            
            // Validate input
            $required = ['domain', 'auth_code'];
            foreach ($required as $field) {
                if (empty($input[$field])) {
                    return $this->error("Missing required field: {$field}", 400)->send();
                }
            }
            
            // Get domain
            $domain = Capsule::table('tbldomain')
                ->where('domain', $input['domain'])
                ->first();
            
            if (!$domain) {
                return $this->error('Domain not found', 404)->send();
            }
            
            // Initiate transfer via registrar
            $result = localAPI('RequestDomainTransfer', [
                'domainid' => $domain->id,
                'transfersecret' => $input['auth_code'],
            ]);
            
            if ($result['result'] === 'success') {
                return $this->success([
                    'transfer_id' => $result['transferid'] ?? null,
                    'domain' => $input['domain'],
                    'status' => 'initiated',
                ], 'Transfer initiated successfully', 201)->send();
            }
            
            return $this->error($result['message'] ?? 'Transfer failed', 400)->send();
            
        } catch (\Exception $e) {
            return $this->error($e->getMessage(), 500)->send();
        }
    }
    
    public function getTransferStatus($domainId)
    {
        try {
            $domain = Capsule::table('tbldomain')
                ->where('id', $domainId)
                ->first();
            
            if (!$domain) {
                return $this->error('Domain not found', 404)->send();
            }
            
            $tracking = Capsule::table('mod_transfer_tracking')
                ->where('domain_id', $domainId)
                ->orderBy('id', 'desc')
                ->first();
            
            return $this->success([
                'domain_id' => $domain->id,
                'domain' => $domain->domain,
                'status' => $domain->status,
                'transfer_id' => $tracking->transfer_id ?? null,
                'initiated_at' => $tracking->initiated_at ?? null,
                'completed_at' => $tracking->completed_at ?? null,
            ])->send();
            
        } catch (\Exception $e) {
            return $this->error($e->getMessage(), 500)->send();
        }
    }
}
```

### Step 6: Monitor Transfer Progress

Dashboard for transfer tracking:

```php
<?php
// admin/configdomtransfers.php - Admin dashboard for transfer monitoring

require_once __DIR__ . '/../init.php';

if (!checkPermission('Configure Domain Transfer Settings', true)) {
    exit('Access Denied');
}

$action = $_GET['action'] ?? 'list';

switch ($action) {
    case 'list':
        echo $twig->render('admin/domain_transfers.html', [
            'transfers' => getTransferStats(),
            'pending' => getPendingTransfers(),
            'completed' => getCompletedTransfers(),
            'failed' => getFailedTransfers(),
        ]);
        break;
        
    case 'retry':
        $transferId = (int) $_GET['id'];
        retryFailedTransfer($transferId);
        redir('action=list&success=retry');
        break;
        
    case 'cancel':
        $transferId = (int) $_GET['id'];
        cancelTransfer($transferId);
        redir('action=list&success=cancelled');
        break;
}

function getTransferStats()
{
    return [
        'pending' => Capsule::table('tbldomain')
            ->where('status', 'Pending Transfer')
            ->count(),
        'completed_7d' => Capsule::table('tbldomain')
            ->where('status', 'Active')
            ->where('transfertime', '>=', Carbon::now()->subDays(7)->toDateTimeString())
            ->count(),
        'failed' => Capsule::table('mod_transfer_tracking')
            ->where('status', 'failed')
            ->count(),
    ];
}

function getPendingTransfers($limit = 50)
{
    return Capsule::table('tbldomain')
        ->join('tblclients', 'tbldomain.userid', '=', 'tblclients.id')
        ->where('tbldomain.status', 'Pending Transfer')
        ->select([
            'tbldomain.id',
            'tbldomain.domain',
            'tblclients.firstname',
            'tblclients.lastname',
            'tblclients.email',
            'tbldomain.transfertime',
            'tbldomain.registrar',
        ])
        ->orderBy('tbldomain.transfertime', 'desc')
        ->limit($limit)
        ->get();
}
```

## Verification Checklist

- [ ] Registrar module configured for transfers
- [ ] Transfer hooks implemented
- [ ] Cron job scheduled for monitoring
- [ ] Outgoing transfer validation working
- [ ] Transfer status tracking enabled
- [ ] Email notifications configured
- [ ] Admin dashboard accessible
- [ ] Failed transfer handling implemented
- [ ] Transfer statistics reporting active

## Related Skills and Documentation

- [WHMCS Webhook Automation](whmcs-webhook-automation-workflow.md)
- [WHMCS Client Migration](whmcs-client-migration-workflow.md)
- WHMCS Domain Documentation: https://developers.whmcs.com/domain-registrar/
- ICANN Transfer Policy: https://www.icann.org/transfers/

## Notes

- Respect ICANN's 60-day transfer lock rule
- Always verify auth codes before initiating
- Monitor transfers for potential fraud
- Send timely notifications to clients
- Keep detailed logs of all transfer activities
- Handle failed transfers with clear escalation paths
