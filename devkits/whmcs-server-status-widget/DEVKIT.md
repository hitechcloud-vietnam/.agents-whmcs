# WHMCS Server Status Widget Module

## Overview
Server health monitoring widget displaying server status, uptime, load, and resource usage.

## Module File: widget.php

```php
<?php
/**
 * WHMCS Server Status Widget
 * 
 * @package    WHMCS\Module\Widgets
 * @copyright  Copyright (c) 2024 HiTech Cloud Ltd
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

class ServerStatusWidget extends WHMCS\Module\Contracts\WidgetModuleInterface
{
    protected $title = 'Server Status';
    protected $description = 'Monitor server health and resource usage';
    protected $priority = 85;
    protected $icon = 'fa-server';

    public function getData(): array
    {
        return [
            'servers' => $this->getServerStatuses(),
            'critical_alerts' => $this->getCriticalAlerts(),
            'total_services' => $this->getTotalServices(),
            'active_services' => $this->getActiveServices(),
            'suspended_services' => $this->getSuspendedServices(),
        ];
    }

    public function generateOutput(array $data): string
    {
        $onlineCount = count(array_filter($data['servers'], fn($s) => $s['status'] === 'online'));
        $totalServers = count($data['servers']);

        return <<<HTML
<div class="widget-content-padded">
    <div class="row text-center">
        <div class="col-sm-4">
            <div class="metric-box">
                <span class="metric-value text-success">{$onlineCount}/{$totalServers}</span>
                <span class="metric-label">Servers Online</span>
            </div>
        </div>
        <div class="col-sm-4">
            <div class="metric-box">
                <span class="metric-value">{$data['active_services']}</span>
                <span class="metric-label">Active Services</span>
            </div>
        </div>
        <div class="col-sm-4">
            <div class="metric-box">
                <span class="metric-value text-warning">{$data['suspended_services']}</span>
                <span class="metric-label">Suspended</span>
            </div>
        </div>
    </div>

    <hr>

    <div class="server-list">
        <h5>Server Status</h5>
        {$this->renderServerList($data['servers'])}
    </div>

    <hr>

    <div class="alerts">
        <h5>Critical Alerts</h5>
        {$this->renderAlerts($data['critical_alerts'])}
    </div>
</div>

<style>
.metric-box { padding: 10px; }
.metric-value { display: block; font-size: 22px; font-weight: bold; }
.metric-label { display: block; font-size: 11px; color: #777; text-transform: uppercase; }
.server-row { display: flex; align-items: center; padding: 8px 0; border-bottom: 1px solid #eee; }
.server-status-dot { width: 10px; height: 10px; border-radius: 50%; margin-right: 10px; }
.server-status-dot.online { background: #27ae60; }
.server-status-dot.offline { background: #e74c3c; }
.server-status-dot.warning { background: #f39c12; }
.server-name { flex: 1; font-weight: 500; }
.server-load { font-size: 12px; color: #777; }
.alert-item { padding: 8px; background: #fdf2f2; border-left: 3px solid #e74c3c; margin-bottom: 8px; }
.alert-time { font-size: 11px; color: #999; }
</style>
HTML;
    }

    protected function getServerStatuses(): array
    {
        $servers = Capsule::table('tblservers')
            ->where('disabled', 0)
            ->get(['id', 'name', 'ipaddress', 'status', 'last_update']);

        $statuses = [];
        foreach ($servers as $server) {
            $load = $this->getServerLoad($server->id);
            $statuses[] = [
                'id' => $server->id,
                'name' => $server->name,
                'ip' => $server->ipaddress,
                'status' => $this->determineStatus($load),
                'load' => $load,
                'last_update' => $server->last_update,
            ];
        }

        return $statuses;
    }

    protected function getServerLoad(int $serverId): array
    {
        $cached = Capsule::table('mod_server_monitoring')
            ->where('server_id', $serverId)
            ->orderBy('checked_at', 'desc')
            ->first();

        if ($cached) {
            return [
                'cpu' => $cached->cpu_usage ?? 0,
                'memory' => $cached->memory_usage ?? 0,
                'disk' => $cached->disk_usage ?? 0,
            ];
        }

        return ['cpu' => 0, 'memory' => 0, 'disk' => 0];
    }

    protected function determineStatus(array $load): string
    {
        if ($load['cpu'] > 90 || $load['memory'] > 90) {
            return 'offline';
        } elseif ($load['cpu'] > 70 || $load['memory'] > 70) {
            return 'warning';
        }
        return 'online';
    }

    protected function getCriticalAlerts(): array
    {
        return Capsule::table('mod_service_incidents')
            ->whereNull('resolved_at')
            ->where('incident_type', '!=', 'maintenance')
            ->orderBy('started_at', 'desc')
            ->limit(3)
            ->get();
    }

    protected function getTotalServices(): int
    {
        return Capsule::table('tblhosting')->count();
    }

    protected function getActiveServices(): int
    {
        return Capsule::table('tblhosting')->where('domainstatus', 'Active')->count();
    }

    protected function getSuspendedServices(): int
    {
        return Capsule::table('tblhosting')->where('domainstatus', 'Suspended')->count();
    }

    protected function renderServerList(array $servers): string
    {
        $html = '';
        foreach ($servers as $server) {
            $statusClass = $server['status'];
            $loadText = "CPU: {$server['load']['cpu']}% | RAM: {$server['load']['memory']}%";
            $html .= <<<HTML
<div class="server-row">
    <span class="server-status-dot {$statusClass}"></span>
    <span class="server-name">{$server['name']}</span>
    <span class="server-load">{$loadText}</span>
</div>
HTML;
        }
        return $html ?: '<p class="text-muted">No servers configured</p>';
    }

    protected function renderAlerts(array $alerts): string
    {
        if (empty($alerts)) {
            return '<p class="text-success"><i class="fa fa-check"></i> No critical alerts</p>';
        }

        $html = '';
        foreach ($alerts as $alert) {
            $time = date('M j, H:i', strtotime($alert->started_at));
            $html .= "<div class=\"alert-item\">
                <div>Service #{$alert->service_id} is down</div>
                <div class=\"alert-time\">{$time}</div>
            </div>";
        }
        return $html;
    }

    public function getId(): string
    {
        return 'server_status_widget';
    }

    public function getName(): string
    {
        return $this->title;
    }
}
```

## Activation & Deactivation

```php
<?php
function whmcs_server_status_widget_activate(): array
{
    return ['status' => 'success', 'description' => 'Server Status Widget activated'];
}

function whmcs_server_status_widget_deactivate(): array
{
    return ['status' => 'success', 'description' => 'Server Status Widget deactivated'];
}
```

## Requirements

- WHMCS 8.0.0+
- PHP 7.4+
