# WHMCS Domain Expiry Widget Module

## Overview
Display expiring domains widget for renewal tracking.

## Module File: widget.php

```php
<?php
/**
 * WHMCS Domain Expiry Widget
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

class DomainExpiryWidget extends WHMCS\Module\Contracts\WidgetModuleInterface
{
    protected $title = 'Domain Expiry';
    protected $description = 'Track expiring domains';
    protected $priority = 65;
    protected $icon = 'fa-globe';

    public function getData(): array
    {
        return [
            'expiring_30' => $this->getExpiringDomains(30),
            'expiring_7' => $this->getExpiringDomains(7),
            'expiring_1' => $this->getExpiringDomains(1),
            'recent_renewals' => $this->getRecentRenewals(),
        ];
    }

    public function generateOutput(array $data): string
    {
        $totalExpiring = count($data['expiring_30']);
        return <<<HTML
<div class="widget-content-padded">
    <div class="row text-center">
        <div class="col-sm-4">
            <div class="metric-box">
                <span class="metric-value text-warning">{$data['expiring_30']}</span>
                <span class="metric-label">30 Days</span>
            </div>
        </div>
        <div class="col-sm-4">
            <div class="metric-box">
                <span class="metric-value text-danger">{$data['expiring_7']}</span>
                <span class="metric-label">7 Days</span>
            </div>
        </div>
        <div class="col-sm-4">
            <div class="metric-box">
                <span class="metric-value text-danger">{$data['expiring_1']}</span>
                <span class="metric-label">1 Day</span>
            </div>
        </div>
    </div>

    <hr>

    <div class="domain-list">
        <h5>Expiring Soon</h5>
        {$this->renderDomainList($data['expiring_30'])}
    </div>
</div>
HTML;
    }

    protected function getExpiringDomains(int $days): array
    {
        $targetDate = date('Y-m-d', strtotime("+{$days} days"));
        return Capsule::table('tbldomains')
            ->join('tblclients', 'tbldomains.userid', '=', 'tblclients.id')
            ->where('tbldomains.expirydate', '<=', $targetDate)
            ->where('tbldomains.expirydate', '>=', date('Y-m-d'))
            ->where('tbldomains.status', 'Active')
            ->orderBy('tbldomains.expirydate', 'asc')
            ->limit(5)
            ->get(['tbldomains.id', 'tbldomains.domain', 'tbldomains.expirydate', 
                Capsule::raw("CONCAT(tblclients.firstname, ' ', tblclients.lastname) as client_name")]);
    }

    protected function getRecentRenewals(): int
    {
        return Capsule::table('tbldomains')
            ->where('expirydate', '>=', date('Y-m-d', strtotime('-30 days')))
            ->where('registrationdate', '<', date('Y-m-d', strtotime('-30 days')))
            ->count();
    }

    protected function renderDomainList(array $domains): string
    {
        if (empty($domains)) {
            return '<p class="text-muted">No domains expiring</p>';
        }

        $html = '<ul class="list-unstyled">';
        foreach ($domains as $domain) {
            $daysLeft = ceil((strtotime($domain->expirydate) - time()) / 86400);
            $html .= '<li style="padding: 5px 0; border-bottom: 1px solid #eee;">
                <a href="domains.php?action=domaindetails&id=' . $domain->id . '">' . htmlspecialchars($domain->domain) . '</a>
                <span class="pull-right ' . ($daysLeft <= 7 ? 'text-danger' : 'text-warning') . '">' . $daysLeft . ' days</span>
            </li>';
        }
        $html .= '</ul>';
        return $html;
    }

    public function getId(): string { return 'domain_expiry_widget'; }
    public function getName(): string { return $this->title; }
}

function whmcs_domain_expiry_widget_activate(): array
{
    return ['status' => 'success', 'description' => 'Domain Expiry Widget activated'];
}

function whmcs_domain_expiry_widget_deactivate(): array
{
    return ['status' => 'success', 'description' => 'Domain Expiry Widget deactivated'];
}
