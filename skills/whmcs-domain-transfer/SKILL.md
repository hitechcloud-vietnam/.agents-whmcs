# WHMCS Domain Transfer Module Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Guide for building domain transfer/sync modules.

## When to Use

- Creating domain status synchronization modules
- Building domain portfolio management
- Implementing domain transfer automation

## Domain Transfer Patterns

```php
<?php
// modules/addons/{domaintransfer}/{domaintransfer}.php
if (!defined("WHMCS")) { die("Direct access denied"); }

function {domaintransfer}_config(): array {
    return [
        'name' => 'Domain Transfer Manager',
        'description' => 'Domain transfer and sync management',
        'version' => '1.0',
        'author' => 'Author',
        'api_provider' => ['FriendlyName' => 'API Provider', 'Type' => 'dropdown', 'Options' => 'enom,opensrs,transip,custom'],
    ];
}

function {domaintransfer}_activate(): array {
    Capsule::schema()->create('mod_domain_transfer_logs', function($t) {
        $t->increments('id');
        $t->integer('domainid')->unsigned();
        $t->string('action', 50);
        $t->string('status', 50);
        $t->text('request')->nullable();
        $t->text('response')->nullable();
        $t->timestamp('created_at');
    });

    return ['status' => 'success', 'description' => 'Domain transfer module activated'];
}

function {domaintransfer}_deactivate(): array {
    Capsule::schema()->dropIfExists('mod_domain_transfer_logs');
    return ['status' => 'success', 'description' => 'Domain transfer module deactivated'];
}

function {domaintransfer}_output(array $vars): void {
    $action = $_REQUEST['action'] ?? 'dashboard';

    echo '<div class="domain-transfer-module">';
    echo '<h1>Domain Transfer Manager</h1>';
    echo '</div>';

    switch ($action) {
        case 'pending':
            showPendingTransfers();
            break;
        case 'completed':
            showCompletedTransfers();
            break;
        case 'failed':
            showFailedTransfers();
            break;
        default:
            showDashboard();
    }
}

// Transfer domain to WHMCS
function initiateDomainTransfer(array $params): array {
    $domain = $params['domain'];
    $authCode = $params['auth_code'] ?? '';

    // Log the transfer request
    logTransferAction($params['domainid'], 'initiate', 'pending', $params, []);

    $domainData = Capsule::table('tbldomains')
        ->where('id', $params['domainid'])
        ->first();

    $registrar = $domainData->registrar;
    $api = new \DomainTransfer\ApiClient($registrar, get RegistrarConfig($registrar));

    try {
        $result = $api->transferDomain($domain, $authCode);

        logTransferAction(
            $params['domainid'],
            'initiate',
            'pending_approval',
            $params,
            $result
        );

        return [
            'success' => true,
            'transfer_id' => $result['transfer_id'],
            'status' => 'pending_approval',
        ];

    } catch (\Exception $e) {
        logTransferAction(
            $params['domainid'],
            'initiate',
            'failed',
            $params,
            ['error' => $e->getMessage()]
        );

        return [
            'success' => false,
            'error' => $e->getMessage(),
        ];
    }
}

// Check transfer status
function checkDomainTransferStatus(int $domainId): array {
    $domain = Capsule::table('tbldomains')
        ->where('id', $domainId)
        ->first();

    $api = new \DomainTransfer\ApiClient($domain->registrar, get RegistrarConfig($domain->registrar));

    try {
        $status = $api->checkTransferStatus($domain->domain);

        logTransferAction(
            $domainId,
            'status_check',
            $status['status'],
            [],
            $status
        );

        // Update domain status in WHMCS
        if ($status['status'] === 'completed') {
            Capsule::table('tbldomains')
                ->where('id', $domainId)
                ->update([
                    'status' => 'Active',
                    'expirydate' => $status['expiry_date'],
                ]);
        }

        return $status;

    } catch (\Exception $e) {
        return [
            'success' => false,
            'error' => $e->getMessage(),
        ];
    }
}

// Cancel transfer
function cancelDomainTransfer(int $domainId): array {
=======

    $domain = Capsule::table('tbldomains')
        ->where('id', $domainId)
        ->first();

    $api = new \DomainTransfer\ApiClient($domain->registrar, get RegistrarConfig($domain->registrar));

    try {
        $api->cancelTransfer($domain->domain);

        logTransferAction($domainId, 'cancel', 'cancelled', [], []);

        Capsule::table('tbldomains')
            ->where('id', $domainId)
            ->update(['status' => 'Cancelled']);

        return ['success' => true];

    } catch (\Exception $e) {
        return [
            'success' => false,
            'error' => $e->getMessage(),
        ];
    }
}

function resendTransferApproval(int $domainId): array {
    $domain = Capsule::table('tbldomains')
        ->where('id', $domainId)
        ->first();

    $api = new \DomainTransfer\ApiClient($domain->registrar, get RegistrarConfig($domain->registrar));

    try {
        $api->resendApprovalEmail($domain->domain);

        logTransferAction($domainId, 'resend_approval', 'pending', [], []);

        return ['success' => true];

    } catch (\Exception $e) {
        return [
            'success' => false,
            'error' => $e->getMessage(),
        ];
    }
}

// Sync all transfers with registrar
function {domaintransfer}_syncAllTransfers(): array {
    $pendingTransfers = Capsule::table('tbldomains')
        ->whereIn('status', ['Pending Transfer', 'Pending'])
        ->where('registrar', '!=', '')
        ->get();

    $results = [
        'total' => count($pendingTransfers),
        'synced' => 0,
        'failed' => 0,
        'details' => [],
    ];

    foreach ($pendingTransfers as $domain) {
        $status = checkDomainTransferStatus($domain->id);

        if ($status['success']) {
            $results['synced']++;
        } else {
            $results['failed']++;
        }

        $results['details'][] = [
            'domain' => $domain->domain,
            'result' => $status,
        ];
    }

    return $results;
}

// Cron job for automatic sync
function {domaintransfer}_cron(): void {
    $result = {domaintransfer}_syncAllTransfers();

    if ($result['failed'] > 0) {
        logActivity("Domain Transfer Sync: {$result['failed']} failed out of {$result['total']}");
    }
}

function logTransferAction(int $domainId, string $action, string $status, array $request, array $response): void {
    Capsule::table('mod_domain_transfer_logs')->insert([
        'domainid' => $domainId,
        'action' => $action,
        'status' => $status,
        'request' => json_encode($request),
        'response' => json_encode($response),
        'created_at' => date('Y-m-d H:i:s'),
    ]);
}

function getRegistrarConfig(string $registrar): array {
    return Capsule::table('tblregistrars')
        ->where('registrar', $registrar)
        ->pluck('value', 'setting')
        ->toArray();
}

// Admin dashboard views
function showDashboard(): void {
    $stats = [
        'pending' => Capsule::table('tbldomains')->where('status', 'Pending Transfer')->count(),
        'active' => Capsule::table('tbldomains')->where('status', 'Active')->count(),
        'failed' => Capsule::table('mod_domain_transfer_logs')->where('status', 'failed')->count(),
    ];

    include __DIR__ . '/templates/admin/dashboard.tpl';
}
```

### Admin Dashboard Template

```smarty
<div class="domain-transfer-dashboard">
    <div class="stats-row">
        <div class="stat-box pending">
            <div class="stat-value">{$stats.pending}</div>
            <div class="stat-label">Pending Transfers</div>
        </div>
        <div class="stat-box active">
            <div class="stat-value">{$stats.active}</div>
            <div class="stat-label">Active Domains</div>
        </div>
        <div class="stat-box failed">
            <div class="stat-value">{$stats.failed}</div>
            <div class="stat-label">Failed</div>
        </div>
    </div>

    <div class="actions">
        <a href="?m={module}&action=sync_all" class="btn btn-primary">
            Sync All Transfers
        </a>
        <a href="?m={module}&action=pending" class="btn">
            View Pending ({$stats.pending})
        </a>
    </div>

    <h3>Recent Activity</h3>
    <table class="data-table">
        <thead>
            <tr>
                <th>Domain</th>
                <th>Action</th>
                <th>Status</th>
                <th>Date</th>
                <th>Actions</th>
            </tr>
        </thead>
        <tbody>
            {foreach $logs as $log}
            <tr>
                <td>{$log->domain}</td>
                <td>{$log->action}</td>
                <td><span class="badge badge-{$log->status}">{$log->status}</span></td>
                <td>{$log->created_at|date_format:'%Y-%m-d H:i'}</td>
                <td>
                    <a href="?m={module}&action=retry&id={$log->domainid}">Retry</a>
                </td>
            </tr>
            {/foreach}
        </tbody>
    </table>
</div>
```

---

**Related Skills:**
- whmcs-domain-sync
- whmcs-registrar-builder
- whmcs-cron-automation
