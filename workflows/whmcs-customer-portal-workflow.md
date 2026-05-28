# WHMCS Customer Portal Workflow

## Purpose

Procedures for developing and customizing WHMCS client portal functionality, including custom pages, widgets, and enhanced user experiences.

## Prerequisites

- WHMCS template development knowledge
- Smarty templating experience
- Custom module development skills
- Client area customization access

## Workflow Steps

### Step 1: Set Up Custom Portal Structure

Organize custom portal code:

```bash
# Create portal structure
mkdir -p /var/www/whmcs/templates/six/custom_portal
mkdir -p /var/www/whmcs/templates/six/custom_portal/assets/css
mkdir -p /var/www/whmcs/templates/six/custom_portal/assets/js
mkdir -p /var/www/whmcs/templates/six/custom_portal/includes
mkdir -p /var/www/whmcs/modules/custom_portal/widgets
```

### Step 2: Create Custom Portal Template

Build enhanced client portal template:

```smarty
{* templates/six/custom_portal/clientareahome.tpl *}
{include file="$template/includes/header.tpl"}

<div class="custom-portal-container">
    {* Welcome Section *}
    <section class="welcome-section">
        <div class="welcome-header">
            <h1>Welcome back, {$client.firstname}!</h1>
            <p class="account-summary">Your account summary for {$client.companyname|default:$client.fullname}</p>
        </div>
        
        <div class="quick-stats">
            <div class="stat-card">
                <i class="fas fa-server"></i>
                <div class="stat-content">
                    <span class="stat-value">{$activeServices}</span>
                    <span class="stat-label">Active Services</span>
                </div>
            </div>
            <div class="stat-card">
                <i class="fas fa-globe"></i>
                <div class="stat-content">
                    <span class="stat-value">{$activeDomains}</span>
                    <span class="stat-label">Domains</span>
                </div>
            </div>
            <div class="stat-card">
                <i class="fas fa-file-invoice-dollar"></i>
                <div class="stat-content">
                    <span class="stat-value">{$unpaidInvoices}</span>
                    <span class="stat-label">Unpaid Invoices</span>
                </div>
            </div>
            <div class="stat-card">
                <i class="fas fa-ticket-alt"></i>
                <div class="stat-content">
                    <span class="stat-value">{$openTickets}</span>
                    <span class="stat-label">Open Tickets</span>
                </div>
            </div>
        </div>
    </section>
    
    {* Quick Actions *}
    <section class="quick-actions">
        <h2>Quick Actions</h2>
        <div class="action-grid">
            <a href="{$BASE_PATH}cart.php" class="action-card">
                <i class="fas fa-shopping-cart"></i>
                <span>Order New Services</span>
            </a>
            <a href="{$BASE_PATH}supporttickets.php" class="action-card">
                <i class="fas fa-headset"></i>
                <span>Get Support</span>
            </a>
            <a href="{$BASE_PATH}clientarea.php?action=invoices" class="action-card">
                <i class="fas fa-receipt"></i>
                <span>View Invoices</span>
            </a>
            <a href="{$BASE_PATH}clientarea.php?action=contacts" class="action-card">
                <i class="fas fa-users"></i>
                <span>Manage Contacts</span>
            </a>
        </div>
    </section>
    
    {* Recent Services *}
    <section class="services-section">
        <div class="section-header">
            <h2>Your Services</h2>
            <a href="{$BASE_PATH}clientarea.php?action=products" class="view-all">View All</a>
        </div>
        
        <div class="services-grid">
            {foreach $services as $service}
            <div class="service-card {if $service.domainstatus eq 'Active'}active{elseif $service.domainstatus eq 'Suspended'}suspended{else}pending{/if}">
                <div class="service-header">
                    <span class="service-name">{$service.productname}</span>
                    <span class="service-status status-{$service.domainstatus|lower}">{$service.domainstatus}</span>
                </div>
                <div class="service-details">
                    <p class="domain">{$service.domain}</p>
                    <p class="billing">
                        <span class="price">{$service.amount}</span>
                        <span class="cycle">/ {$service.billingcycle}</span>
                    </p>
                    <p class="due-date">Next Due: {$service.nextduedate}</p>
                </div>
                <div class="service-actions">
                    <a href="{$BASE_PATH}clientarea.php?action=productdetails&id={$service.id}" class="btn btn-sm btn-primary">
                        Manage
                    </a>
                </div>
            </div>
            {foreachelse}
            <div class="no-services">
                <p>No services found. <a href="{$BASE_PATH}cart.php">Order your first service</a></p>
            </div>
            {/foreach}
        </div>
    </section>
    
    {* Custom Widgets *}
    {foreach $customWidgets as $widget}
        <section class="widget-section">
            {$widget.content}
        </section>
    {/foreach}
    
    {* Recent Activity *}
    <section class="activity-section">
        <h2>Recent Activity</h2>
        <div class="activity-list">
            {foreach $recentActivity as $activity}
            <div class="activity-item">
                <div class="activity-icon">
                    <i class="fas {$activity.icon}"></i>
                </div>
                <div class="activity-content">
                    <p class="activity-description">{$activity.description}</p>
                    <span class="activity-date">{$activity.date}</span>
                </div>
            </div>
            {foreachelse}
            <p class="no-activity">No recent activity</p>
            {/foreach}
        </div>
    </section>
</div>

{include file="$template/includes/footer.tpl"}
```

### Step 3: Create Custom Widgets

Build reusable portal widgets:

```php
<?php
// modules/custom_portal/widgets/ServiceStatusWidget.php

namespace WHMCS\CustomPortal\Widgets;

class ServiceStatusWidget
{
    public function getOutput($params)
    {
        $clientId = $params['clientId'];
        
        $services = Capsule::table('tblhosting')
            ->join('tblproducts', 'tblhosting.packageid', '=', 'tblproducts.id')
            ->where('tblhosting.userid', $clientId)
            ->where('tblhosting.domainstatus', 'Active')
            ->select([
                'tblhosting.id',
                'tblhosting.domain',
                'tblproducts.name as product_name',
                'tblhosting.nextduedate',
                'tblhosting.dedicatedip',
                'tblhosting.suspended',
            ])
            ->orderBy('tblhosting.nextduedate', 'asc')
            ->limit(5)
            ->get();
        
        // Get server status for each service
        foreach ($services as $service) {
            $service->server_status = $this->getServerStatus($service);
        }
        
        return [
            'title' => 'Service Status',
            'content' => \WHMCS\Module\Theme::render(
                'widgets/service_status.tpl',
                ['services' => $services]
            ),
            'order' => 1,
        ];
    }
    
    private function getServerStatus($service)
    {
        // Check if service is suspended
        if ($service->suspended) {
            return ['status' => 'suspended', 'message' => 'Service Suspended'];
        }
        
        // Check server uptime
        $serverId = Capsule::table('tblhosting')
            ->where('id', $service->id)
            ->value('server');
        
        if (!$serverId) {
            return ['status' => 'unknown', 'message' => 'No Server'];
        }
        
        $uptime = Capsule::table('mod_server_metrics')
            ->where('server_id', $serverId)
            ->orderBy('recorded_at', 'desc')
            ->first();
        
        if (!$uptime) {
            return ['status' => 'unknown', 'message' => 'Unknown'];
        }
        
        return [
            'status' => ($uptime->load ?? 0) < 90 ? 'online' : 'warning',
            'message' => 'Load: ' . ($uptime->load ?? 'N/A') . '%',
        ];
    }
}
```

```smarty
{* templates/six/custom_portal/widgets/service_status.tpl *}
<div class="widget service-status-widget">
    <div class="widget-header">
        <h3><i class="fas fa-server"></i> Service Status</h3>
    </div>
    <div class="widget-body">
        <table class="service-status-table">
            <thead>
                <tr>
                    <th>Service</th>
                    <th>Status</th>
                    <th>Next Due</th>
                </tr>
            </thead>
            <tbody>
                {foreach $services as $service}
                <tr>
                    <td>
                        <strong>{$service.domain}</strong>
                        <small>{$service.product_name}</small>
                    </td>
                    <td>
                        <span class="status-badge status-{$service.server_status.status}">
                            {$service.server_status.message}
                        </span>
                    </td>
                    <td>{$service.nextduedate}</td>
                </tr>
                {foreachelse}
                <tr>
                    <td colspan="3">No active services</td>
                </tr>
                {/foreach}
            </tbody>
        </table>
    </div>
</div>
```

### Step 4: Create Portal Hooks

Enhance portal functionality with hooks:

```php
<?php
// modules/custom_portal/hooks/portal_hooks.php

// Modify client area data
add_hook('ClientAreaPageHomepage', 1, function($vars) {
    $clientId = $vars['client']->id;
    
    // Add custom statistics
    $customStats = [
        'total_spent' => getTotalSpent($clientId),
        'member_since' => getMemberSince($clientId),
        'referral_count' => getReferralCount($clientId),
    ];
    
    // Get recent activity
    $recentActivity = getRecentActivity($clientId, 5);
    
    // Get custom widgets
    $widgets = \WHMCS\CustomPortal\WidgetManager::getWidgets($clientId);
    
    return [
        'customStats' => $customStats,
        'recentActivity' => $recentActivity,
        'customWidgets' => $widgets,
    ];
});

// Add navigation items
add_hook('ClientAreaNavbars', 1, function($vars) {
    return [
        'primary' => [
            'custom_dashboard' => [
                'label' => 'Dashboard',
                'uri' => 'clientarea.php?custom=1',
                'order' => 1,
            ],
            'custom_reports' => [
                'label' => 'Reports',
                'uri' => 'clientarea.php?action=reports',
                'order' => 50,
            ],
        ],
    ];
});

// Modify service list
add_hook('ClientAreaPageProductsServices', 1, function($vars) {
    foreach ($vars['services'] as &$service) {
        // Add custom data to each service
        $service['usage_stats'] = getServiceUsageStats($service['id']);
        $service['uptime_percent'] = getServiceUptime($service['id']);
        $service['custom_links'] = getServiceCustomLinks($service);
    }
    
    return ['services' => $vars['services']];
});

function getTotalSpent($clientId)
{
    return Capsule::table('tblinvoiceitems')
        ->join('tblinvoices', 'tblinvoiceitems.invoiceid', '=', 'tblinvoices.id')
        ->where('tblinvoiceitems.relid', $clientId)
        ->where('tblinvoices.status', 'Paid')
        ->sum('tblinvoiceitems.amount');
}

function getMemberSince($clientId)
{
    return Capsule::table('tblclients')
        ->where('id', $clientId)
        ->value('datecreated');
}

function getRecentActivity($clientId, $limit = 5)
{
    $activities = Capsule::table('tblactivitylog')
        ->where('userid', $clientId)
        ->orderBy('id', 'desc')
        ->limit($limit)
        ->get();
    
    return $activities->map(function($activity) {
        return [
            'description' => $activity->description,
            'date' => $activity->date,
            'icon' => $this->getActivityIcon($activity->description),
        ];
    })->toArray();
}

function getServiceUsageStats($serviceId)
{
    // Get bandwidth, storage, etc.
    $stats = Capsule::table('mod_service_usage')
        ->where('service_id', $serviceId)
        ->orderBy('recorded_at', 'desc')
        ->first();
    
    return [
        'bandwidth_used' => $stats->bandwidth_used ?? 0,
        'bandwidth_limit' => $stats->bandwidth_limit ?? 0,
        'storage_used' => $stats->storage_used ?? 0,
        'storage_limit' => $stats->storage_limit ?? 0,
    ];
}
```

### Step 5: Create Custom Portal Pages

Add custom pages to the portal:

```php
<?php
// modules/custom_portal/routes.php

// Custom page routes
add_hook('ClientAreaPage', 1, function($vars) {
    $request = $_GET['custom'] ?? $_GET['action'] ?? '';
    
    switch ($request) {
        case 'dashboard':
            return include __DIR__ . '/pages/dashboard.php';
            
        case 'reports':
            return include __DIR__ . '/pages/reports.php';
            
        case 'usage':
            return include __DIR__ . '/pages/usage.php';
    }
});
```

```php
<?php
// modules/custom_portal/pages/reports.php

namespace WHMCS\CustomPortal\Pages;

class ReportsPage
{
    private $template;
    
    public function __construct()
    {
        $this->template = ROOTDIR . '/templates/six/custom_portal/';
    }
    
    public function render($client)
    {
        $reportType = $_GET['type'] ?? 'billing';
        
        switch ($reportType) {
            case 'billing':
                $data = $this->getBillingReport($client->id);
                break;
            case 'usage':
                $data = $this->getUsageReport($client->id);
                break;
            case 'services':
                $data = $this->getServicesReport($client->id);
                break;
            default:
                $data = [];
        }
        
        $smarty = new \Smarty();
        $smarty->assign('client', $client);
        $smarty->assign('report', $data);
        $smarty->assign('reportTypes', $this->getReportTypes());
        
        return $smarty->fetch($this->template . 'pages/reports.tpl');
    }
    
    private function getBillingReport($clientId)
    {
        $invoices = Capsule::table('tblinvoices')
            ->where('userid', $clientId)
            ->whereIn('status', ['Paid', 'Unpaid'])
            ->orderBy('date', 'desc')
            ->limit(24)
            ->get();
        
        $totalPaid = $invoices->where('status', 'Paid')->sum('total');
        $totalUnpaid = $invoices->where('status', 'Unpaid')->sum('total');
        
        return [
            'title' => 'Billing Report',
            'summary' => [
                'total_paid' => $totalPaid,
                'total_unpaid' => $totalUnpaid,
                'invoice_count' => $invoices->count(),
            ],
            'invoices' => $invoices,
            'chart_data' => $this->prepareBillingChart($invoices),
        ];
    }
    
    private function getUsageReport($clientId)
    {
        $services = Capsule::table('tblhosting')
            ->join('tblproducts', 'tblhosting.packageid', '=', 'tblproducts.id')
            ->where('tblhosting.userid', $clientId)
            ->where('tblhosting.domainstatus', 'Active')
            ->get();
        
        foreach ($services as &$service) {
            $service->usage = Capsule::table('mod_service_usage')
                ->where('service_id', $service->id)
                ->orderBy('recorded_at', 'desc')
                ->first();
        }
        
        return [
            'title' => 'Resource Usage Report',
            'services' => $services,
        ];
    }
    
    private function prepareBillingChart($invoices)
    {
        $monthlyData = [];
        
        foreach ($invoices as $invoice) {
            $month = date('Y-m', strtotime($invoice->date));
            if (!isset($monthlyData[$month])) {
                $monthlyData[$month] = 0;
            }
            if ($invoice->status === 'Paid') {
                $monthlyData[$month] += $invoice->total;
            }
        }
        
        return [
            'labels' => array_keys($monthlyData),
            'values' => array_values($monthlyData),
        ];
    }
}
```

```smarty
{* templates/six/custom_portal/pages/reports.tpl *}
{include file="$template/includes/header.tpl"}

<div class="reports-page">
    <h1>{$report.title}</h1>
    
    <div class="report-filters">
        <form method="get">
            <input type="hidden" name="action" value="reports">
            <select name="type" onchange="this.form.submit()">
                {foreach $reportTypes as $value => $label}
                <option value="{$value}" {if $_GET.type eq $value}selected{/if}>
                    {$label}
                </option>
                {/foreach}
            </select>
        </form>
    </div>
    
    {if $report.summary}
    <div class="report-summary">
        <div class="summary-card">
            <span class="summary-label">Total Paid</span>
            <span class="summary-value">{$report.summary.total_paid|currency}</span>
        </div>
        <div class="summary-card">
            <span class="summary-label">Total Unpaid</span>
            <span class="summary-value">{$report.summary.total_unpaid|currency}</span>
        </div>
        <div class="summary-card">
            <span class="summary-label">Total Invoices</span>
            <span class="summary-value">{$report.summary.invoice_count}</span>
        </div>
    </div>
    {/if}
    
    <div class="report-chart">
        <canvas id="reportChart" data-labels='{$report.chart_data.labels|json_encode}'
                data-values='{$report.chart_data.values|json_encode}'></canvas>
    </div>
    
    <div class="report-table">
        <table class="data-table">
            <thead>
                <tr>
                    <th>Date</th>
                    <th>Invoice #</th>
                    <th>Amount</th>
                    <th>Status</th>
                </tr>
            </thead>
            <tbody>
                {foreach $report.invoices as $invoice}
                <tr>
                    <td>{$invoice.date}</td>
                    <td><a href="{$BASE_PATH}viewinvoice.php?id={$invoice.id}">#{$invoice.id}</a></td>
                    <td>{$invoice.total|currency}</td>
                    <td><span class="status-{$invoice.status|lower}">{$invoice.status}</span></td>
                </tr>
                {/foreach}
            </tbody>
        </table>
    </div>
</div>

{include file="$template/includes/footer.tpl"}
```

### Step 6: Create Portal API

Expose portal data via API:

```php
<?php
// modules/custom_portal/api/PortalAPI.php

namespace WHMCS\CustomPortal\Api;

class PortalAPI
{
    public function getClientSummary($clientId)
    {
        $client = Capsule::table('tblclients')
            ->where('id', $clientId)
            ->first();
        
        if (!$client) {
            throw new \Exception('Client not found', 404);
        }
        
        return [
            'id' => $client->id,
            'name' => $client->firstname . ' ' . $client->lastname,
            'email' => $client->email,
            'stats' => [
                'active_services' => Capsule::table('tblhosting')
                    ->where('userid', $clientId)
                    ->where('domainstatus', 'Active')
                    ->count(),
                'active_domains' => Capsule::table('tbldomain')
                    ->where('userid', $clientId)
                    ->where('status', 'Active')
                    ->count(),
                'open_tickets' => Capsule::table('tbltickets')
                    ->where('userid', $clientId)
                    ->whereNotIn('status', ['Closed'])
                    ->count(),
                'unpaid_invoices' => Capsule::table('tblinvoices')
                    ->where('userid', $clientId)
                    ->where('status', 'Unpaid')
                    ->count(),
            ],
        ];
    }
    
    public function getServiceUsage($serviceId)
    {
        $service = Capsule::table('tblhosting')
            ->where('id', $serviceId)
            ->first();
        
        $usage = Capsule::table('mod_service_usage')
            ->where('service_id', $serviceId)
            ->orderBy('recorded_at', 'desc')
            ->first();
        
        return [
            'service_id' => $serviceId,
            'domain' => $service->domain,
            'usage' => $usage ?? [],
            'limits' => [
                'bandwidth' => $service->bandwidth_limit ?? 0,
                'disk' => $service->disk_limit ?? 0,
            ],
        ];
    }
}
```

## Verification Checklist

- [ ] Custom template structure created
- [ ] Portal pages custom-built
- [ ] Widgets implemented and working
- [ ] Hooks functioning correctly
- [ ] Mobile responsiveness tested
- [ ] API endpoints accessible
- [ ] User permissions respected
- [ ] Performance acceptable
- [ ] Security reviewed
- [ ] Cross-browser testing passed

## Related Skills and Documentation

- [WHMCS User Documentation](whmcs-user-documentation-workflow.md)
- [WHMCS API Development](whmcs-api-development-workflow.md)
- WHMCS Template Documentation: https://developers.whmcs.com/themes/
- Smarty Template Engine: https://www.smarty.net/docs/

## Notes

- Always follow WHMCS coding standards
- Test customizations in development first
- Keep customizations separate from core files
- Document any template modifications
- Consider child themes for easier upgrades
- Optimize images and assets
- Maintain accessibility standards
